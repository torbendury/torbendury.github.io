---
layout: post
title: Jenkins Shared Libraries 101
categories: [DevOps, Development, Jenkins]
---

A quick glance on how Jenkins shared libraries work, how to write your own libraries and some deeper thoughts of structuring and extending your library.

## What Are Shared Libraries?

The name itself says a lot about the topic. Shared libraries are **functions** or complete **CI/CD pipelines** that are made available (*more or less*) globally in your Jenkins environment. The functions or pipelines can be used within calling pipelines like standard Groovy functions and thus significantly reduce the effort of CI/CD.

However, shared libraries not only help to **reduce effort**, they also create general **simplicity** and thus significantly improve the **maintainability** of the in-house CI/CD. Since such shared libraries can also be hosted internally in a `git` repository, they also benefit from an easily traceable history and versioning, for example via **tags**. A shared library can thus be treated like a separate software project and benefits from all the tools that git provides.

So while you are developing new features in your pipeline, developer A can still use version v4.2.0, while developer B likes to be cutting-edge and use the `main` branch of the library.

## When To Use Shared Libraries

Shared libraries should be used wherever you have to do **similar or the same tasks** in a CI/CD pipeline in several software projects. One can think of 10 backend applications built on NodeJS. They are all built with the same in-house toolstack and can all access the shared library. This allows us to let developers develop so that they can put their brains into real work and rely on a stable pipeline.

If we move away from building software, we can also use shared libraries in the **Infrastructure-as-Code** area. We can imagine a library that allows us to deploy our Terraform code in Azure, AWS or Google Cloud Platform, while we only specify the code and provide credentials via Jenkins.

We can also provide **general tasks** (that are not related to the actual software development) as a function in a shared library. With a commit to the `main` branch, a ticket can be created via a library function, which is then assigned to the QS team for review. Fully automated without a developer having to worry about the process and ticketing API.

Certain **standards can also be enforced**. When we deploy infrastructure-as-code, we want security scans to be run across the entire configuration beforehand. If these fail, the code is not deployed.

### Opionating Libraries

**Shared libraries tend to be opinionated.**

While we can *of course* force standards or bring about our own standards via the shared libraries, it is not always the best way.

If you make shared libraries available in your company, it must follow certain company standards. That doesn't mean it is opinionated. It only meets certain minimum requirements for a quality standard that is required in the company.

Opinionated is a library that goes *beyond* these requirements. Speaking at a higher level, a library should only be **a framework** to be used. This is the case, for example, if you provide individual functions that take over certain CI/CD software development tasks.

Of course, as a developer, I still want to be able to influence how packages are built, archived, signed and published. It is up to the *author* of the library to make this possible or to nip it in the bud.

Another thing is to provide **complete pipelines** for convenience. Here you can tighten the belt of possibilities very tightly, after all it is a company-compliant pipeline that has to work with any software projects and also has to guarantee **a certain stability**.

## How Much Do You Want To Share?

As with a normal software project, anyone providing a shared library must identify stakeholders and get them on board. It is not enough to develop a library according to your *own taste* and switch to maintenance mode. A shared library is always aimed at a user group. This user group can have a lot of knowledge about CI/CD or none at all.

No single DevOps should be solely responsible for such a library. It is an organic product that **grows and changes with changing requirements**. It is therefore important to also involve the software developers, i.e. the user group, closely in the development. Otherwise you develop the library past the need and have sunk time for dead code.

### Splitting Up Libraries

You will most certainly reach a point where you've written a shared library that provides atomic functions and some shared pipelines around it which calls those functions. This will also be the point where you will have written at least *some* functions that can be useful not only in the context of the library you have written, but can also be used in other libraries or **user-defined pipelines**.

Take for example a function that sends a webhook to a Microsoft Teams channel or to a similar notification endpoint, or creates a calendar entry to notify co-workers about an upcoming maintenance window. Such functions (and other helpers) can not only be useful in your library, but can also provide great improvement to the workflow of other CI/CD pipelines in your company or organization.

