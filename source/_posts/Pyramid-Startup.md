---
title: Pyramid Startup
date: 2015-08-27 18:12:40
categories: Python
tags: [Python, Pyramid]
---

Here's a high-level time-ordered overview of what happens when you press return after running pserve development.ini.

### Pserve command
The pserve command is invoked under your shell with the argument development.ini. As a result, Pyramid recognizes that it is meant to begin to run and serve an application using the information contained within the development.ini file.

### Main section
The framework finds a section named either [app:main], [pipeline:main], or [composite:main] in the .ini file. This section represents the configuration of a WSGI application that will be served. If you're using a simple application (e.g. [app:main]), the application's paste.app_factory entry point will be named on the use= line within the section’s configuration.

If, instead of a simple application, you're using a WSGI pipeline (e.g. a [pipeline:main] section), the application named on the “last” element will refer to your Pyramid application. If instead of a simple application or a pipeline, you're using a “composite” (e.g. [composite:main]), refer to the documentation for that particular composite to understand how to make it refer to your Pyramid application. In most cases, a Pyramid application built from a scaffold will have a single [app:main] section in it, and this will be the application served.

### Logging
The framework finds all logging related configuration in the .ini file and uses it to configure the Python standard library logging system for this application. See Logging Configuration for more information.

### Application constructor
The application's constructor named by the entry point reference on the use= line of the section representing your Pyramid application is passed the key/value parameters mentioned within the section in which it’s defined. The constructor is meant to return a router instance, which is a WSGI application.<br>
For Pyramid applications, the constructor will be a function named main in the __init__.py file within the package in which your application lives. If this function succeeds, it will return a Pyramid router instance. Here’s the contents of an example __init__.py module:
``` python
from pyramid.config import Configurator

def main(global_config, **settings):
    """ This function returns a Pyramid WSGI application.
    """
    config = Configurator(settings=settings)
    config.include('pyramid_chameleon')
    config.add_static_view('static', 'static', cache_max_age=3600)
    config.add_route('home', '/')
    config.scan()
    return config.make_wsgi_app()
```
Note that the constructor function accepts a global_config argument, which is a dictionary of key/value pairs mentioned in the [DEFAULT] section of an .ini file (if [DEFAULT] is present).
It also accepts a **settings argument, which collects another set of arbitrary key/value pairs. The arbitrary key/value pairs received by this function in **settings will be composed of all the key/value pairs that are present in the [app:main] section (except for the use= setting) when this function is called by when you run pserve.

Our generated development.ini file looks like so:
``` ini
###
# app configuration
# http://docs.pylonsproject.org/projects/pyramid/en/latest/narr/environment.html
###

[app:main]
use = egg:MyProject

pyramid.reload_templates = true
pyramid.debug_authorization = false
pyramid.debug_notfound = false
pyramid.debug_routematch = false
pyramid.default_locale_name = en
pyramid.includes =
pyramid_debugtoolbar

# By default, the toolbar only appears for clients from IP addresses
# '127.0.0.1' and '::1'.
# debugtoolbar.hosts = 127.0.0.1 ::1

###
# wsgi server configuration
###

[server:main]
use = egg:waitress#main
host = 0.0.0.0
port = 6543

###
# logging configuration
# http://docs.pylonsproject.org/projects/pyramid/en/latest/narr/logging.html
###

[loggers]
keys = root, myproject

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = INFO
handlers = console

[logger_myproject]
level = DEBUG
handlers =
qualname = myproject

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(asctime)s %(levelname)-5.5s [%(name)s][%(threadName)s] %(message)s
```

In this case, the myproject.__init__:main function referred to by the entry point URI egg:MyProject (see development.ini for more information about entry point URIs, and how they relate to callables), will receive the key/value pairs {'pyramid.reload_templates':'true', 'pyramid.debug_authorization':'false', 'pyramid.debug_notfound':'false', 'pyramid.debug_routematch':'false', 'pyramid.debug_templates':'true', 'pyramid.default_locale_name':'en'}. See Environment Variables and .ini File Settings for the meanings of these keys.

### Settings dictinary
The main function first constructs a Configurator instance, passing the settings dictionary captured via the **settings kwarg as its settings argument.

