---
layout: post
title: Golang Project Structure - Where to Start and Where to Go
categories: [Development, Golang]
---

How you can start structuring your Golang projects, depending on your journey.

## Intro

If you're starting your dive into the world of Go, you've certainly made a wise choice. Go (or Golang) is a statically typed, compiled language known for its simplicity, performance and efficiency.
Whether you're a seasoned developer or just starting your coding journey, it's essential to establish a solid project structure for each of your projects.
A well-organized project structure does not only enhance code maintainability but also sets the stage for collaboration and scalability (and future refactoring).

## I'm Getting Started

When you're just getting started to learn Go, you can read this passage and come back later. Go allows you to maintain a very very simple project structure by basically having a `main.go` file in which your code resides. Since your only restriction is to provide a `main()` function, a simple `main.go` file fits your start perfectly.

Your project structure at the very beginning will, after initializing your Go module with `go mod init my-golang-app`, look like this:

```bash
my-golang-app/
    ├── main.go
    ├── go.mod
    ├── go.sum
```

With those files, you're perfectly set up to start your Go journey.

## I'm Creating my First Real Applications

That's great! Even if you're starting to dive deeper into the world of Golang, the language itself does not restrict you in any way of finding your own project structure.

You can get an example on [torbendury/books-go](https://github.com/torbendury/books-go) where I created my own version of a project structure when I wrote a dummy REST API.

However, if you're already a bit more deeper inside the world of Go, you might recall one or the other directory in there. This is because I used my own very simplified version of the [golang-standards/project-layout](https://github.com/golang-standards/project-layout) which I will talk about more in the next section. This section is intended to encourage you to find your own way rather than blindly following a standard layout.

## I'm Experienced or Want to Contribute to Open Source

Now that you've seen basic project structures in place, let's explore some advanced considerations to take your Go projects to the next level.

### Directory Structure

For larger projects, consider organizing your code into separate directories based on functionality. Here's a more complex project structure, derived from [golang-standards/project-layout](https://github.com/golang-standards/project-layout):

```bash
my-golang-app/
    ├── cmd/
    │   ├── myapp/
    │   │   ├── main.go
    ├── pkg/
    │   ├── mymodule/
    │   │   ├── mymodule.go
    ├── internal/
    │   ├── secret/
    │   │   ├── secret.go
    ├── api/
    │   ├── api.go
    ├── web/
    │   ├── server.go
    ├── go.mod
    ├── go.sum
```

As you can see, the structure has grown a bit, using the following standards:

- `cmd` contains executable entry points for your application. Each subdirectory can represent a different component of the application.
- `pkg` houses reusable packages and modules. Those can also be reused by other developers that fork your project or use it as a dependency.
- `internal` contains packages and modules which should NOT be imported by external packages, they are intended for internal use only. However, it is not technically enforced.
- `api` includes API definitions and related code, i.e. models.
- `web` contains web server code if your projects serves a web application.

This structure, being a bit more mature, keeps your code very well organized, making it easier to maintain, refactor and extend your project as it grows.

### Testing

Writing tests is crucial to ensure you code's realiability and maintainability. Create a `<foo>_test.go` file in the same directory as your code files to write tests. You can use the built-in `testing` package for writing unit tests.

### Documentation

Document your code using [GoDoc-style](https://go.dev/blog/godoc) comments. Go has a built-in tool called `godoc` that generated documentation for your code.

## `golang-standards/project-layout`

Although I mentioned the [golang-standards/project-layout](https://github.com/golang-standards/project-layout) multiple times in this project, I wouldn't necessarily recommend it to everyone. To me, it is more of a "if you ask yourself whether you need this layout, you don't" standard. Even the Go dev team states that this is not an official standard, so you are very free to find your own one. However, it is worth understanding this layout when you're working with external dependencies, independent developers or open source projects since this is a very good set of common historical and emerging project layout patterns which you can find all over the whole Golang ecosystem.

As I said, if you're starting out, a `main.go`, `go.mod` and `go.sum` file is all you need. If you're working on your own (and even more important: For your own!), you're also very free to find your own layout (see [`torbendury/books-go`](https://github.com/torbendury/books-go)). But if you need to work with other developers or want your project to be used by others, you might consider adhering to this unofficial standard.

If you're unsure about naming, formatting and styling, Go is there for you. Use `gofmt` and `golint` to get some help - in IDEs by JetBrains and VSCode they ship with batteries included.

## More Ideas

I personally derived my layout for `books-go` from a [YouTube video by Anthony GG](https://www.youtube.com/watch?v=EqniGcAijDI) which you might be interested in, as he also shares some thoughts about why he structures his code in a certain way.

There are also [many](https://www.youtube.com/watch?v=PTE4VJIdHPg), [many](https://www.youtube.com/watch?v=MzTcsI6tn-0), [many](https://www.youtube.com/watch?v=oL6JBUk6tj0), [many](https://www.youtube.com/watch?v=ltqV6pDKZD8) GopherCon talks about project layouts, anti-patterns and best practices which you might be interested in. They're all free to watch and hold so much information for you!

Now that you're armed with some information, go and see if you can build up your very own fitting project layouts!
