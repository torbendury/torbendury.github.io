---
layout: post
title: Writing clean code in Python
categories: [Programming, Development, Python]
---

There are many articles and books out there which describe several code standards, software engineering principles and some guidelines to write clean code. Still, there's an insane amount of spaghetti flying around in repositories. This (hopefully not yet-another-article) will give you some hints to get started when you're looking for simple ways to commit better code.

## The best code is the one you didn't write

There's many ways to understand this principle. Let's dig into this.

First, think before you code. When you come up with an idea on solving a problem, do not start solving it in your IDE. Think about it first. Use some awesome free tools like [DrawIO](https://diagrams.net) to record what your solution will look like. Save this diagram, I promise it is going to change soon.

Then, write pseudocode. Pseudocode doesn't have to follow any particular syntax, it can be done in your native language. Try to cover all possible paths. Write full sentences and take notes if you feel something might go wrong at some point.

```python
# A function to divide a number by two. It leaves error handling to the caller.
  # function divide_by_two(number: integer)
      # return number / 2
```

It is up to you how much you let your pseudocode already look like real syntactic code. Once you've given some sample input to your pseudocode and validated that it's producing the right output, you're ready to start writing real code.

Another way of understanding the title is that you should actually not write any code yourself at all. There are thousands of computer scientists who, with a probability bordering on certainty, have solved either the same problem or at least a similar one. Open source gives you the opportunity to let your project stand on the shoulders of giants. If someone has already bothered to solve a specific problem or solve many problems with a general solution, credit the code. Be a good computer scientist, be lazy, recycle the code.

## Standards, standards, standards (PEP8)

Standards are important. They basically are a collection of rules, guidelines and best practices. Some conventions require that you know them in order to apply and conform to them. However, many standards can be rolled out fully automatically to your code.

A great example from the Python world is the [PEP8 standard](https://peps.python.org/pep-0008/). It sprang from the same problem you have. Too many programmers had different views on how code should look like, how it should be structured, how it should be documented and what naming to use.

There are Python packages that (recursively) check your code for PEP8 compliance and give you tips on how to restructure your code. The most important tool is probably the `autopep8` package that can be installed via `pip`.
Better still, most IDEs have plugins/extensions that allow to automatically check for PEP8 compliance on every save and even customize the code directly wherever possible.

When solving minor problems, it is preferable to always refer to Python's [standard library](https://docs.python.org/3/library/). It is regularly maintained and developed by real gurus who live Python. It should always be preferred when you have to choose between the standard library and an external module. The decision path should be: **standard library -> external module -> custom development**

## General Coding Principles

There is an almost unmanageable amount of coding principles. You can follow them to write better code, but you should always be aware of the pros and cons of each principle. Not every principle is applicable to every problem. Your company may also have established some company-wide principles to follow. Always think about the big picture of your solution architecture before applying code principles.

Some of the more popular and general principles are as follows:

### DRY (Do not repeat yourself)

This is one of the simpler principles that can be applied to any code. It consists of just one rule: don't repeat yourself. Write your code exactly once, write it as well as you can, and reuse it.

Since we also want to look at the negative sides, here's a con: If you try to make code too general, it will eventually become too abstract and incomprehensible. You will eventually try to solve a problem that is too general, rendering your code unusable.

### KISS (keep it simple, stupid)

The KISS principle can also be used almost anywhere. It is preferable to write simple code than complex one. Even if you lose a nanosecond of performance, it helps you, your peers, and your successors to understand what you were trying to do.

## Format all the Things

Code formatters force a certain coding style. Most of these are configurable and can therefore be fully tailored to your needs. This configuration can be shared with all your colleagues as it is also stored "as code". The most popular formatters, which I use myself, are `autopep8` and `black`. They both try not to break your code (black even promises) while formatting and restyled, and you'll find that every time you save your code, it's much more readable.

### Also lint all the things

Most IDEs come with linters pre-installed - if not, you should definitely install them afterwards.
They prevent you from making small mistakes, including potentially dangerous code, or breaking the formatting. In short: They are the watchdog of your code while you are programming. The most popular Python linters are "Pylint", "PyFlakes" and "mypy". [Here](https://testdriven.io/blog/python-code-quality/) is a very good article about Python code quality, if you want a deep dive in linting and linter configuration.

## Think before you commit

You probably check your code into an SVN like a Git repository. If you don't want your co-workers to start hating you, you should write code they love.

To prevent yourself from checking in malicious code, there are several ways.

On the one hand, you should definitely integrate a good workflow into your repository. Prevent yourself from being able to push to the `master` branch. Not even for "tiny changes". You can protect branches in most Git servers for this.

Second, after all that linting and formatting, you should double-check that you're conforming to the standards for your repository. It is worth taking a look at the ["pre-commit"](https://pre-commit.com/) tool for this.

pre-commit allows you to attach a configured hook to your repository that will be executed each time you want to check in a commit. A predefined number of conformances will be checked and you will be notified if you violate one. If you break any of the rules, you can't submit your commit until you've fixed everything.

## Conclusion

It's not easy to write clean code. There is no universal rule as to what you have to do. But there are some philosophies and conventions you can stick to. There are many tools that do the manual work for you so you can focus more on your code itself.

The most important thing to remember is to stay consistent. Write simple code that is easy to understand and easy to debug. Write it so that you can understand it and make it so that everyone else can understand it (there is no code that is self-documenting).

Clean code is learning-by-doing and learning-by-refactoring.