This is the point where you should start a shared library that consists of those **utility functions**. You can then import this utility library into your shared library and use those utilities there while you **prevent duplicated code and human errors**.

## Versioning

This part is easy. I have come to the conclusion that a shared library is nothing else than a software product that is shipped to a customer, so I use [semantic versioning](https://semver.org/). Major, breaking changes, as well as minor compatible changes and small fixes can be shipped safely.

The best thing about it is that users can use your library on a specific version that you have pushed with a git tag, like so:

```groovy
// use the latest version or the default version bound via Jenkins
@Library('shared-library') _

// use a specific version that has been pushed to a git tag
@Library('my-shared-library@1.2.3') _

// use the latest version from a specified branch
@Library('my-shared-library@master') _
```

Tags and branches work that easy because a git tag is basically nothing else than a branch that does not change anymore but is more like a pointer to a specific git commit (**Note**: At least if you use lightweight tags - but this is another story).

## Writing Your First Shared Library

Let us take two simple examples. First, we will create a shared library that will contain a function `sayHello(String name)` which can be called independently as long as the caller has imported the library. Second, we will create a shared pipeline that can be called like so: `helloPipeline(Map config)` in which the user will enter some configuration for the pipeline.

### Directory structure

For our example, your directory structure will look something like this:

```text
(root)
+- vars
|   +- hello.groovy
|   +- hello.txt             # help text for 'hello'
```

For more complex examples or for a full directory structure, I will provide you some links at the end of the post.

Next, type the following into your `hello.groovy` file:

```groovy
def sayHello(String name) {
    echo "Hello, " + name
}
```

Your `hello.txt` file will contain documentation about your library for users. This help will be present at the *Global Variables Reference* page in Jenkins.

```markdown
# hello Library

This library contains useful functionalities for greeting users when they run pipelines.

## sayHello

Arguments:

- `string` name

This function prints a friendly greeting to a specified name on the Jenkins job console using the `echo` method.
```

Now, create a file `helloPipeline.groovy` inside the same directory as the `hello.groovy` and `hello.txt` file:

```groovy
@Library('shared-library') _

def call(Map config){
    pipeline {
        agent none
        environment {
            USER_TO_GREET = config['name']
            JENKINS_LABEL = config['jenkinsLabel']
        }
        stages {
            stage('Greet') {
                agent { label JENKINS_LABEL }
                steps {
                    script {
                        hello.sayHello(USER_TO_GREET)
                    }
                }
            }
        }
    }
}
```

Congratulations! You have created your first shared library function as well as your first shared pipeline. Call it like this:

```groovy
#!/usr/bin/groovy

def config = [
    name: "Torben",
    jenkinsLabel: "myjenkins"
]

helloPipeline(config)
```

As soon as you run the job, the defined `helloPipeline` will be called and execute the stage `Greet` that will print `Hello, Torben` to the job console.

## Summary And Helpful Links

As you see, it is **definitely not hard** getting started with writing your own Jenkins libraries. They can make it *very* easy to define standardized pipeline steps and functions that can be used by you and your colleagues without any hassle. Make sure to **document** your functions and pipelines well so it will actually be used and won't end as dead code.

If you want to read on some more or add some bookmarks, here are some links which I found helpful when taking my first steps with shared libraries in Jenkins. **Soon I'll probably publish my own Terraform Jenkins library and write a devlog to it, so stay tuned.**

- [Jenkins - Extending with Shared Libraries](https://www.jenkins.io/doc/book/pipeline/shared-libraries/)
- [Best practices for writing Jenkins shared libraries](https://bmuschko.com/blog/jenkins-shared-libraries/)
- [Aimtheory - Global shared library best practices](https://www.aimtheory.com/jenkins/pipeline/continuous-delivery/2017/12/02/jenkins-pipeline-global-shared-library-best-practices.html)
