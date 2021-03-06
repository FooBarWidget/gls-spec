# Generic language support design document

Passenger only supports a few programming languages right now. "Generic language support" — **"GLS"** — is what we call the effort to make Passenger support *all* programming languages. This document describes what the goal should look like (mostly from a UX point of view) and how we'll get there (the implementation strategy).

This document is meant for Passenger developers and contributors in order to get feedback on whether the UX is desirable and whether the implementation strategy is viable. But we welcome the public to comment as well, especially about the rationale and UX parts.

<!-- TOC depthFrom:2 depthTo:4 -->

- [1. Why GLS?](#1-why-gls)
- [2. What is GLS and how to get there?](#2-what-is-gls-and-how-to-get-there)
    - [2.1. Language-agnostic basic architecture: process manager and reverse proxy](#21-language-agnostic-basic-architecture-process-manager-and-reverse-proxy)
    - [2.2. Application process spawning traditionally via wrappers](#22-application-process-spawning-traditionally-via-wrappers)
        - [2.2.1. Drawbacks](#221-drawbacks)
        - [2.2.2. Hardcoded list](#222-hardcoded-list)
    - [2.3. Recently introduced: non-wrapper approaches](#23-recently-introduced-non-wrapper-approaches)
        - [2.3.1. How the generic approach works](#231-how-the-generic-approach-works)
        - [2.3.2. How the Kuria approach works](#232-how-the-kuria-approach-works)
    - [2.4. Towards GLS](#24-towards-gls)
        - [2.4.1. Keeping Kuria support stealth in initial launch](#241-keeping-kuria-support-stealth-in-initial-launch)
        - [2.4.2. List of auto-supported apps not dynamically extensible](#242-list-of-auto-supported-apps-not-dynamically-extensible)
- [3. User experience](#3-user-experience)
    - [3.1. Generic apps](#31-generic-apps)
        - [3.1.1. User configured start command](#311-user-configured-start-command)
        - [3.1.2. Developer-provided start command: Passengerfile.json](#312-developer-provided-start-command-passengerfilejson)
    - [3.2. Kuria apps](#32-kuria-apps)
        - [3.2.1. Kuria support indicator](#321-kuria-support-indicator)
- [4. Implementation direction](#4-implementation-direction)
    - [4.1. Setting the SpawningKit config](#41-setting-the-spawningkit-config)
    - [4.2. How user config and autodetected values flow](#42-how-user-config-and-autodetected-values-flow)
    - [4.3. How to access the `app_start_command` value from `setConfigFromAppPoolOptions()`](#43-how-to-access-the-app_start_command-value-from-setconfigfromapppooloptions)
    - [4.4. How to access `app_supports_kuria_protocol` from `setConfigFromAppPoolOptions()`](#44-how-to-access-app_supports_kuria_protocol-from-setconfigfromapppooloptions)
    - [4.5. New config options](#45-new-config-options)
    - [4.6. Standalone::AppFinder](#46-standaloneappfinder)
    - [4.7. Testing checklist](#47-testing-checklist)
- [5. Future work: extensions for more auto-supported apps](#5-future-work-extensions-for-more-auto-supported-apps)
    - [5.1. Anatomy of an extension](#51-anatomy-of-an-extension)
    - [5.2. Where to look for extensions](#52-where-to-look-for-extensions)
    - [5.3. Nginx config](#53-nginx-config)
    - [5.4. Apache config](#54-apache-config)
    - [5.5. Standalone config](#55-standalone-config)
    - [5.6. Extension installation experience](#56-extension-installation-experience)

<!-- /TOC -->

## 1. Why GLS?

Application delivery and DevOps are too hard. They shouldn't be. Passenger's goal is to improve this, by rethinking this into a simpler experience that is a joy to use and that allows teams to move faster.

We started out with Ruby and slowly moved to cover Python and Node.js as well. But with microservices and containers, the world is becoming increasingly polyglot. We see that every language reinvents tooling and duplicates efforts. Not only is this a waste, it is also so that the quality and usability of such efforts vary wildly.

Today Passenger helps more than 650.000 websites world-wide, such as Apple, Pixar and Intercom. But why stop here? The GLS effort is the first step towards:

 * Bringing the benefits of Passenger into the hands of more people. We envision that Passenger brings _standardization_, which allows teams to move faster and with more confidence. One tool to rule them all.
 - Rethinking Passenger's concept to address tomorrow's needs.

   > **How could GLS benefit you?** What use cases could we help you with? Post a comment and let us know!

## 2. What is GLS and how to get there?

TLDR:

 * [Passenger 5.3](https://blog.phusion.nl/2018/05/09/passenger-5-3-0/) introduced a rewrite of the SpawningKit subsystem — the subsystem responsible for spawning application processes.
 * This rewrite contains all the low-level machinery needed to spawn apps written in any language.
 * The rest of Passenger doesn't care what language an app is written in, as long as they can communicate with the app.
 * So to implement GLS, we "simply" need to connect the rest of Passenger with the new machinery.
 * In order to save time and release earlier, we're not going to expose all low-level machinery and features yet. We'll prioritize the most important features and do the rest in a future release.

But of course, the devil is in the details. For example, how we connect the low-level ⬌ high-level machinery depends on the UX implications.

### 2.1. Language-agnostic basic architecture: process manager and reverse proxy

<a href="https://github.com/FooBarWidget/gls-spec/blob/master/Passenger-SpawningKit%20interaction.png?raw=true">![SpawningKit config flow](https://github.com/FooBarWidget/gls-spec/blob/master/Passenger-SpawningKit%20interaction.png?raw=true)</a>
<!-- <a href="https://github.com/FooBarWidget/gls-spec/blob/master/Passenger-SpawningKit%20interaction.png?raw=true">![SpawningKit config flow](Passenger-SpawningKit%20interaction.png)</a> -->

> This simplified interaction diagram shows how Passenger outsources the task of spawning an app process to a subsystem named SpawningKit. The HTTP server and reverse proxy parts of Passenger don't care what language that process is written in, as long as it can communicate with that process over a socket.

Passenger is basically a process manager and reverse proxy. It spawns application processes and makes them listen on a certain local port. Passenger sits in between app processes and HTTP clients and acts as a reverse proxy: it applies load balancing etc.

This basic architecture **does not care which languages apps are written in**. As long as Passenger knows how to spawn an app process and how to make it listen on a certain port, Passenger can do its job.

### 2.2. Application process spawning traditionally via wrappers

The "how to spawn an app process" part is handled by a subsystem named SpawningKit. Support for the current languages (Ruby, Python, Node.js) is implemented through "wrappers". When we spawn an app process, we actually start the Ruby/Python/Node.js wrapper, which is written in the same language. The wrapper:

 1. Reads arguments from SpawningKit (whose values are passed by higher-level mechanisms in Passenger).
 2. Loads the target application and injects various behavior into the app.
 3. Sets up a local server socket.
 4. Reports the socket's port information back to SpawningKit.

#### 2.2.1. Drawbacks

This wrapper-based approach has drawbacks:

 1. It only works for interpreted (or more generally: dynamically loadable) languages. An operation like "loads the target application" is not available for languages like Go or C++.
 2. It requires a wrapper to be written for every language we want to support.

#### 2.2.2. Hardcoded list

The list of available wrappers is stored in the **wrapper registry**. The contents of this registry is currently hardcoded.

### 2.3. Recently introduced: non-wrapper approaches

> See [the SpawningKit README](https://github.com/phusion/passenger/tree/master/src/agent/Core/SpawningKit#readme) for more details on SpawningKit.

SpawningKit has recently received a big overhaul that addresses both drawbacks of the wrapper-based approach. SpawningKit now supports two alternative approaches, so in total three kinds of apps are now supported:

 1. **Generic apps** (new!) — any kind of app (that can listen on a TCP port) can be supported through this approach, as long as one can specify a command string for starting it on a certain port.

 2. **Kuria apps** (new!) — apps that have been manually modified to support the Kuria protocol. One needs to specify a command string for starting the app.

    The benefits compared to (1) is better startup error reporting and better performance.

 3. **Auto-supported apps**  — apps for which a wrapper is available. This wrapper speaks the Kuria protocol.

    This is the "classic" wrapper-based approach, but it is not supported for merely backwards-compatibility reasons. The benefits compared to (2) is the developer doesn't need to manually modify the app, and doesn't need to supply a command string (hence "auto-supported").
    
    But sometimes, the user or developer may have to specify a startup file, if it differs from the convention. For example when a Ruby app's Rack entrypoint is not `config.ru`.

#### 2.3.1. How the generic approach works

How does the generic approach allow SpawningKit support nearly all apps? Here's how it works:

 1. The user supplies a command string which tells SpawningKit how the app can be started on a certain port.
 2. SpawningKit looks for a free port on the system.
 3. SpawningKit spawns a process using the given command string, substituting `$PORT` with the found free port.
 4. SpawningKit waits until the port becomes in use. SpawningKit then considers the process to have successfully spawned.

This approach is similar to how the Heroku Procfile system works.

#### 2.3.2. How the Kuria approach works

This approach is similar to the generic approach works in that it does not use wrappers and instead requires a command string.

There is one additional requirement: the app must have been manually modified to support **"Kuria"** — the SpawningKit protocol.

 1. The user supplies a command string which tells SpawningKit how the app can be started. No port parameter is needed.
 2. SpawningKit spawns a process using the given command string.
 3. The app process detects (through an environment variable) that it is spawned by SpawningKit, and initiates the use of the Kuria protocol.
 4. The app process finishes internal initialization, then listens on a random local port.
 5. The app process reports this port, as well as a success indicator, back to SpawningKit through the Kuria protocol.

### 2.4. Towards GLS

<a href="https://github.com/FooBarWidget/gls-spec/blob/master/Spawning%20approaches.png?raw=true">![SpawningKit config flow](https://github.com/FooBarWidget/gls-spec/blob/master/Spawning%20approaches.png?raw=true)</a>
<!-- <a href="https://github.com/FooBarWidget/gls-spec/blob/master/Spawning%20approaches?raw=true">![SpawningKit config flow](Spawning%20approaches.png)</a> -->

Even though SpawningKit internally supports all 3 approaches, Passenger currently only exposes user-visible mechanisms for the "auto-supported apps" approach. So we need to add mechanisms to allow the use of the "generic apps" and the "Kuria apps" approaches.

#### 2.4.1. Keeping Kuria support stealth in initial launch

Implementing support for Kuria apps is not much more work compared to implementing support for generic apps (as is apparent from section "Implementation direction"). However, documenting the Kuria protocol and writing related documentation such as tutorials *do* take quite some time. Therefore, for the initial GLS launch, we should implement Kuria support, but keep it under the radar. We'll market Kuria in a future release.

#### 2.4.2. List of auto-supported apps not dynamically extensible

What about auto-supported apps? The list of wrappers is currently hardcoded. Do we want to allow the user to install more wrappers, similar to how one installs extra language syntax highlighting into an editor?

Answer: no, not for now. The work required to make this work is much more than that of implementing Kuria support. It's also not clear whether users will want such a feature.

For now, people will have to contribute a wrapper to be included in Passenger. We should eventually release documentation on how to contribute a wrapper.

## 3. User experience

How would things work from the user point of view?

### 3.1. Generic apps

Let's say we have an application executable /webapps/fooapp/fooexe. We want Passenger to require just two pieces of information:

 1. The app's directory.
 2. A shell command string for starting the app on a certain port.

We want to allow two ways to specify the command string:

 1. Through a user config option.

    Rationale: this is required. Users may want to run a generic app that they can't modify.

 2. Through a Passengerfile.json supplied by the app's developer.

    Rationale: developers may want to help users out by supplying a command string, even if they don't bother modifying the app's code for Kuria support.

#### 3.1.1. User configured start command

The config `app_start_command` (and equivalents) should be used to tell Passenger that this is a generic app, and how to start it.

Nginx example:

~~~nginx
server {
    listen 80;
    server_name foo.com;
    passenger_enabled on;
    passenger_app_root /webapps/fooapp;
    passenger_app_start_command './fooexe --port=$PORT';
}
~~~

Apache example:

~~~
<VirtualHost *:80>
    ServerName foo.com
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

#### 3.1.2. Developer-provided start command: Passengerfile.json

The config `app_start_command` in Passengerfile.json (so basically the same as the above Standalone example) should be used as a mechanism for the app developer to specify that this is a generic app that should be started with a specific command.

No changes in Standalone required (obviously). But Nginx and Apache should check whether a Passengerfile.json exists in the app root and whether it contains `app_start_command`. If so then it'll use that.

Example Nginx config for an app with a Passengerfile.json containing `app_start_command`:

~~~nginx
server {
    listen 80;
    server_name foo.com;
    passenger_enabled on;
    passenger_app_root /webapps/fooapp;

    # No passenger_app_start_command, yay!
}
~~~

### 3.2. Kuria apps

Let's say we have a Go app executable in /webapps/fooapp/fooexe. This app has been modified by its developer for Kuria support. We want Passenger to require just three pieces of information:

 1. The app's directory (like in the generic case).
 2. A shell command string for starting the app (like in the generic case). Unlike the generic case, no port parameter is required because the app will auto-detect whether it needs to initiate the Kuria protocol.
 3. An indicator that this app supports Kuria.

Like in the generic case, we want to allow two ways to specify the command string:

 1. Through a user config option.
 2. Through a Passengerfile.json supplied by the app's developer.

But this time the rationale is different:

One could argue that since the developer already modified the app, s/he may as well provide a start command in Passengerfile.json too. But that would make the start command fixed. The developer may want to allow the user to customize the start command with additional CLI flags (e.g. because the app does not support config files). Conversely, the user may want to override the default start command provided by the developer (e.g. for troubleshooting reasons).

#### 3.2.1. Kuria support indicator

Where should the Kuria support indicator be set? I think it should be in Passengerfile.json, in the same way `app_start_command` in Passengerfile.json works:

~~~javascript
{
    // Omitted `--port=$PORT` compared to the generic case. Not necessary because
    // Kuria apps autodetect whether Passenger is requesting the Kuria protocol.
    "app_start_command": "./fooexe",
    "app_supports_kuria_protocol": true
}
~~~

The Nginx/Apache config then only requires `passenger_app_root`. Nginx/Apache sees that Passengerfile.json contains `app_supports_kuria_protocol: true` and autodetects it as such. Passenger will use the `app_start_command` specified in Passengerfile.json.

~~~nginx
server {
    listen 80;
    server_name foo.com;
    passenger_enabled on;
    passenger_app_root /webapps/fooapp;
}
~~~

The user can also choose to override the start command:

~~~nginx
server {
    listen 80;
    server_name foo.com;
    passenger_enabled on;
    passenger_app_root /webapps/fooapp;

    # Overridden start command:
    passenger_app_start_command './fooexe --dbpath=/home/db/dir';
}
~~~

## 4. Implementation direction

### 4.1. Setting the SpawningKit config

The 3 approaches listed in "Recently introduced: non-wrapper approaches" correspond to the following SpawningKit config options:

| Approach       | SpawningKit config                                   |
|----------------|------------------------------------------------------|
| Generic        | genericApp = true,<br> startsUsingWrapper irrelevant,<br> appType irrelevant,<br> startupFile irrelevant,<br> startCommand = [something] |
| Kuria          | genericApp = false,<br> startsUsingWrapper = false,<br> appType irrelevant,<br> startupFile irrelevant,<br> startCommand = (something) |
| Auto-supported | genericApp = false,<br> startsUsingWrapper = true,<br> appType = (ruby,python,nodejs),<br> startupFile = (something),<br> startCommand = (path to wrapper) |

Passenger currently only configures SpawningKit to use the auto-supported approach. We should set the SpawningKit config as follows:

 * If Passengerfile.json sets `app_supports_kuria_protocol`, then:
    - Ensure `genericApp = false`
    - Ensure `startsUsingWrapper = false`
    - Ensure `startCommand = (value from integration mode-specific app_start_command config)`. If not provided, present an error.
 * Otherwise, if the integration mode-specific `app_start_command` config is set, or if Passengerfile.json sets `app_start_command` (even in case of Nginx/Apache), then:
    - Ensure `genericApp = true`
    - Ensure `startCommand = (value of app_start_command, from either integration-mode-specific config, or from Passengerfile.json)`. If neither provided, present an error.
 * Otherwise, keep current behavior which is:
    - Ensure `genericApp = false`
    - Ensure `appType = (value from integration mode-specific app_type config, or autodetected)`
    - Ensure `startupFile = (value from integration mode-specific startup_file config, or autodetected)`
    - Ensure `startCommand` is set (later sections will explain how this works)

Where do we set the SpawningKit config? And since we need to know the values of `app_supports_kuria_protocol` and `app_start_command` (the latter of which from two sources: integration mode-specific, and Passengerfile.json), how do we obtain those values from where we set the SpawningKit config? The following sections will guide you.

### 4.2. How user config and autodetected values flow

Before we describe how to access `app_supports_kuria_protocol` and `app_start_command`, let's step back and take a look at how other values flow into the place where the SpawningKit config is set.

`appType` and `startupFile` are set either based on user-provided config, or are autodetected, through this flow:

<a href="https://github.com/FooBarWidget/gls-spec/blob/master/SpawningKit%20config%20flow.png?raw=true">![SpawningKit config flow](https://github.com/FooBarWidget/gls-spec/blob/master/SpawningKit%20config%20flow.png?raw=true)</a>
<!-- <a href="https://github.com/FooBarWidget/gls-spec/blob/master/SpawningKit%20config%20flow.png?raw=true">![SpawningKit config flow](SpawningKit%20config%20flow.png)</a> -->

`startCommand` is set based on `appType` and the **wrapper registry**. The wrapper registry is a hard-coded in-memory database which maps an app type name to various information, such as the name of the corresponding wrapper. The final `startCommand` value is calculated in `ApplicationPool2::Options::getStartCommand()`.

The diagram shows that **`SpawningKit::Spawner::setConfigFromAppPoolOptions()`** is where the SpawningKit config is set. From there, you have access to an `ApplicationPool2::Options` object. That object contains the values of:

 * `appType` (from user config or autodetected)
 * `startupFile` (from user config or autodetected)
 * `startCommand` (from user config; but the final value passed to SpawningKit will be calculated with `getStartCommand()`)

### 4.3. How to access the `app_start_command` value from `setConfigFromAppPoolOptions()`

In this section, let's call the `ApplicationPool2::Options` object as follows: `poolOptions`.

There is already a `poolOptions.startCommand` field. We should rename `poolOptions.startCommand` to `poolOptions.appStartCommand` (this is safe, see the notes below).

Then we should modify the flow so that the Core controller sets `poolOptions.startCommand`. We'll describe this in section "New config options".

> Yeah, the existing `poolOptions.startCommand` field is an oddball. It is safe to rename and reuse this field because it is only used by unit tests.
>
> `ApplicationPool2::Options::getStartCommand()` skips the auto calculation if `poolOptions.startCommand` is already set. In the Core controller's `createNewPoolOptions()`, that field is set based on the `!~PASSENGER_START_COMMAND` header. But this is a mechanism reserved for future use that never actually got used. Nginx/Apache do not set this header: they don't have a config option for doing so.
>
> A later section will suggest introducing a `passenger_app_start_command` config option, which will cause `!~PASSENGER_APP_START_COMMAND` to be passed to the Core controller. So just modify the Core controller code that uses `!~PASSENGER_START_COMMAND`, and make it use `!~PASSENGER_APP_START_COMMAND` instead.

### 4.4. How to access `app_supports_kuria_protocol` from `setConfigFromAppPoolOptions()`

We should make `setConfigFromAppPoolOptions()` check Passengerfile.json directly.

We could have chosen to have higher layers (such as the web server module) check for `app_supports_kuria_protocol` in Passengerfile.json, and then pass that information to lower layers, like how `app_type`'s value is passed down. But I see no reason why we should go through this trouble in case of `app_supports_kuria_protocol`. This value is not user-configurable and depends purely on Passengerfile.json.

### 4.5. New config options

We should introduce the following new config options:

  * Nginx:

     - `passenger_app_start_command` (per-app)

  * Apache:

     - `PassengerAppStartCommand` (per-app)

  * Standalone/Passengerfile.json:

     - `app_start_command`
     - `app_supports_kuria_protocol`

> Don't forget: in the Standalone + builtin engine case we should also introduce a `single_app_mode_app_start_command`/`single_app_mode_app_supports_kuria_protocol` config to the Core controller ConfigKit schema, in the same way `single_app_mode_app_type` is handled.

Making sure that the value of `passenger_app_start_command` (and equivalent) flows into `poolOptions.appStartCommand` is the easy part. But according to the UX spec, Nginx and Apache should not need `passenger_app_start_command` if Passengerfile.json already contains `app_start_command`. How do we handle this?

Let's do this in the same way that the value of `appType` is autodetected if it isn't set by the user config:

 * Modify the `AppTypeDetector` to parse Passengerfile.json. Ensure that its Result struct contains the value of `app_start_command`.
 * In the Nginx/Apache module, if `passenger_app_start_command` is not set, autodetect it and set the `!~PASSENGER_APP_START_COMMAND` header based on that.

   Mind the performance here. Parsing JSON is expensive so we don't want to do that on every request. You should look for a way to cache the result. The current caching mechanism using CachedFileStat assumes that the only expensive work done by the AppTypeDetector is stat()ing files, but that won't be true anymore.

 * In the Core's `initSingleAppMode()`, autodetect the value of `single_app_mode_app_start_command` if not set.

### 4.6. Standalone::AppFinder

Passenger Standalone's AppFinder class (written in Ruby) is sort of a lightweight version of the C++ AppTypeDetector class and the WrapperRegistry combined. Passenger Standalone uses AppFinder to detect whether a directory contains an app. The result is then used to raise an error if one tries to start Passenger Standalone in a non-app directory. Passenger Enterprise also uses AppFinder as part of the "mass deployment" feature.

The key method is `#looks_like_app_directory?` which returns a boolean. It's similar to `AppTypeDetector::checkAppRoot()`: it checks whether any of the startup files in the WrapperRegistry exists in a given directory. This method should return true as well if the `start_command` option is given.

> It's too bad that we need to duplicate some of the C++ stuff here. In the future we should find a way to unify these, maybe by having Passenger Standalone outsource this work to a CLI invocation of the PassengerAgent. But then Passenger Standalone would have to ensure that the PassengerAgent is compiled, before it starts scanning app directories.

### 4.7. Testing checklist

Tests should be automated. There should be suitable unit and integration tests at the right levels.

Check these configurations:

 * Do generic apps without `app_start_command` in Passengerfile.json work?
 * Do generic apps with `app_start_command` in Passengerfile.json work?
 * (Future) Do Kuria apps without `app_start_command` in Passengerfile.json work?
 * (Future) Do Kuria apps with `app_start_command` in Passengerfile.json work?

Check each configuration against:

  * Apache
  * Nginx
  * Standalone+Nginx
  * Standalone+Builtin
  * Standalone with Mass deployment
  * Flying Passenger

## 5. Future work: extensions for more auto-supported apps

> In section "Towards GLS" we said that we don't want to allow users to install extra wrappers, and that all wrappers shall be bundled with Passenger. This section explores how the UX could look like if we ever want to allow installing extra wrappers, and explores UX and implementation implications.

Let's say we have an Elixir app in /webapps/fooapp/myapp.elixir, and a Perl app in /webapps/barapp/myapp.pl.

Passenger does not automatically support Elixir and Perl (no wrappers). We want the user to be able to install "language support extensions" for Elixir and Perl support, which provides those wrappers.  After installation, Elixir/Perl apps would be supported in the same way Ruby apps are supported.

### 5.1. Anatomy of an extension

An extension is a _directory_ (although it could be packaged in .tar.gz). Example contents for a hypothetical Elixir extension:

    extension
     |
     +-- manifest.json (info about this language support extension)
     |
     +-- wrapper.elixir (the wrapper script; can be named anything)

The manifest.json hypothetically contains something like this:

~~~javascript
{
    // An identifier for this language, used by e.g. passenger_app_type.
    "name": "elixir",
    // Path to the wrapper, relative to this manifest.json,
    "wrapper": "wrapper.elixir",
    // The title that spawned Elixir processes should assume,
    "process_title": "Passenger ElixirApp",
    // A default command of the interpreter for this language.
    // Will be found in $PATH.
    "default_interpreter": "elixir",
    // A list of startup file names that Passenger should look for
    // in order to autodetect whether an app belongs to this language.
    "default_startup_files": ["main.elixir"]
}
~~~

Note that the contents map very well to the fields in `WrapperRegistry::Entry`.

### 5.2. Where to look for extensions

Where should Passenger look for extensions? I think that Passenger should look in the following directories by default. This list would be configurable via a config option `passenger_language_extension_dirs`. That option accepts a list of paths and **prepends to** (instead of replacing) the search list.

> Note: Why prepend instead of replace? This way there will always be a clear location to which extensions can be installed. If we allow replacing, then figuring out where extensions should be installed to requires querying the web server config, which requires Passenger to be running.
>
> This assumes that nobody would ever want to remove the default directories from the list. On weird systems /usr/local *might* be insecure and that might be a reason for excluding it from the list, but it's very hypothetical. I think we shouldn't bother with this until people actually present a real case.

 * Both OSS and enterprise:

    - /usr/local/share/passenger/extensions/language-support/[NAME]
    - /usr/share/passenger/extensions/language-support/NAME]

 * Enterprise only:

    - /usr/local/share/passenger-enterprise/extensions/language-support/[NAME]
    - /usr/share/passenger-enterprise/extensions/language-support/[NAME]

In addition, Passenger should also look in the following directories no matter the value of `passenger_language_extension_dirs`:

 * $HOME/.passenger/extensions/language-support/[NAME] (OSS & enterprise)
 * $HOME/.passenger-enterprise/extensions/language-support/[NAME] (enterprise only)

Here, $HOME is the home directory of the user that the Passenger watchdog runs as, before lowering privileges. So on most Nginx/Apache instances it will be /root. On most Standalone instances it will be the the home directory of the user that invoked Passenger Standalone.

Note that I specifically omitted a directory under `$PASSENGER_ROOT` (which would only be applicable to source, git, tarball or gem installations) as part of the default list! I don't see the point in that.

### 5.3. Nginx config

Suppose the user puts the Elixir extension in /opt/passenger/language-support/elixir, and the Perl extension in /var/lib/passenger-language-support/perl. The final Nginx config would look like this:

~~~nginx
passenger_language_extension_dirs /opt/passenger/language-support /var/lib/passenger-language-support;

server {
    listen 80;
    server_name foo.com;
    passenger_enabled on;
    passenger_app_root /webapps/fooapp;

    # If the startup file is not 'main.elixir' (as
    # specified in manifest.json's default_startup_files),
    # then also specify these:
    passenger_app_type elixir;
    passenger_startup_file myapp.elixir;
}

server {
    listen 80;
    server_name bar.com;
    passenger_enabled on;
    passenger_app_root /webapps/barapp;

    # If the startup file is not 'main.pl' (as
    # specified in manifest.json's default_startup_files),
    # then also specify these:
    passenger_app_type perl;
    passenger_startup_file myapp.pl;
}
~~~

### 5.4. Apache config

Apache equivalent of the Nginx config:

~~~
PassengerLanguageExtensionDirs /opt/passenger/language-support /var/lib/passenger-language-support

<VirtualHost *:80>
    ServerName foo.com
    PassengerAppRoot /webapps/fooapp

    # If the startup file is not 'main.elixir' (as
    # specified in manifest.json's default_startup_files),
    # then also specify these:
    PassengerAppType elixir
    PassengerStartupFile myapp.elixir
</VirtualHost>

<VirtualHost *:80>
    ServerName bar.com
    PassengerAppRoot /webapps/fooapp

    # If the startup file is not 'main.pl' (as
    # specified in manifest.json's default_startup_files),
    # then also specify these:
    PassengerAppType perl
    PassengerStartupFile myapp.pl
</VirtualHost>
~~~

### 5.5. Standalone config

Passenger Standalone equivalent of the Nginx config (Passengerfile.json):

~~~javascript
{
    "language_extension_dirs": [
        "/opt/passenger/language-support",
        "/var/lib/passenger-language-support"
    ],

    // If the startup file is not 'main.elixir' (as
    // specified in manifest.json's default_startup_files),
    // then also specify these:
    "app_type": "elixir",
    "startup_file": "myapp.elixir"
}
~~~

But there are some issue with the above. Passengerfile.json tends to be committed into the app's repository and thus needs to work on a wide range of machines. But the paths above are machine-dependent. Furthermore, I can imagine that users want to install/configure an extension just once and expect it to work no matter which app Passenger Standalone is serving.

So some kind of machine-local configuration — one that is separate from the app-specific configuration — is desirable. For now, an environment variable can do this job:

    export PASSENGER_LANGUAGE_EXTENSION_DIRS=/opt/passenger/language-support:/var/lib/passenger-language-support

But environment variables are error-prone. Even if users put the variable in .bashrc, they could still invoke Passenger Standalone from environments that don't load .bashrc. So in the long run we may want to introduce a global config file for Passenger Standalone.

### 5.6. Extension installation experience

We want installing an extension to be as easy as possible. Therefore we should supply an installation command. It accepts the filename of a packaged extension. Then it asks where to install it to:

    $ passenger-config install-extension passenger-elixir.tar.gz
    Language support extension detected: Elixir
    Docs on installing extensions: http://location-at-passenger-library

    Where do you want to install this extension to?

     >> System-wide directory
        Home directory (only works if Passenger runs as `<current user name>`)
        Cancel
    
    Extracting to /usr/local/passenger/extensions/language-support/elixir... done!

    All set! Restart Passenger and run this to check whether it worked:
    
        passenger-config list-extensions

The help link explains what it means to install an extension and what the differences are between the different directories.

The `--help` screen should display some basic help on what this command does and how extension installation works and where to get more information.

The tool installs only to /usr/local/passenger or ~/.passenger because these are guaranteed to be in the search list regardless of configuration. If users want to put the extension elsewhere then they should do it manually (the procedure for which should be made clear in the docs).

If the user selects "System-wide", then:

 1. If the command is already running as root, then it performs the extraction/copying directly.
 2. Otherwise, it performs the operation using sudo.
 3. Otherwise, if sudo is not available, then it aborts with an error, telling the user to run the command with root privileges.

`passenger-config list-extensions` should connect to the running Passenger instance, query the paths of all detected extensions, query the directories in which Passenger looks for extensions, and print all of these paths.

If it detects that the list of extensions is empty, then it should print a basic help message telling the user how to install an extension, as well as link to the docs regarding extensions.
