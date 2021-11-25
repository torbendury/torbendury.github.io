---
layout: post
title: Introduction to Rapid Web Development Using Python Flask
categories: [Programming, Web]
---

This post is meant to be a light introduction into Python Flask. At the end of this post, you'll be able to rapidly develop small web applications. With some more learning, those apps will be enterprise-ready. The best of it? They're natively built to run in containerized runtimes (like Docker, containerd, ...) and the most available serverless platforms support the usage of Flask.

## What is Flask?

Basically, Flask is a Python framework meant for web development. It provides you with libraries and technologies which allow you to build stable web applications in a very fast manner. This includes simple web pages, dynamic blogs or complex commercial websites. Flask itself has little dependencies to external libraries and comes with "batteries included".

## Pros and Cons of Flask

Flask is light. This is both a pro and a contra. Positively, you don't need to worry about dependencies (and keeping them up-to-date) and watch for security bugs etc. - on the other side, this means that you will either need to work more by yourself to get the features you want, or search for and install libraries by yourself. The main dependencies of Flask are:

- `Werkzeug`, which is a WGSI (Web Server Gateway Interface) library
  - this helps Python communicating with servers
- `jinja2`, which is a templating engine
  - this enables you to write e.g. HTML templates which can be filled dynamically

### Outlook: Templating

If you've ever worked with HTML or built your own website, you probably ran into the problem of keeping multiples pages in your website consistent. Templating makes this easy. Basically, you define your page layout once (think of a blog: There's a layout for your home page, all blog posts look the same, ...) and can use them infinite times.

You can not just define them once, but you can change them whenever you like and your whole website is still going to match!


## Your first web application

Let's stop the theoretic part and get our hands dirty. First, get up-to-date and install everything you need. It's just two commands, so start your terminal (in this case, bash) and type:

```bash
  # update local repository and look for software upgrades
  $ sudo apt update && sudo apt upgrade -y
  # check if you have Python installed
  $ python3 --version
  # If it's not installed, go ahead with this:
  $ sudo apt install -y python3.8 python3-pip
  # Install Flask
  $ pip3 install flask
```

When you're finished, start up by creating your first project directories:

```bash
  $ mkdir -p flask_projects/hello_world/{static,templates}
  $ cd flask_projects/hello_world/
```

Now open this directory in your favorite IDE (or if you like staying in terminal, use `vim`) and create a file called `main.py` which has the following content inside:

```python
#!/usr/bin/env python3
# up there is a shebang, so you can execute the file and it will automatically run the users installed python3.x

# Import the flask module itself
import flask

# Instanciate the Flask app
app = flask.Flask(__name__)

@app.route('/')
def index():
    """ Displays the index at / """
    return flask.render_template('index.html')

if __name__ == '__main__':
    app.debug=True
    app.run()
```

Now in the sub-directory `templates/`, create a sample `index.html` file and paste the following:

```html
<!DOCTYPE html>
<html lang='de'>
  <head>
    <meta charset="utf-8"/>
    <title>Hello world!</title>
    <link type="text/css" rel="stylesheet" href="{{ url_for('static', filename='example.css')}}"/>
  </head>
  <body>

Hello from <a href="https://torbentechblog.com">TorbenTechBlog</a>!

  </body>
</html>
```

Hit save, and from your terminal (or IDE) run the application:

```bash
  $ chmod +x main.py
  $ ./main.py
```

This is going to start your local webserver. Try to access it under [http://localhost:5000](http://localhost:5000) (the port may differ, it will be printed out on your log output on which port exactly the app is running).

Congratulations, you just built your first web application with Flask and served some content!

## Now what about templating? You promised templating!

Sure thing. In this section, I'll show you a quick introduction to templating. It's easy, free and fun! 

When you opened up the browser last time, you saw "Hello from TorbenTechBlog". Quite boring (not the blog, I heard it's awesome), so let's be more dynamic.

First, update the code to serve not only on `/` but also on `/<name>`. How do we do that? We simply add another `route`:

```python
@app.route('/<name>/')
def hello(name):
    """ Displays a nice hello to whoever's there."""
    return flask.render_template('hello.html', name=name)
```

Create the following template `hello.html` to be served from your app:

```html
<!DOCTYPE html>
  <html lang='de'>
  <head>
    <meta charset="utf-8"/>
    <title>Hello {{name}}</title>
    <link type="text/css" rel="stylesheet" href="{{ url_for('static', filename='hello.css')}}"/>
  </head>
  <body>
      Hello {{name}}
  </body>
</html>
```

You know the drill, hit save and start your app.

When accessing [http://localhost:5000/](http://localhost:5000), you will still see the first page we've built. But when you access [http://localhost:5000/Torben](http://localhost:5000/Torben), you will see "Hello Torben"!
