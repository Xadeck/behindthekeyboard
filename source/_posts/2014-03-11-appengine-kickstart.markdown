---
layout: post
title:  "Kickstart for Google Appengine"
categories: appengine
comments: true
---

In this post, I want to describe a very simple setup for getting started with AppEngine and being able to prototype an application very quickly, with _minimum_ boilerplate code and additional frameworks to understand.

# Pre-requisites

This assumes you're a programmer, know how to use the shell in Linux/OsX, and have your favorite text editor configured for you.

You must have minimum knowledge of Python 2.7. If you don't, you can for example start with [Dive Into Python](http://www.diveintopython.net/).  You must have downloaded and installed [AppEngine SDK for Python](https://developers.google.com/appengine/downloads). From the command line, you should have a command `dev_appserver.py` available. You must also have a Google account and an AppEngine account. If you don't, go to [AppEngine](https://appengine.google.com/) and do what it takes to get to _My applications_ page that looks like this.

{% img /assets/appengine_myapplications.png My Applications page %}

Finally, it's recommended to setup versioning. Easiest is to setup a git repository:
{% terminal %}
$ cd ~
$ mkdir appengine_tutorial && cd appengine_tutorial
$ git init .
{% endterminal %}
And later on push it to [GitHub](http://github.com). Also configure your `.gitignore` to ignore `*.pyc` files.

# Sanity check

First thing to run an AppEngine application is to _configure_ it by filling an `app.yml` file. Copy the following content:

``` yaml app.yaml 
application: appengine-tutorial
version: 1
runtime: python27
threadsafe: true
api_version: 1

handlers:
- url: /.*
  script: main.APP
```

In the first block, the application is the name of the AppEngine application you created in _My Applications_  dashboard earlier.  The rest is boilerplate you can ignore for now. The second block indicates that all incoming requests (the `url:` regular expression will be matched against the path of the URL, so `/.*` will match anything) will be _dispatched_ to the object `APP` in the file `main.py`. This object must be callable with two parameters, so the simplest version of the file is:

``` python main.py
def APP(env, resp): 
  return "OK"
```

Now run the dev server locally:

{% terminal %}
$ dev_appserver.py .
{% endterminal %}

The output should show something like:

{% terminal %}
INFO     2014-03-03 22:11:44,827 sdk_update_checker.py:241] Checking for updates to the SDK.
INFO     2014-03-03 22:11:45,080 sdk_update_checker.py:269] The SDK is up to date.
WARNING  2014-03-03 22:11:45,086 api_server.py:341] Could not initialize images API; you are likely missing the Python "PIL" module.
INFO     2014-03-03 22:11:45,107 api_server.py:138] Starting API server at: http://localhost:55832
INFO     2014-03-03 22:11:45,112 dispatcher.py:176] Starting module "default" running at: http://localhost:8080
INFO     2014-03-03 22:11:45,132 admin_server.py:117] Starting admin server at: http://localhost:8000
{% endterminal %}

As indicated on the last line, open your web browser at [http://localhost:8080](http://localhost:8080). You should see a simple OK page as below:

{% img /assets/localhost_ok.png OK page %}

Looking back at the terminal from which you launched the server, you should also see the following log statements:

{% terminal %}
INFO     2014-03-03 22:15:27,050 module.py:612] default: "GET / HTTP/1.1" 500 12
INFO     2014-03-03 22:15:27,107 module.py:612] default: "GET /favicon.ico HTTP/1.1" 500 12
{% endterminal %}

From now on, everytime a change will be made to either `app.yaml` or `main.py`, the server will have to be shutdown from the terminal with ^C, and restarted with `dev_appserver.py .`.

# Components of the kickstart

An AppEngine application comes naked. You have to program it from scratch, that is treating the incoming HTTP requests, and building the HTML content to return. In a final application, you will never do that and will instead use one of the (many) existing frameworks. Those layers usually add up both to the number of things to install and to the learning curve. Moreover, it hides some of the underlying things that you - in my opinion - should know.

The goal of the kickstart is to provide you with a scaffolding:

-  that provides the basic functionalities (routing, templates, styling)
-  yet quite "close to the metal" so you know what's happening
-  and with minimum boilerplate code and added dependency

For these reasons, it uses:

-  [Bottle][bottle] for the routing and request manipulation
-  [Jinja2][jinja2] for template rendering
-  [Boostrap][boostrap] for styling

Jinja2 is part of AppEngine. Just add the following lines to the bottom of the `app.yaml` file:

``` yaml app.yaml
libraries:
- name: jinja2
  version: latest
```

Bottle comes as a single file that you just have to put in your main directory. And boostrap can also be bundled as a single file. Go to the respective websites to have instructions, or just execute the following (update the version numbers if necessary):

{% terminal %}
$ wget https://raw.github.com/defnull/bottle/0.12.4/bottle.py
$ mkdir -p static/css
$ wget -P static/css \
  https://raw.github.com/twbs/bootstrap/v3.1.1/dist/css/bootstrap.min.css
$ mkdir -p static/js
$ wget -P static/js \
  https://raw.github.com/twbs/bootstrap/v3.1.1/dist/js/bootstrap.min.js
{% endterminal %}

So your directory structure should now be like this:
{% terminal %}
{% raw %}
$ tree .
.
|-- app.yaml
|-- bottle.py
|-- main.py
`-- static
    |-- css
    |   `-- bootstrap.min.css
    `-- js
        `-- bootstrap.min.js
{% endraw %}
{% endterminal %}

# Routing with Bottle

The first step is to be able to decide which content to serve for different URLs on our site. For example, the root path '/' would display a home page, and `/about.html` would display information about the site and its author. Let's use [Bottle][bottle] for that, changing `main.py` for:

``` python main.py
import bottle

APP = bottle.Bottle()

@APP.route('/')
def home():
  return "<h1>Home page</h1>"

@APP.route('/about')
def about():
  return "<h1>About this site</h1>"
```

Restarting the server and navigating to [http://localhost:8080](http://localhost:8080) and [http://localhost:8080/about](http://localhost:8080/about) should show the simple HTML contact above. But navigating to some other URL like [http://localhost:8080/contact](http://localhost:8080/contact) would render a default 404 error page. 

{% img /assets/default_404.png Default 404 %}

The log would also report the 404:
{% terminal %}
INFO     2014-03-05 20:26:16,855 module.py:612] default: "GET /contact HTTP/1.1" 404 734
{% endterminal %}

Using [Bottle](bottle) to direct all 404 errors to render a custom page instead is trivial:

``` python main.py
@APP.error(404)
def error404(error):
    return 'Nothing here, sorry'
```

Of course, the text rendered so far is limited. More annoying, it's not well formed HTML. Constructing the HTML manually using string manipulation in Python is a no go. Templates are instead the way.

# Templates for pages

Templating frameworks offer many features, but there are two basic useful ones. First they allow writing HTML files with extra syntax to indicate placeholders that can be rendered from the code data injected in the place holders. Second, they allow those HTML files to _inherit_ one another, making it easy to share boiler plate and layout. 

[Jinja2][jinja2] is such a framework and is very simple to use. Add the following lines to `main.py`

``` python main.py
import jinja2

ENV = jinja2.Environment(loader=jinja2.FileSystemLoader('templates'))
```

And then to produce the content for a route, for example `/`, simply use:

``` python main.py
@APP.route('/')
def home():
  return ENV.get_template('home.html').render()
```

The result of these two changes is that the about page will now be rendered from the file `templates/index.html`. Similarly, there would be afiles for the other pages. These files would have different content but would share a lot of similarities. In order to avoid duplicating content, these files will _inherit_ from a base one named `templates/_base.html` whose content will be:

``` html templates/_base.html
{% raw %}<!DOCTYPE HTML>
<html>
	<head>
	  <meta charset="utf-8">
	</head>
	<body>
	  {% block body %}
	  {% endblock body %}
	</body>
</html>{% endraw %}
```

This would then be referenced as follow from the `index.html` file:

``` html templates/index.html
{% raw %}{% extends "_base.html" %}
{% block body %}
<h1>Home page</h1>
{% endblock body %}{% endraw %}
```

The `about.html` and `error404.html` would do the same, and the result is that each page only specifiy its content, and the structure is shared. Of course, [Jinja2][jinja2] offers many more perks, but this is how to get started. At this point, the structure of the directory is:

