---
layout: post
title: Building PubSub Shovel - A Message Transfer Tool for Google Cloud PubSub
categories: [Development, Open Source, Google Cloud, PubSub, Serverless]
---

Building a flexible, serverless tool for transferring messages between Google Cloud PubSub subscriptions and topics, inspired by RabbitMQ's shovel plugin.

## Background

Working with Google Cloud PubSub in production environments, I frequently encountered scenarios where I needed to move messages from one subscription to another topic. Whether for data migration, reprocessing failed messages, or routing messages to different processing pipelines, there wasn't a straightforward tool that offered the flexibility I needed.

RabbitMQ has an excellent "shovel" plugin that handles message transfer between queues, exchanges, and even different RabbitMQ clusters. However, Google Cloud PubSub lacks an equivalent built-in mechanism for flexible message routing and transfer operations.

This gap led me to create [PubSub Shovel](https://github.com/torbendury/pubsub-shovel) - a serverless tool that provides the missing message transfer capabilities for Google Cloud PubSub environments.

## Problem Statement

The challenges I faced with existing solutions:

**Limited Transfer Options**: Google Cloud's native tools primarily focus on streaming and real-time processing, but don't provide simple batch transfer capabilities for existing messages in subscriptions.

**Operational Complexity**: Moving messages between topics often required writing custom scripts, deploying temporary compute resources, or using complex streaming pipelines for what should be simple operations.

**Lack of Flexibility**: Existing solutions didn't offer fine-grained control over how many messages to transfer, concurrent processing limits, or proper error handling for partial failures.

**No Reusable Tool**: Every time I needed to move messages, I found myself writing similar code patterns. There was no standardized, deployable tool that could handle various message transfer scenarios.

The goal was to create a flexible, serverless tool that could handle message transfers efficiently while being easy to deploy and operate.

## Design Principles

### Serverless-First Architecture

I designed PubSub Shovel as a Google Cloud Function to eliminate infrastructure management overhead. The tool needed to be:

- **Zero-maintenance**: Deploy once, use repeatedly without managing servers
- **Cost-effective**: Pay only when transferring messages, no idle compute costs
- **Auto-scaling**: Handle varying message volumes without configuration
- **Cloud-native**: Integrate seamlessly with Google Cloud IAM and monitoring

### Flexible Message Processing

The tool supports two primary usage patterns:

**Batch Transfer**: Transfer a specific number of messages for controlled, incremental processing:

```json
{
  "numMessages": 100,
  "sourceSubscription": "projects/my-project/subscriptions/source-sub",
  "targetTopic": "projects/my-project/topics/target-topic"
}
```

**Complete Transfer**: Process all available messages in a subscription:

```json
{
  "allMessages": true,
  "sourceSubscription": "projects/my-project/subscriptions/source-sub",
  "targetTopic": "projects/my-project/topics/target-topic"
}
```

### Asynchronous Processing

Rather than making clients wait for potentially long-running transfers, the function immediately returns an acknowledgment and processes messages asynchronously:

```go
// Return immediate response
response := ShovelResponse{
    Status:    "accepted",
    Message:   "Message shoveling started asynchronously",
    RequestID: requestID,
}

w.WriteHeader(http.StatusAccepted)
```

This design enables:

- **Better user experience**: Immediate feedback without blocking
- **Timeout handling**: Long transfers don't cause HTTP timeouts
- **Monitoring capability**: Request IDs enable tracking and logging

## Technical Implementation

### Concurrent Message Processing

The core processing logic uses Go's concurrency features to handle messages efficiently:

```go
sourceSub.ReceiveSettings.Synchronous = false
sourceSub.ReceiveSettings.NumGoroutines = 10
sourceSub.ReceiveSettings.MaxOutstandingMessages = 100
```

Key design decisions:

**Controlled Concurrency**: Limited to 10 concurrent goroutines to prevent overwhelming the target topic while maintaining good throughput.

**Message Acknowledgment**: Original messages are only acknowledged after successful republication to the target topic, ensuring no message loss on failures.

**Race Condition Prevention**: Careful mutex usage around message counting to prevent accepting more messages than requested.

### Error Handling and Reliability

The implementation includes comprehensive error handling:

```go
// Validate target topic exists before processing
exists, err := targetTopic.Exists(ctx)
if err != nil {
    return 0, fmt.Errorf("failed to check if target topic exists: %v", err)
}
if !exists {
    return 0, fmt.Errorf("target topic %s does not exist", req.TargetTopic)
}
```

**Pre-flight Validation**: Checks target topic existence before starting to fail fast on configuration errors.

**Atomic Operations**: Messages are only acknowledged from the source after successful republication to prevent partial failures.

**Timeout Protection**: Processing includes configurable timeouts (5-10 minutes) to prevent indefinite hanging.

### Configuration Flexibility

The tool extracts project information and resource names from fully qualified domain names (FQDNs), eliminating the need for separate configuration:

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

This design enables cross-project transfers and simplifies usage by requiring only the standard PubSub resource names.

## Deployment Options

### Google Cloud Functions (Primary)

The serverless deployment option provides the best operational experience:

```bash
gcloud functions deploy pubsub-shovel \
  --runtime go124 \
  --trigger-http \
  --entry-point Handler \
  --allow-unauthenticated
```

Benefits:

- Zero infrastructure management
- Automatic scaling based on demand
- Integrated with Cloud Logging and Monitoring
- Pay-per-use pricing model

### Local Development

For testing and development, the tool can run locally using the Functions Framework:

```bash
go run cmd/main.go
```

This enables:

- Local testing with real PubSub resources
- Development iteration without deployment cycles
- Integration testing in CI/CD pipelines

### Containerized Deployment

The included Dockerfile supports deployment to any container runtime:

```dockerfile
FROM golang:1.24-alpine AS builder
# Build process...

FROM alpine:latest
COPY --from=builder /app/main .
CMD ["./main"]
```

This option enables:

- Deployment to Cloud Run for HTTP-triggered scenarios
- Integration into existing container orchestration systems
- Custom networking and resource configurations

## Testing Strategy

### Unit Testing

Comprehensive test coverage for request validation and utility functions:

```go
func TestHandler_ValidationErrors(t *testing.T) {
    tests := []struct {
        name         string
        payload      ShovelRequest
        expectedCode int
    }{
        // Test cases for various scenarios...
    }
    // Test implementation...
}
```

**Input Validation**: Tests ensure proper error handling for invalid requests, missing parameters, and conflicting options.

**CORS Handling**: Validates proper CORS headers for web application integration.

**Method Restrictions**: Confirms only POST requests are accepted with appropriate error responses.

### Integration Testing

The tool is designed to work with real PubSub resources for integration testing:

```bash
curl -X POST http://localhost:8080 \
  -H "Content-Type: application/json" \
  -d '{"numMessages": 10, "sourceSubscription": "projects/test/subscriptions/test-sub", "targetTopic": "projects/test/topics/test-topic"}'
```

This approach enables testing against actual PubSub behavior rather than mocks.

## Real-World Use Cases

### Data Migration

Moving messages from legacy topics to new topic structures during system migrations:

```bash
# Transfer all messages from old topic pattern to new pattern
curl -X POST https://YOUR_FUNCTION_URL \
  -d '{"allMessages": true, "sourceSubscription": "projects/prod/subscriptions/legacy-processor", "targetTopic": "projects/prod/topics/new-processing-pipeline"}'
```

### Reprocessing Failed Messages

Transferring messages from dead letter queues back to main processing topics:

```bash
# Reprocess 50 failed messages at a time
curl -X POST https://YOUR_FUNCTION_URL \
  -d '{"numMessages": 50, "sourceSubscription": "projects/prod/subscriptions/dlq-subscription", "targetTopic": "projects/prod/topics/main-processor"}'
```

### Message Routing

Redistributing messages between different processing pipelines based on operational needs:

```bash
# Route messages to different processing environment
curl -X POST https://YOUR_FUNCTION_URL \
  -d '{"numMessages": 100, "sourceSubscription": "projects/prod/subscriptions/backlog", "targetTopic": "projects/staging/topics/batch-processor"}'
```

## Security and IAM

The tool leverages Google Cloud IAM for authentication and authorization, requiring appropriate permissions:

**Required Permissions**:

- `pubsub.subscriber` on source subscription
- `pubsub.publisher` on target topic
- `pubsub.viewer` for topic existence checks

**Security Features**:

- CORS support for secure web application integration
- Input validation on all parameters
- No sensitive data stored in function code
- Uses Google Cloud's built-in authentication mechanisms

## Performance Characteristics

**Throughput**: Processes up to 10 messages concurrently with 100 outstanding messages, providing good balance between speed and resource usage.

**Latency**: Immediate HTTP response (sub-second) with asynchronous background processing.

**Scalability**: Cloud Functions automatically scale based on incoming requests, handling multiple concurrent shovel operations.

**Cost Efficiency**: Pay-per-invocation model means zero cost when not actively transferring messages.

## Challenges and Learning

### Concurrency Control

Managing concurrent message processing while maintaining accurate counts proved complex. The solution required careful mutex usage and atomic operations:

```go
var acceptedCount int
var processedCount int
var mutex sync.Mutex

// Increment accepted count immediately to prevent race condition
mutex.Lock()
acceptedCount++
mutex.Unlock()
```

### PubSub API Intricacies

Learning the nuances of PubSub's receive settings and message acknowledgment patterns:

- **Synchronous vs Asynchronous**: Asynchronous processing provides better throughput
- **Outstanding Messages**: Limiting outstanding messages prevents memory issues
- **Acknowledgment Timing**: Messages must only be acked after successful republishing

### Functions Framework Integration

Integrating with Google Cloud Functions Framework while maintaining local development capability required understanding the framework's initialization patterns and HTTP handling.

## Future Enhancements

**Message Filtering**: Add support for filtering messages based on attributes or content before transfer.

**Transformation Support**: Enable message transformation during transfer (format conversion, attribute modification).

**Batch Operations**: Support transferring to multiple target topics in a single operation.

**Monitoring Integration**: Enhanced metrics and alerting integration for production usage.

**Dead Letter Handling**: Built-in dead letter queue support for messages that fail to transfer.

## Conclusion

Building PubSub Shovel addressed a real operational need while exploring serverless architecture patterns and PubSub API capabilities. The project demonstrates how a focused, well-designed tool can solve specific problems more effectively than complex, general-purpose solutions.

Key takeaways from the project:

**Serverless Benefits**: Cloud Functions provided an excellent deployment model for this type of utility, eliminating operational overhead while providing excellent scalability.

**API Design Matters**: Simple, well-validated APIs enable reliable automation and easy integration into operational workflows.

**Concurrent Programming**: Proper concurrency control is essential when dealing with message ordering and count accuracy in distributed systems.

**Testing is Critical**: Comprehensive unit tests combined with integration testing against real PubSub resources provided confidence in reliability.

The tool is now available as an open-source project at [github.com/torbendury/pubsub-shovel](https://github.com/torbendury/pubsub-shovel), ready for deployment in any Google Cloud environment needing flexible PubSub message transfer capabilities.

Whether you're migrating data, reprocessing messages, or routing traffic between topics, PubSub Shovel provides a reliable, serverless solution that handles the operational complexity while offering the flexibility needed for real-world message transfer scenarios.
