---
layout: post
title: PubSub Shovel - Moving Messages Around Like It's 2025
categories: [Development, Open Source, Google Cloud, PubSub, Serverless]
---

I got tired of writing the same message transfer scripts over and over, so I built a proper tool for it.

## The Problem

You know that feeling when you're working with PubSub and suddenly you need to move a bunch of messages from one subscription to another topic? Maybe you're migrating data, or reprocessing some failed messages, or just need to route stuff around.

Google's tooling is great for streaming and real-time processing, but when you just want to grab some existing messages and dump them somewhere else? Good luck with that. You end up writing custom scripts every single time.

RabbitMQ folks had this figured out ages ago with their shovel plugin. It just works - you point it at a source and destination, and it moves messages. Simple.

So I built [PubSub Shovel](https://github.com/torbendury/pubsub-shovel) to fill that gap.

## What It Does

Simple concept: HTTP endpoint that moves messages from a PubSub subscription to a topic. You can either move a specific number of messages or just say "move everything".

Here's what a request looks like:

```json
{
  "numMessages": 100,
  "sourceSubscription": "projects/my-project/subscriptions/source-sub",
  "targetTopic": "projects/my-project/topics/target-topic"
}
```

Or if you want to move everything:

```json
{
  "allMessages": true,
  "sourceSubscription": "projects/my-project/subscriptions/source-sub",
  "targetTopic": "projects/my-project/topics/target-topic"
}
```

The function returns immediately and processes stuff in the background. No more waiting around for transfers to complete:

```go
response := ShovelResponse{
    Status:    "accepted",
    Message:   "Message shoveling started asynchronously",
    RequestID: requestID,
}
```

You get a request ID back so you can track it in the logs if needed.

## Implementation Details

### Concurrency Without Chaos

The tricky part was handling concurrent message processing without making a mess:

```go
sourceSub.ReceiveSettings.Synchronous = false
sourceSub.ReceiveSettings.NumGoroutines = 10
sourceSub.ReceiveSettings.MaxOutstandingMessages = 100
```

I limited it to 10 concurrent goroutines because more than that tends to overwhelm the target topic. Plus 100 outstanding messages keeps memory usage reasonable.

The important bit: messages only get acknowledged from the source *after* they're successfully republished to the target. No message loss if something goes wrong.

Had to be careful with race conditions around message counting. Nothing worse than accepting more messages than you asked for because of timing issues.

### Error Handling

Basic stuff but important:

```go
// Check if target topic actually exists before we start
exists, err := targetTopic.Exists(ctx)
if err != nil {
    return 0, fmt.Errorf("failed to check if target topic exists: %v", err)
}
if !exists {
    return 0, fmt.Errorf("target topic %s does not exist", req.TargetTopic)
}
```

Fail fast if the target doesn't exist. Also has timeouts (5-10 minutes) so it won't hang forever if something's broken (or if you request more messages than are available).

### Configuration

One nice thing - you just use the full PubSub resource names and it figures out projects automatically:

```go
func extractProjectID(resourceName string) string {
    parts := splitResourceName(resourceName)
    for i, part := range parts {
        if part == "projects" && i+1 < len(parts) {
            return parts[i+1]
        }
    }
    return ""
}
```

This means you can do cross-project transfers without extra config.

## Deployment

### Cloud Functions

The obvious choice for this kind of utility:

```bash
gcloud functions deploy pubsub-shovel \
  --runtime go124 \
  --trigger-http \
  --entry-point Handler \
  --allow-unauthenticated
```

Zero infrastructure to manage, scales automatically, and you only pay when you use it.

### Running Locally

For development, just:

```bash
go run cmd/main.go
```

Works with real PubSub resources, so you can test properly without deploying every time.

### Docker

There's a Dockerfile if you want to run it somewhere else:The included Dockerfile supports deployment to any container runtime:

```dockerfile
FROM golang:1.24-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/main .
CMD ["./main"]
```

Works with Cloud Run if you want HTTP triggers with more control.

Although I did not test it myself, you should also be able to build a standalone binary and deploy it to any environment that can run the resulting executable.

## Testing

I wrote tests for the important stuff - input validation, CORS handling, utility functions:

```go
func TestHandler_ValidationErrors(t *testing.T) {
    tests := []struct {
        name         string
        payload      ShovelRequest
        expectedCode int
    }{
        // Test cases...
    }
}
```

The integration tests use real PubSub resources instead of mocks. More reliable that way.

## Use Cases

### Data Migration

```bash
# Move everything from old to new topic structure
curl -X POST https://YOUR_FUNCTION_URL \
  -d '{"allMessages": true, "sourceSubscription": "projects/prod/subscriptions/legacy-processor", "targetTopic": "projects/prod/topics/new-pipeline"}'
```

### Dead Letter Queue Recovery

```bash
# Reprocess failed messages, 50 at a time so you don't flood the system
curl -X POST https://YOUR_FUNCTION_URL \
  -d '{"numMessages": 50, "sourceSubscription": "projects/prod/subscriptions/dlq-sub", "targetTopic": "projects/prod/topics/main-processor"}'
```

### Environment Routing

```bash
# Move backlog to staging for testing
curl -X POST https://YOUR_FUNCTION_URL \
  -d '{"numMessages": 100, "sourceSubscription": "projects/prod/subscriptions/backlog", "targetTopic": "projects/staging/topics/batch-processor"}'
```

## Permissions

You need the usual PubSub permissions:

- `pubsub.subscriber` on the source subscription
- `pubsub.publisher` on the target topic
- `pubsub.viewer` to check if topics exist

CORS is enabled so you can call it from web apps if needed.

## What I Learned

### Concurrency is Hard

Getting the message counting right with multiple goroutines was trickier than expected. Had to use mutexes carefully to avoid race conditions:

```go
var acceptedCount int
var processedCount int
var mutex sync.Mutex

// Lock immediately to prevent accepting more than requested
mutex.Lock()
acceptedCount++
mutex.Unlock()
```

### PubSub Quirks

Had to learn some PubSub API specifics the hard way:

- Asynchronous processing is way faster than synchronous
- Outstanding message limits prevent memory bloat
- Only ack messages after successful republishing, otherwise you lose them

### Functions Framework

The Go Functions Framework took a bit of figuring out, especially making it work for both local development and Cloud Functions deployment. Worth it though. `functions-framework` makes it easy to switch between environments and keeps local development and GCP Cloud Function / Run Functions at parity without me having to jump through fire hoops. :-)

## What's Next

Could add message filtering based on attributes, or transformation during transfer. Maybe support for multiple target topics in one operation.

But honestly, it does what I need it to do right now.

## Conclusion

Built this because I was tired of writing the same message moving code over and over. Now it's a proper tool that just works.

Cloud Functions turned out to be perfect for this - zero maintenance, scales automatically, and costs nothing when you're not using it.

The code is on [GitHub](https://github.com/torbendury/pubsub-shovel) if you want to use it or improve it. Deploy once and you're set for all your message moving needs.