{% terminal %}
{% raw %}
$ tree .
.
|-- app.yaml
|-- bottle.py
|-- main.py
|-- static
|   |-- css
|   |   `-- bootstrap.min.css
|   `-- js
|       `-- bootstrap.min.js
`-- templates
    |-- _base.html
    |-- about.html
    |-- error404.html
    `-- index.html
</pre>
{% endraw %}
{% endterminal %}

Note that the code above used the convention to prefix layout template with `_` and to 
name the HTML files as the function using them, but that is just a convention.

# Serving static content

At this stage, page can be easily added and rendered. Going to the logs of the locally running server, you would notice lines like this (if you don't go to [http://localhost:8080](http://localhost:8080) and do a forced reload for example with `Cmd+Shift+R`):

{% terminal %}
INFO     2014-03-05 21:32:49,636 module.py:612] default: "GET /favicon.ico HTTP/1.1" 500 62
{% endterminal %}

To add a [favicon](http://en.wikipedia.org/wiki/Favicon) to the site, simply get the icon - for example using [http://www.favicon.cc/](http://www.favicon.cc/) - and put it in the `static/` folder. Then, add the following lines to the _beginning_  of the handlers section of `app.yml` file:
``` yaml app.yaml
- url: /favicon.ico
  static_files: static/favicon.ico
  upload: static/favicon.ico
```
This has to be added to the beginning because handlers are processed in order, so the pattern '/.*' must be last.

Note that reading the documentation of [Bottle][bottle], you would find that it also has a mechanism for serving static content. However, using AppEngine ones is more efficient as the server can do extra optimization to serve that content (hence the extra `upload:` field, read the documentation for more information). 

Note also that routing could be done by adding handlers to the `app.yml` file, or using Bottle. This time, it's better to do it in Bottle, as it's easier. 

# A zest of Bootstrap

The last step of this tutorial is to use all the pieces introduced to be able to use Bootstrap in our pages, in order to have good looking prototype pages. First, handlers are added to `app.yml` so that the stylesheets and javascripts can be served:
``` yaml app.yaml
- url: /css
  static_dir: static/css  
- url: /js
  static_dir: static/js  
```

Then, the `_base.html` template is edited to be:
``` html templates/_base.html
{% raw %}<!DOCTYPE HTML>
<html>
	<head>
	  <meta charset="utf-8">
	  <meta name="viewport" content="width=device-width, initial-scale=1.0">
	  <link rel="stylesheet" href="css/bootstrap.min.css">
	</head>
	<body>
	  {% block body %}
	  {% endblock body %}
	  <script src="//ajax.googleapis.com/ajax/libs/jquery/1/jquery.min.js"></script>
	  <script src="js/bootstrap.min.js"></script>
	</body>
</html>{% endraw %}
```

And now bootstrap can be used, for example like this in the `body` section of `index.html`:
``` html templates/index.html
<div id="wrap">
  <div class="container">
    <div class="page-header">
      <h1>Page header</h1>
    </div>
    <p class="lead">Lead paragraph.</p>
  </div>
</div>
```

This concludes the construction of the AppEngine kickstart. The last step is to make this live to the world by pushing it to appengine, which is a single command line using `appcfg.py`.
{% terminal %}
$ appcfg.py update .
{% endterminal %}

This will ask the email and password for the account used to create the AppEngine applications at the beginning. And a few seconds later, the application will be serving. An example is running at [http://btk-gae-kickstart.appspot.com/](http://btk-gae-kickstart.appspot.com/)!

# From there...

The goal of this kickstart is to get you started with AppEngine. The real work starts now, with learning from the [AppEngine developer docs](https://developers.google.com/appengine/) how to store data, how to use models, APIs, etc. But you can now do so with a boilerplate application that's very lightweight. This boilerplate is easily accessible from [its GitHub page](https://github.com/Xadeck/appengine_kickstart) so when you want to start an new AppEngine project, you can simply:

{% terminal %}
$ git clone https://github.com/Xadeck/appengine_kickstart.git
{% endterminal %}

Of course, once you've prototyped your idea, or when you will be designing more and more complex applications, you will probably want to invest in higher level frameworks that can be ran ontop of AppEngine such as Django. But before you get there, you can use this as a playground to become fluent.

[bottle]: http://bottlepy.org/docs/dev/index.html
[jinja2]: http://jinja.pocoo.org/docs/
[boostrap]: http://getbootstrap.com/