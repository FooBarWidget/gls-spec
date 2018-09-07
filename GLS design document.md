# Generic language support design document

Passenger only supports a few programming languages right now. "Generic language support" — **"GLS"** — is what we call the effort to make Passenger support *all* programming languages. This document describes what the goal should look like (mostly from a UX point of view) and how we'll get there (the implementation strategy).

This document is meant for Passenger developers/contributors in order to get feedback on whether the UX is desirable and whether the implementation strategy is viable. But we welcome the public to comment as well, especially about the rationale and UX parts.

<!-- TOC depthFrom:2 depthTo:4 -->

- [1. Why GLS?](#1-why-gls)
- [2. What is GLS and how to get there?](#2-what-is-gls-and-how-to-get-there)
    - [2.1. The basic architecture is language-agnostic: process manager and reverse proxy](#21-the-basic-architecture-is-language-agnostic-process-manager-and-reverse-proxy)
    - [2.2. Application process spawning traditionally via wrappers](#22-application-process-spawning-traditionally-via-wrappers)
        - [2.2.1. Drawbacks](#221-drawbacks)
        - [2.2.2. Hardcoded list](#222-hardcoded-list)
    - [2.3. Recently introduced: non-wrapper approaches](#23-recently-introduced-non-wrapper-approaches)
    - [2.4. Summary of spawning approaches](#24-summary-of-spawning-approaches)
    - [2.5. Towards GLS](#25-towards-gls)
        - [2.5.1. We won't market Kuria apps yet](#251-we-wont-market-kuria-apps-yet)
        - [2.5.2. List of auto-supported not dynamically extensible](#252-list-of-auto-supported-not-dynamically-extensible)
- [3. User experience](#3-user-experience)
    - [3.1. Generic apps](#31-generic-apps)
        - [3.1.1. User configured start command](#311-user-configured-start-command)
        - [3.1.2. Developer-provided start command: Passengerfile.json](#312-developer-provided-start-command-passengerfilejson)
    - [3.2. Kuria apps](#32-kuria-apps)
- [4. Implementation direction](#4-implementation-direction)
    - [4.1. Setting the SpawningKit config](#41-setting-the-spawningkit-config)
    - [4.2. How user config and autodetected values flow](#42-how-user-config-and-autodetected-values-flow)
    - [4.3. New config options](#43-new-config-options)
    - [Standalone::AppFinder](#standaloneappfinder)
    - [4.4. Testing checklist](#44-testing-checklist)
- [5. Future work: extensions for more auto-supported apps](#5-future-work-extensions-for-more-auto-supported-apps)
    - [5.1. Anatomy of an extension](#51-anatomy-of-an-extension)
    - [5.2. Where to look for extensions](#52-where-to-look-for-extensions)
    - [5.3. Nginx config](#53-nginx-config)
    - [5.4. Apache config](#54-apache-config)
    - [5.5. Standalone config](#55-standalone-config)
    - [5.6. Extension installation experience](#56-extension-installation-experience)

<!-- /TOC -->

## 1. Why GLS?

Application delivery and DevOps are too hard. They shouldn't be. Passenger's goal is to improve this by rethinking the experience into something simpler and happier.

We started out with Ruby and slowly moved to cover Python and Node.js too. But with microservices and containers, the world is becoming increasingly polyglot. We see that every language reinvents tooling and duplicates efforts. The quality and usability also varies wildly.

Today Passenger helps more than 650.000 websites world-wide, such as Apple, Pixar and Intercom. But why stop here? With GLS, we want to bring the benefits of Passenger into the hands of more people. More importantly, we envision that Passenger brings _standardization_, which allows teams to move faster and with more confidence. One tool to rule them all.

> How could GLS benefit you? What use cases could we solve for you? Post a comment and let us know!

## 2. What is GLS and how to get there?

TLDR:

 * Passenger 5.3 introduced a rewrite of the SpawningKit subsystem — the subsystem responsible for spawning application processes.
 * This rewrite contains all the low-level machinery needed to spawn apps written in any language.
 * The rest of Passenger doesn't care what language an apps is written in, as long as they can communicate with the app.
 * So to implement GLS, we "simply" need to connect the rest of Passenger with the new machinery.

But of course, the devil is in the details. For example, how we write the connections depends on the UX implications.

### 2.1. The basic architecture is language-agnostic: process manager and reverse proxy

Passenger is basically a process manager and reverse proxy. It spawns application processes and makes them listen on a certain local port. Passenger sits in between app processes and HTTP clients and acts as a reverse proxy: it applies load balancing etc.

This basic architecture does not care which languages apps are written in. As long as Passenger knows how to spawn an app process and how to make it listen on a certain port, Passenger can do its job.

### 2.2. Application process spawning traditionally via wrappers

The "how to spawn an app process" part is handled by a subsystem named SpawningKit. Support for the current languages (Ruby, Python, Node.js) is implemented through "wrappers". When we spawn an app process, we actually start the Ruby/Python/Node.js wrapper, which is written in the same language. The wrapper:

 1. Reads arguments from Passenger (SpawningKit actually).
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

> For more details, see [the SpawningKit README](https://github.com/phusion/passenger/tree/configkit_nested/src/agent/Core/SpawningKit#readme).

SpawningKit has recently received a big overhaul that addresses both drawbacks. SpawningKit now supports two alternative approaches:

 1. A generic approach that works for all apps:
 
     1. The user supplies a command string which tells SpawningKit how the application can be started on a certain port.
     2. SpawningKit looks for a free port on the system.
     3. SpawningKit spawns a process using the given command string, substituting `$PORT` with the found free port.
     4. SpawningKit waits until the port becomes in use. SpawningKit then considers the process to have successfully spawned.

 2. An approach that does not use wrappers, but instead requires the application to have been manually modified to support **"Kuria"** — the SpawningKit protocol.

    1. The user supplies a command string which tells SpawningKit how the application can be started. No port parameter is needed.
    2. SpawningKit spawns a process using the given command string.
    3. The app process detects (through an environment variable) that it is spawned by SpawningKit, and initiates the use of the Kuria protocol.
    4. The app process finishes internal initialization, then listens on a random local port.
    5. The app process reports this port, as well as a success indicator, back to SpawningKit through the Kuria protocol.

SpawningKit did not do away with the old wrapper-based approach. That approach is still available.

### 2.4. Summary of spawning approaches

In summary, the current situation is that SpawningKit supports the following kinds of apps:

 1. **Generic apps** — any kind of app can be supported through this approach, as long as one can specify a command string for starting it on a certain port.

 2. **Kuria apps** — apps that have been manually modified to support the Kuria protocol. One needs to specify a command string for starting the app.

    The benefits compared to (1) is better startup error reporting and better performance.

 3. **Auto-supported apps** — apps for which a wrapper is available. This wrapper speaks the Kuria protocol.
 
    The benefits compared to (2) is the developer doesn't need to manually modify the app, and doesn't need to supply a command string (hence "auto-supported").
    
    But sometimes, the user or developer may have to specify a startup file, if it differs from the convention. For example when a Ruby app's Rack entrypoint is not `config.ru`.

### 2.5. Towards GLS

Even though SpawningKit internally supports all 3 approaches, Passenger currently only exposes user-visible mechanisms for the "auto-supported apps" approach. So we need to add mechanisms to allow the use of the "generic apps" and the "Kuria apps" approaches.

#### 2.5.1. We won't market Kuria apps yet

Let's do one thing at a time. When we initially launch GLS, we won't market Kuria apps yet. We'll do that later. And when we do we will also have to publish Kuria protocol specs.

#### 2.5.2. List of auto-supported not dynamically extensible

What about auto-supported apps? The list of wrappers is currently hardcoded. Do we want to allow the user to install more wrappers, similar to how one installs extra language syntax highlighting into an editor?

The answer is no, at least for now. It's more work and it's not clear whether users will want such a feature. Instead, people will have to contribute a wrapper to be included in Passenger. We shall eventually release documentation on how to contribute a wrapper.

## 3. User experience

How would things work from the user point of view?

### 3.1. Generic apps

Let's say we have an app in /webapps/fooapp/foo. We want Passenger to require just two pieces of information:

 1. The app's directory.
 2. A shell command string for starting the app on a certain port.

We want to allow two ways to specify the command string:

 1. Through a user config option.

    Rationale: this is required. Users may want to run a generic app that they can't modify.

 2. Through a Passengerfile.json supplied by the app's developer.

    Rationale: developers may want to help users out by supplying a command string, even if they don't bother modifying the app's code for Kuria support.

#### 3.1.1. User configured start command

The config `app_start_command` (and variants) should be used to tell Passenger that this is a generic app, and how to start it.

Nginx example:

~~~nginx
server {
    listen 80;
    server_name foo.com;
    passenger_enabled on;
    passenger_app_root /webapps/fooapp;
    passenger_app_start_command './foo --port=$PORT';
}
~~~

Apache example:

~~~
<VirtualHost *:80>
    ServerName foo.com
    PassengerAppRoot /webapps/fooapp
    PassengerAppStartCommand './foo --port=$PORT'
</VirtualHost>
~~~

Standalone example (Passengerfile.json):

~~~json
{
    "app_start_command": "./foo --port=$PORT"
}
~~~

#### 3.1.2. Developer-provided start command: Passengerfile.json

The config `app_start_command` in Passengerfile.json (so basically the same as the above Standalone example) should be used as a mechanism for the app developer to specify that this is a generic app that should be started with a specific command.

No changes in Standalone required (obviously). But if Nginx or Apache should check whether a Passengerfile.json exists in the app root and whether it contains `app_start_command`. If so then it'll use that.

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

Let's say we have a Go app in /webapps/fooapp/foo. This app has been modified by its developer for Kuria support. We want Passenger to require just three pieces of information:

 1. The app's directory (like in the generic case).
 2. A shell command string for starting the app on a certain port (like in the generic case).
 3. An indicator that this app supports Kuria.

Like in the generic case, we want to allow two ways to specify the command string:

 1. Through a user config option.
 2. Through a Passengerfile.json supplied by the app's developer.

But this time the rationale is different:

> One could argue that since the developer already modified the app, s/he may as well provide a start command in Passengerfile.json too. But that would make the start command fixed. The developer may want to allow the user to customize the start command with additional CLI flags (e.g. because the app does not support config files). Conversely, the user may want to override the default start command provided by the developer (e.g. for troubleshooting reasons).

Where should the Kuria support indicator be set? I think it should be in Passengerfile.json, in the same way `app_start_command` in Passengerfile.json works:

~~~javascript
{
    // Omited `--port=$PORT` compared to the generic case. Not necessary because Kuria apps autodetect whether Passenger is requesting the Kuria protocol.
    "app_start_command": "./myapp",
    "app_supports_kuria_protocol": true
}
~~~

Nginx/Apache config then only requires `passenger_app_root`. Nginx/Apache sees that Passengerfile.json contains `app_supports_kuria_protocol: true` and autodetects it as such. Passenger will use the `app_start_command` specified in Passengerfile.json.

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
    passenger_app_start_command './myapp --dbpath=/home/db/dir';
}
~~~

## 4. Implementation direction

### 4.1. Setting the SpawningKit config

The 3 approaches listed in "Summary of spawning approaches" correspond to the following SpawningKit config options:

| Approach       | SpawningKit config                                   |
|----------------|------------------------------------------------------|
| Generic        | genericApp = true,<br> startsUsingWrapper irrelevant,<br> appType irrelevant,<br> startupFile irrelevant,<br> startCommand = [something] |
| Kuria          | genericApp = false,<br> startsUsingWrapper = false,<br> appType irrelevant,<br> startupFile irrelevant,<br> startCommand = (something) |
| Auto-supported | genericApp = false,<br> startsUsingWrapper = true,<br> appType = (ruby,python,nodejs),<br> startupFile = (something),<br> startCommand = (path to wrapper) |

Passenger currently only configures SpawningKit to use the auto-supported approach. We need to set the SpawningKit config as follows:

 * If Passengerfile.json sets `app_supports_kuria_protocol`, then:
    - Ensure `genericApp = false`
    - Ensure `startsUsingWrapper = false`
    - Ensure `startCommand = (value from app_start_command config)`. If not provided, present an error.
 * Otherwise, if the `passenger_app_start_command` config (or equivalent) is set, or if Passengerfile.json sets `app_start_command` (even in case of Nginx/Apache), then:
    - Ensure `genericApp = true`
    - Ensure `startCommand = (value from app_start_command config)`. If not provided, present an error.
 * Otherwise, ensure `genericApp = false`.

Where do we set the SpawningKit config? And since we need to know the values of `app_supports_kuria_protocol` and `app_start_command`, how do we obtain those values from where we set the SpawningKit config? The following sections will guide you.

### 4.2. How user config and autodetected values flow

`appType` and `startupFile` are set either based on user-provided config, or are autodetected, through this flow:

<a href="SpawningKit%20config%20flow.png">![SpawningKit config flow](https://github.com/FooBarWidget/gls-spec/blob/master/SpawningKit%20config%20flow.png?raw=true)</a>

`startCommand` is set based on `appType` and the **wrapper registry**. The wrapper registry is a hard-coded in-memory database which maps an app type name to various information, such as the name of the corresponding wrapper. The value is calculated in `ApplicationPool2::Options::getStartCommand()`.

The diagram shows that **`SpawningKit::Spawner::setConfigFromAppPoolOptions()`** is where the SpawningKit config is set. From there, you have access to an `ApplicationPool2::Options` object. That object contains the values of:

 * `appType` (from user config or autodetected)
 * `startupFile` (from user config or autodetected)
 * `startCommand` (auto calculated based on wrapper registry)

So to access `app_supports_kuria_protocol` and `app_start_command`, you need to:

 1. Add an `appSupportsKuriaProtocol` field to ApplicationPool2::Options.

    There is already a `startCommand` field. We can use it after renaming this to `appStartCommand`. This is safe, see below.

 2. Modify the flow so that the Core controller sets those two fields.

> Yeah, the existing `startCommand` field is an oddball. It is safe to rename and reuse this field because it is only used by unit tests.
>
> `ApplicationPool2::Options::getStartCommand()` skips the auto calculation if the `startCommand` field is already set. In the Core controller's `createNewPoolOptions()`, that field is set based on the `!~PASSENGER_START_COMMAND` header. But this is a mechanism reserved for future use that never actually got used. Nginx/Apache do not set this header: they don't have a config option for doing so.
>
> The next section will suggest introducing a `passenger_app_start_command` option. So just rename the `!~PASSENGER_START_COMMAND` header to `!~PASSENGER_APP_START_COMMAND` and use the header for that option.

### 4.3. New config options

We need to introduce the following new config options:

  * Nginx:

     - `passenger_app_start_command` (per-app)

  * Apache:

     - `PassengerAppStartCommand` (per-app)

  * Standalone/Passengerfile.json:

     - `app_start_command`
     - `app_supports_kuria_protocol`

> Note: in the Standalone + builtin engine case you will need to introduce a `single_app_mode_app_start_command` config to the Core controller ConfigKit schema, in the same way `single_app_mode_app_type` is handled.

The value of `passenger_app_start_command` (and equivalent) will flow into `ApplicationPool2::Options.appStartCommand`. But according to the UX spec, Nginx and Apache should not need `passenger_app_start_command` if Passengerfile.json already contains `app_start_command`. How do we handle this?

Let's do this in the same way that the value of `appType` is autodetected if it isn't set by the user config:

 * Modify the `AppTypeDetector` to parse Passengerfile.json. Ensure that its Result struct contains the value of `app_start_command`.
 * In the Nginx/Apache module, if `passenger_app_start_command` is not set, autodetect it and set the `!~PASSENGER_APP_START_COMMAND` header based on that.

   Mind the performance here. Parsing JSON is expensive so we don't want to do that on every request. You should look for a way to cache the result. The current caching mechanism using CachedFileStat assumes that the only expensive work done by the AppTypeDetector is stat()ing files, but that won't be true anymore.

 * In the Core's `initSingleAppMode()`, autodetect the value of `single_app_mode_app_start_command` if not set.

What about `appSupportsKuriaProtocol`? That should be set to true if and only if Passengerfile.json contains `app_supports_kuria_protocol`. Again, let's do this in the same way as `appType`:

TODO is this the right approach? doublecheck again "Setting the SpawningKit config"

 * Modify the `AppTypeDetector`'s Result struct and ensure that it contains the value of Passengerfile.json's `app_supports_kuria_protocol`.
 * In the Nginx/Apache module, autodetect it and set the `!~PASSENGER_APP_START_COMMAND` header based on that.

   Mind the performance here. Parsing JSON is expensive so we don't want to do that on every request. You should look for a way to cache the result. The current caching mechanism using CachedFileStat assumes that the only expensive work done by the AppTypeDetector is stat()ing files, but that won't be true anymore.

### Standalone::AppFinder

Passenger Standalone's AppFinder class (written in Ruby) is another oddball that you need to deal with. This class is sort of a lightweight version of the C++ AppTypeDetector class and the WrapperRegistry combined.

Passenger Standalone uses AppFinder to detect whether a directory contains an app. The result is then used to raise an error if one tries to start Passenger Standalone in a non-app directory. Passenger Enterprise also uses AppFinder as part of the "mass deployment" feature.

The key method is `#looks_like_app_directory?` which returns a boolean. It's similar to `AppTypeDetector::checkAppRoot()`: it checks whether any of the startup files in the WrapperRegistry exists in a given directory.

...TODO

> It's too bad that we need to duplicate the C++ stuff here. In the future we should find a way to unify these, maybe by having Passenger Standalone outsource this work to a CLI invocation of the PassengerAgent. But then Passenger Standalone would have to ensure that the PassengerAgent is compiled, before it starts scanning app directories.

### 4.4. Testing checklist

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

## 5. Future work: extensions for more auto-supported apps

> In section "Towards GLS" we said that we don't want to allow users to install extra wrappers, and that all wrappers shall be bundled with Passengers. This section explores how the UX could look like if we ever want to allow installing extra wrappers, and explores UX and implementation implications.

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
