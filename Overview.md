# How Passenger supports multiple programming languages

Passenger supports different programming languages through different mechanics. This guide explains:

 * Which mechanisms Passenger supports.
 * What the differences between the mechanisms are (including pros and cons).
 * How these mechanisms relate to different programming languages and to the way one configures Passenger.

## Overview

Passenger works with apps written in any programming language by communicating with them via local HTTP sockets. Thus, Passenger does not care which language an app is written in. Passenger only needs to be able to spawn an app in a way that:

 * The app listens on a random local address.
 * Passenger knows which address the app listens on.

Passenger supports two mechanisms for spawning apps:

 * **Generic** — any kind of app (that can listen on a TCP port) can be supported through this mechanism, as long as one can specify a command string for starting the app on a certain port.
 * **Auto-support** — Passenger has built-in support for a limited number of programming languages (mostly interpreted ones). One does not need to specify how to start the app; Passenger already knows what to do.

   This is the classic mechanism used by Passenger, i.e. the only mechanism supported by Passenger versions prior to 6.

> NOTE: The spawning mechanisms described here should not be confused with the smart vs direct spawn methods. The latter is a completely orthogonal concept that deal with copy-on-write preloading of apps.

## Comparison

Both mechanisms have pros and cons:

<table>
  <thead>
    <tr>
      <th>Property</th>
      <th>Generic</th>
      <th>Auto-support</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Language support</td>
      <td>Broad</td>
      <td>Limited</td>
    </tr>
    <tr>
      <td>Required information about app</td>
      <td>Start command</td>
      <td>Entrypoint filename (only if different from default)</td>
    </tr>
    <tr>
      <td>Startup error diagnostics possibilities</td>
      <td>Limited</td>
      <td>Many</td>
    </tr>
    <tr>
      <td>Socket types</td>
      <td>TCP</td>
      <td>Unix domain (faster)</td>
    </tr>
  </tbody>
</table>

## How the generic mechanism works

The generic mechanism works as follows:

 1. The user supplies a command string which tells Passenger how the app can be started on a certain port. For example, when using the Nginx integration mode, one would specify this config:

        passenger_app_start_command './myapp --port $PORT';
    
    This tells Passenger what command it should execute to start the app.
    
    > NOTE: The command string is a shell command to be executed by /bin/sh, thus shell variables are allowed.

 2. Passenger looks for a free local TCP port on the system.
 3. Passenger spawns a process using the given command string, substituting `$PORT` with the found free port.
 4. Passenger waits until the port becomes in use. Passenger then considers the process to have successfully spawned.

This approach is similar to how the Heroku Procfile system works.

### Activation via `app_start_command`

The existance of the `app_start_command` option is what activates the generic mechanism in Passenger. `app_start_command` can either be specified in the integration-mode-specific configuration, or in Passengerfile.json. If neither are specified, then Passenger does not treat an app as generic.

### Error handling

Passenger considers a generic app to have failed to spawn if one of the following conditions occur:

 * The app exits before the port becomes in use.
 * The app closes its stdout channel.
 * The configured timeout expires.

When this happens, Passenger will dump an error report containing the app's stdout/stderr output, as well as various operating system-level diagnostics.

## How the auto-support mechanism works

The auto-support mechanism works through the use of "wrappers". Passenger has a separate wrapper program for each auto-supported programming language. When Passenger spawns an app, Passenger actually spawns the corresponding wrapper, not the app directly.

The wrapper then performs this work:

 1. It reads arguments passed from Passenger.
 2. It loads the target application and injects various behavior into the app.
 3. It sets up a mini-HTTP server on a random local server socket.
 4. It reports the socket's address back to Passenger.

### Error handling

Passenger considers an auto-supported app to have failed to spawn if one of the following conditions occur:

 * The app exits before the port becomes in use.
 * The app closes its stdout channel.
 * The configured timeout expires.
 * The app reports an error using a specific protocol.

When this happens, Passenger will dump an error report containing the app's stdout/stderr output, various operating system-level diagnostics, as well as any wrapper-specific error reports.

The wrapper-specific error reports allows contextual error diagnostics and troubleshooting hints. For example, when a Ruby app fails to spawn due to a Bundler error, the Ruby wrapper will include Bundler-specific troubleshooting advice in the error report.