The settings dictionary contains all the options in the [app:main] section of our .ini file except the use option (which is internal to PasteDeploy) such as pyramid.reload_templates,
pyramid.debug_authorization, etc.

### Application registry
The main function then calls various methods on the instance of the class Configurator created in the previous step. The intent of calling these methods is to populate an application registry, which represents the Pyramid configuration related to the application.

### Call make_wsgi_app
The make_wsgi_app() method is called. The result is a router instance. The router is associated with the application registry implied by the configurator previously populated by other methods run against the Configurator. The router is a WSGI application.

### Emmit ApplicationCreated event
An ApplicationCreated event is emitted (see Using Events for more information about events).

### Router
Assuming there were no errors, the main function in myproject returns the router instance created by pyramid.config.Configurator.make_wsgi_app() back to pserve. As far as pserve is concerned, it is “just another WSGI application”.

### WSGI server
pserve starts the WSGI server defined within the [server:main] section. In our case, this is the Waitress server (use = egg:waitress#main), and it will listen on all interfaces (host = 0.0.0.0), on port number 6543 (port = 6543). The server code itself is what prints serving on 0.0.0.0:6543 view at http://127.0.0.1:6543. The server serves the application, and the application is running, waiting to receive requests.

**About development.ini**

The development.ini file is a PasteDeploy configuration file. Its purpose is to specify an application to run when you invoke pserve, as well as the deployment settings provided to that application.

This file contains several sections including [app:main], [server:main] and several other sections related to logging configuration.

The [app:main] section represents configuration for your Pyramid application. The use setting is the only setting required to be present in the [app:main] section. Its default value, egg:MyProject, indicates that our MyProject project contains the application that should be served. Other settings added to this section are passed as keyword arguments to the function named main in our package's __init__.py module. You can provide startup-time configuration parameters to your application by adding more settings to this section.

The line in [app:main] above that says use = egg:MyProject is actually shorthand for a longer spelling: use = egg:MyProject#main. The #main part is omitted for brevity, as #main is a default defined by PasteDeploy. egg:MyProject#main is a string which has meaning to PasteDeploy. It points at a setuptools entry point named main defined in the MyProject project.

See Entry Points and PasteDeploy .ini Files for more information about the meaning of the use = egg:MyProject value in this section.

Take a look at the generated setup.py file for this project.
``` python
import os

from setuptools import setup, find_packages

here = os.path.abspath(os.path.dirname(__file__))
with open(os.path.join(here, 'README.txt')) as f:
    README = f.read()
with open(os.path.join(here, 'CHANGES.txt')) as f:
    CHANGES = f.read()

requires = [
    'pyramid',
    'pyramid_chameleon',
    'pyramid_debugtoolbar',
    'waitress',
]

setup(name='MyProject',
    version='0.0',
    description='MyProject',
    long_description=README + '\n\n' + CHANGES,
    classifiers=[
        "Programming Language :: Python",
        "Framework :: Pyramid",
        "Topic :: Internet :: WWW/HTTP",
        "Topic :: Internet :: WWW/HTTP :: WSGI :: Application",
    ],
    author='',
    author_email='',
    url='',
    keywords='web pyramid pylons',
    packages=find_packages(),
    include_package_data=True,
    zip_safe=False,
    install_requires=requires,
    tests_require=requires,
    test_suite="myproject",
    entry_points="""\
    [paste.app_factory]
    main = myproject:main
    """,
)
```

Note that entry_points is assigned a string which looks a lot like an .ini file. This string representation of an .ini file has a section named [paste.app_factory]. Within this section, there is a
key named main (the entry point name) which has a value myproject:main. The key main is what our egg:MyProject#main value of the use section in our config file is pointing at, although it isactually shortened to egg:MyProject there. The value represents a dotted Python name path, which refers to a callable in our myproject package's __init__.py module.

The egg: prefix in egg:MyProject indicates that this is an entry point URI specifier, where the “scheme” is “egg”. An “egg” is created when you run setup.py install or setup.py develop within your project.

In English, this entry point can thus be referred to as a “PasteDeploy application factory in the MyProject project which has the entry point named main where the entry point refers to a main function in the mypackage module”. Indeed, if you open up the __init__.py module generated within any scaffold-generated package, you'll see a main function. This is the function called by PasteDeploy when the pserve command is invoked against our application. It accepts a global configuration object and returns an instance of our application.