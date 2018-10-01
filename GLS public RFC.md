# General language support RFC

Passenger only supports a few programming languages right now. "Generic language support" — **"GLS"** — is what we call the effort to make Passenger support *all* programming languages. This document:

 * Explains why we think GLS will benefit the world.
 * Proposes how GLS would work, and discusses possible issues.
 * Proposes a user experience to expose GLS features.

This document is meant for Passenger users. We would like to get your feedback on these questions:

 * How could GLS benefit you? How should we change Passenger to address your use cases for today and for tomorrow?
 * Are the proposed mechanisms are viable?
 * Is the proposed UX desirable?

Let us know by posting a comment!

<!-- TOC depthFrom:2 depthTo:4 -->

- [1. Why GLS?](#1-why-gls)
- [2. Operating principle](#2-operating-principle)
    - [2.1. Concurrency concerns](#21-concurrency-concerns)
- [3. User experience](#3-user-experience)
    - [3.1. User configured start command](#31-user-configured-start-command)
    - [3.2. Developer-provided start command: Passengerfile.json](#32-developer-provided-start-command-passengerfilejson)
    - [3.3. Concurrency configuration](#33-concurrency-configuration)
- [4. Tell us about your use cases](#4-tell-us-about-your-use-cases)

<!-- /TOC -->

## 1. Why GLS?

Application delivery and DevOps are too hard. They shouldn't be. Passenger's goal is to provide an experience that people will love, and to allow teams to move faster.

Passenger is an application server. You can see it as an "application service library" that exists outside the app. It takes care of concerns such as connection handling, concurrency, troubleshooting, admin tools and much more.

We started out with Ruby and slowly moved to cover Python and Node.js as well. But with microservices and containers, the world is becoming increasingly polyglot. We see that every language reinvents tooling and duplicates efforts. Not only is this a waste, it is also so that the quality and usability of tooling vary wildly.

Today Passenger helps more than 650.000 websites world-wide, such as Apple, Pixar and Intercom. With GLS, we want to bring all the good stuff from Passenger to the whole world:

 * **Standardization** — Onboard team members faster. Streamline procedures. Reduce inconsistencies. Share and retain operational knowledge.
 * **Focus** — Write less code. Reduce infrastructure complexity. Automate the boring details. Focus on your core business.

These benefits are achieved through features like web server integration, process management, inter-process load balancing, request/memory/queue limiting, admin tools, etc. And course, through great UX and documentation.

## 2. Operating principle

Passenger's basic architecture **does not care which languages apps are written in**. Passenger runs apps in an out-of-process manner: apps do not run inside Passenger but outside of it. Passenger just needs to know how to spawn an app process, and how to communicate with that process via HTTP.

Many apps and frameworks designed in the past rely on an external server to provide HTTP connectivity. Examples include: Ruby (Rack), Python (WSGI), PHP (mod_php/FastCGI) and Java (J2EE/Tomcat). But newer apps and frameworks make use of builtin, lightweight HTTP servers to directly provide HTTP connectivity. Think Node.js, Go and Java (Jetty). This approach has become the norm for nearly all modern apps. GLS makes use of this.

The idea is that the user supplies a command string that tells Passenger how to start the app on a specific port `$PORT`. Passenger then simply looks for a free port, then executes the command string, substituting `$PORT` with the actual port, then waits until the port becomes in use.

This approach is similar to how the Heroku Procfile system works.

### 2.1. Concurrency concerns

Different apps, languages and frameworks have wildly different concurrency properties. For example Node.js makes use of evented concurrency, Go makes use of an M:N threading model with many connections per thread, while most Ruby and Python frameworks only support a one-connection-per-thread model. Luckily, Passenger's core server is evented and thus is able to handle huge amounts of concurrency. However, the best load balancing strategy depends on the app's own concurrency model. Since Passenger does not know the app's concurrency model, the user will have to specify the desired load balancing strategy.

## 3. User experience

Let's say we have an application executable /webapps/fooapp/fooexe. We want Passenger to require just two pieces of information:

 1. The app's directory.
 2. A shell command string for starting the app on a certain port.

We want to allow two ways to specify the command string:

 1. Through a user config option.

    Rationale: this is required. Users may want to run a generic app that they can't modify.

 2. Through a Passengerfile.json supplied by the app's developer.

    Rationale: developers may want to help users out by supplying a command string, even if they don't bother modifying the app's code for Kuria support.

### 3.1. User configured start command

The config `app_start_command` (and other integration mode equivalents) should be used to tell Passenger that this is a generic app, and how to start it.

Nginx example:

~~~nginx
server {
    listen 80;
    server_name foo.com;
    root /webapps/fooapp/public;
    passenger_enabled on;
    passenger_app_root /webapps/fooapp;
    passenger_app_start_command './fooexe --port=$PORT';
}
~~~

Apache example:

~~~
<VirtualHost *:80>
    ServerName foo.com
    DocumentRoot /webapps/fooapp/public
    PassengerAppRoot /webapps/fooapp
    PassengerAppStartCommand './fooexe --port=$PORT'
</VirtualHost>
~~~

Standalone example (Passengerfile.json):

~~~json
{
    "app_start_command": "./fooexe --port=$PORT"
}
~~~

### 3.2. Developer-provided start command: Passengerfile.json

The config `app_start_command` in Passengerfile.json (so basically the same as the above Standalone example) should be used as a mechanism for the app developer to specify that this is a generic app that should be started with a specific command.

No changes in Standalone required (obviously). But Nginx and Apache should check whether a Passengerfile.json exists in the app root and whether it contains `app_start_command`. If so then it'll use that.

Example Nginx config for an app with a Passengerfile.json containing `app_start_command`:

~~~nginx
server {
    listen 80;
    server_name foo.com;
    root /webapps/fooapp/public;
    passenger_enabled on;
    passenger_app_root /webapps/fooapp;

    # No passenger_app_start_command, yay!
}
~~~

### 3.3. Concurrency configuration

The user should also specify the config `force_max_concurrent_requests_per_process` (and other integration mode equivalents) in order to tell Passenger about the concurrency properties of the app.

For example, if the app uses a one-connection-per-thread model, and the number of threads is configured via a command line option, then one can specify the following Nginx config:

~~~nginx
server {
    listen 80;
    server_name foo.com;
    root /webapps/fooapp/public;
    passenger_enabled on;
    passenger_app_root /webapps/fooapp;

    # Start the app with 32 threads.
    passenger_app_start_command './fooexe --port=$PORT --threads=32';
    # Tell Passenger that one process can handle at most 32 concurrent requests.
    passenger_force_max_concurrent_requests_per_process 32;
}
~~~

If `force_max_concurrent_requests_per_process` is not specified then Passenger should assume that the number of concurrent requests a process can handle is unlimited.

## 4. Tell us about your use cases

What do you think of this proposal? What use cases would you use this for, and how can we make Passenger accomodate your use case better? Please let us know by contacting us or posting a comment!