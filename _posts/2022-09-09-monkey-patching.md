---
layout: post
title: Monkey Patching 101
categories: [Programming, Development, Python]
---

Monkey patching allows you to modify your code functions at runtime. Have a look into it to grasp the advantages and use cases.

## Introduction

Monkey patching (probably derived from *guerrilla patching* - a practice of changing code in a sneaky way - and then by homophony becoming *gorilla*) is a way for developers to modify (and extend!) software **while it is running**. You can find it e.g. when programming Python.

## Practical Examples

So, we can modify code in runtime. Why should we? Let's go through some samples.

### Stubs

A stub is a piece of code used as a placeholder. Commonly, you use it when another method or API is not yet existing or finished. While this is the case, you can use stubs to mock the functionality of the method or API you want to use later. Stubs can be very useful in distributed systems or in general testing scenarios.

An example would be the (very simple) processing of a humidity value:

```python
def process_humidity(hum: int):
  if hum > 50:
    print(f"Humidity is over {hum} percent.")

def humidity_stub(sensor: Sensor):
  return 35

process_humidity(humidity_stub(my_sensor))
```

In this example, the `humidity_stub` is used to mock the retrieval of the humidity measured by a hardware IoT device. This is a very popular use case of monkey patching for when you are already developing data processing software while no real data is available.

### Patching Third Party Software

Another use case is patching/extending/modifying third party software. Monkey patching of third party software is mainly done when you do not want to keep (and maintain!) a local copy of the third party software.

A simple example would be some simple value adjustment, i.e. when processing images and prerendering a set of preset sizes which the third party software does not normally offer:

```python
image_presets = [
  [128, 128],
  [256, 256],
  [512, 512],
  [1028, 1028]
]

input_picture = "/tmp/input.png"

import ImageProcessing

ip = ImageProcessing()

# IMAGE_PRESETS is an internal constant of the module `ImageProcessing` and normally
# may not be accessed.
ip.IMAGE_PRESETS = image_presets

ip.render_presets()
```

Note that this patching can often only be done with sufficient knowledge of internal processes that rely on the modified module, while extensions can often be done without the internal processes noticing.

### Live Security Patches (or Workarounds)

This is one of my favorite ways of using monkey patching. Countless times, you will face the fact that a piece of software has an unpatched security issue. More often than not, you will face security issues that are coming from features you are probably not even using, like [Stored XSS via labels color](https://about.gitlab.com/releases/2022/08/30/critical-security-release-gitlab-15-3-2-released/#stored-xss-via-labels-color).

Software that is backed by teams, organizations, or companies is often patched quite fast, but there is software that is no longer being maintained or is backed by one or two aspiring developers.

At least in the second case, it might be handy that you can patch certain pieces of software from the outside. The simplest example might be the following:

```python
# funky module
def funky_function():
    buffer = [None]*10
    for i in range(0,41):
        buffer[i]=7 
```

This is a very basic example of a buffer overflow. Our buffer consists of 10 elements while our for-loop is iterating over 42 elements, which results in an error.

Since we made the decision that we can live without the features provided by the `funky_function` - or a function that relies on it can live without it, we can simply patch it when using it:

```python
from funky_module import funky_function

def funky_function():
    # temporary fix until patch is released
    buffer = []
    for i in range(0,41):
        buffer.append(7)
    # rest of code stays the same
```

Fun fact: It is still a common practice to deliver security patches by dynamic class modification, so-called *hot fixes*.

### Self Healing

Although it is not the most common use case, it might be one of the most interesting to think about. I recently read multiple papers and degree projects about JavaScript errors that were caused by browser extensions (script and ad blockers). They discussed options for how to repair websites that were broken by such extensions.

A presented solution was to develop extensions that fix broken JavaScript code caused by blocked third-party scripts or blocked advertising blocks, i.e. by replacing such blocks with dummy fills or even partly re-activating features on websites that were included in very large libraries or frameworks.

Unfortunately, I can not provide a simple example for this topic at the moment. But a really interesting project regarding self-healing code is [Healenium](https://healenium.io/), a self-healing framework for Selenium tests.

## The Bad and the Ugly

Of course, monkey patches can - just like every other piece of code that is not originally intended to be there - be harmful.

Some of the most common problems are undocumented and/or poorly written code. This can lead to (but is not limited to) problems like:

- (Un)intended malicious code that causes even worse problems (or ones that would not exist without the patch)
- Concurring patches by developers which are unaware of each other and tamper each others code
- Unstable software that often breaks when being upgraded
  - Hint: This can be avoided by applying monkey patches optionally and with easy to plug out feature flags

Fun fact: One of the most popular mutual (one may call it malicious) monkey patch battles was fought between Giorgio Maone (developer of NoScript) and the developers of Adblock Plus. Maone utilized his browser extension to add exceptions to the ad-blocker plugin. This allowed him to allow-list his own advertisement blocks.

## Conclusion

Monkey patches can be a very good method in your toolbox when you are testing API integrations in distributed systems while your counterpart is not yet ready to receive requests or when you are designing extensions to your programs and want to test processing functions.

However, you will have to be most careful when monkey patching third party software or when you are trying to mitigate security issues.

Wherever you might use monkey patches in the future, always make sure to properly document a) the patch itself and b) the reasons why it is being applied.
