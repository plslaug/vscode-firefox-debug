# VS Code Debug Adapter for Firefox

A Visual Studio Code extension to debug your web application or browser extension in Firefox.

## Starting
You can use this extension in launch or attach mode. 

In launch mode it will start an instance of Firefox navigated to the start page of your application
and terminate it when you stop debugging.
You can also set the `reAttach` option in your launch configuration to `true`, in this case Firefox
won't be terminated at the end of your debugging session and the debugger will re-attach to it when
you start the next debugging session - this is a lot faster than restarting Firefox every time.
`reAttach` also works for add-on debugging: in this case, the add-on is (re-)installed as a
[temporary add-on](https://developer.mozilla.org/en-US/Add-ons/WebExtensions/Temporary_Installation_in_Firefox).

In attach mode the extension attaches to a running instance of Firefox (which must be manually
configured to allow remote debugging - see [below](#attach)).

To configure these modes you must create a file `.vscode/launch.json` in the root directory of your
project. You can do so manually or let VS Code create an example configuration for you by clicking 
the gear icon at the top of the Debug pane.
Finally, if `.vscode/launch.json` already exists in your project, you can open it and add a 
configuration snippet to it using the "Add Configuration" button in the lower right corner of the
editor.

### Launch
Here's an example configuration for launching Firefox navigated to the local file `index.html` 
in the root directory of your project:
```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch index.html",
            "type": "firefox",
            "request": "launch",
            "reAttach": true,
            "file": "${workspaceRoot}/index.html"
        }
    ]
}
```

You may want (or need) to debug your application running on a Webserver (especially if it interacts
with server-side components like Webservices). In this case replace the `file` property in your
`launch` configuration with a `url` and a `webRoot` property. These properties are used to map
urls to local files:
```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch localhost",
            "type": "firefox",
            "request": "launch",
            "reAttach": true,
            "url": "http://localhost/index.html",
			"webRoot": "${workspaceRoot}"
        }
    ]
}
```
The `url` property may point to a file or a directory, if it points to a directory it must end with
a trailing `/` (e.g. `http://localhost/my-app/`).

### Attach
To use attach mode, you have to launch Firefox manually from a terminal with remote debugging enabled.
Note that you must first configure Firefox to allow remote debugging. To do this, open the Firefox 
configuration page by entering `about:config` in the address bar. Then set the following preferences:

Preference Name                       | Value   | Comment
--------------------------------------|---------|---------
`devtools.debugger.remote-enabled`    | `true`  | Required
`devtools.chrome.enabled`             | `true`  | Required
`devtools.debugger.workers`           | `true`  | Required if you want to debug WebWorkers
`devtools.debugger.prompt-connection` | `false` | Recommended
`devtools.debugger.force-local`       | `false` | Set this only if you want to attach VS Code to Firefox running on a different machine (using the `host` property in the `attach` configuration)

Then close Firefox and start it from a terminal like this:

__Windows__

`"C:\Program Files\Mozilla Firefox\firefox.exe" -start-debugger-server -no-remote`

__OS X__

`/Applications/Firefox.app/Contents/MacOS/firefox -start-debugger-server -no-remote`

__Linux__

`firefox -start-debugger-server -no-remote`

Navigate to your web application and use this `launch.json` configuration to attach to Firefox:
```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch index.html",
            "type": "firefox",
            "request": "attach"
        }
    ]
}
```

If your application is running on a Webserver, you need to add the `url` and `webRoot` properties
to the configuration (as in the second `launch` configuration example above).

### Debugging Firefox add-ons
If you want to debug a Firefox add-on, you have to install the developer edition of Firefox. In
launch mode, it will automatically be used if it is installed in the default location.
If your add-on is developed with the add-on SDK, you also have to ensure that the `jpm` command
is in the system path.

Here's an example configuration for add-on debugging:
```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch addon",
            "type": "firefox",
            "request": "launch",
            "reAttach": true,
            "addonType": "addonSdk",
            "addonPath": "${workspaceRoot}"
        }
    ]
}
```
The `addonType` property must be set to `addonSdk`, `webExtension` or `legacy`, depending on the
type of your add-on. The `addonPath` must be the absolute path to the directory containing the
add-on manifest (`package.json` for `addonSdk` add-ons, `manifest.json` for `webExtension` add-ons
or `install.rdf` for `legacy` add-ons).

You can reload your add-on using the command "Firefox: Reload add-on" (`extension.firefox.reloadAddon`)
from the VS Code command palette. If you're using the add-on SDK, you can also use the command
"Firefox: Rebuild and reload add-on" (`extension.firefox.rebuildAndReloadAddon`) if you made changes
that influence the `install.rdf` file generated by `jpm`.
The add-on will also be reloaded when you restart the debugging session, unless you have set
`reloadOnAttach` to `false`.
You can also use the `reloadOnChange` property to let VS Code reload your add-on automatically
whenever you change a file.

### Optional configuration properties
* `reAttach`: If you set this option to `true` in a `launch` configuration, Firefox won't be 
  terminated at the end of your debugging session and the debugger will re-attach to it at the
  start of your next debugging session. If you're debugging an add-on developed with the add-on SDK,
  messages sent to the javascript console won't be shown in the VS Code debug console in `reAttach`
  mode.
* `reloadOnAttach`: This flag controls whether the web page(s) should be automatically reloaded
  after attaching to Firefox. The default is to reload in a `launch` configuration with the
  `reAttach` flag set to `true` and to not reload in an `attach` configuration.
* `reloadOnChange`: Automatically reload the Firefox tabs or your add-on whenever files change.
  You can specify single files, directories or glob patterns to watch for file changes and
  additionally specify files to be ignored. Since watching files consumes system resources,
  make sure that you are not watching more files than necessary.
  The following example will watch all javascript files in your workspace except those under
  `node_modules`:
  ```
    "reloadOnChange": {
        "watch": [ "${workspaceRoot}/**/*.js" ],
        "ignore": [ "${workspaceRoot}/node_modules/**" ]
    }
  ```
  By default, the reloading will be "debounced": the debug adapter will wait until the last file
  change was 100 milliseconds ago before reloading. This is useful if your project uses a build
  system that generates multiple files - without debouncing, each file would trigger a separate
  reload. You can use `reloadOnChange.debounce` to change the debounce time span or to disable
  debouncing (by setting it to `0` or `false`).

  Instead of string arrays, you can also use a single string for `watch` and `ignore` and if you
  don't need to specify `ignore` or `debounce`, you can specify the `watch` value directly, e.g.
  ```
  "reloadOnChange": "${workspaceRoot}/lib/*.js"
  ```
* `skipFiles`: An array of glob patterns specifying javascript files that should be skipped while
  debugging: the debugger won't break in or step into these files. This is the same as "black boxing"
  scripts in the Firefox Developer Tools. If the URL of a file can't be mapped to a local file path,
  the URL will be matched against these glob patterns, otherwise the local file path will be matched.
  Examples for glob patterns:
  * `"${workspaceRoot}/skipThis.js"` - will skip the file `skipThis.js` in the root folder of your project
  * `"**/skipThis.js"` - will skip files called `skipThis.js` in any folder
  * `"${workspaceRoot}/node_modules/**"` - will skip all files under `node_modules`
  * `"http?(s):/**"` - will skip files that could not be mapped to local files
  * `"**/google.com/**"` - will skip files containing `/google.com/` in their url, in particular
    all files from the domain `google.com` (that could not be mapped to local files)
* `pathMappings`: An array of urls and corresponding paths to use for translating the URLs of
  javascript files to local file paths. Use this if the default mapping of URLs to paths is 
  insufficient in your setup. In particular, if you use [webpack](https://webpack.github.io/), you
  may need to use one of the following mappings:
  ```
  { "url": "webpack:///", "path": "${webRoot}" }
  ```
  or
  ```
  { url": "webpack:///./", "path": "${webRoot}" }
  ```
  or
  ```
  { "url": "webpack:///", "path": "" }
  ```
  To figure out the correct mappings for your project, you can use the `PathConversion` logger
  (see the [Diagnostic logging](#diagnostic-logging) section below) to see all mappings that are
  being used, how URLs are mapped to paths and which URLs couldn't be mapped.
  If you specify more than one mapping, the first mappings in the list will take precedence over 
  subsequent ones and all of them will take precedence over the default mappings.
* `profileDir`, `profile`: You can specify a Firefox profile directory or the name of a profile
  created with the Firefox profile manager. The extension will create a copy of this profile in the
  system's temporary directory and modify the settings in this copy to allow remote debugging.
* `keepProfileChanges`: Use the specified profile directly instead of creating a temporary copy.
  Since this profile will be permanently modified for debugging, you should only use this option
  with a dedicated debugging profile.
* `port`: Firefox uses port 6000 for the debugger protocol by default. If you want to use a different
  port, you can set it with this property.
* `firefoxExecutable`: The absolute path to the Firefox executable (`launch` configuration only).
  If not specified, this extension will use the default Firefox installation path. It will look for
  both regular and developer editions of Firefox; if both are available, it will use the developer
  edition.
* `firefoxArgs`: An array of additional arguments used when launching Firefox (`launch` configuration only)
* `host`: If you want to debug with Firefox running on different machine, you can specify the 
  device's address using this property (`attach` configuration only).
* `log`: Configures diagnostic logging for this extension. This may be useful for troubleshooting
  (see below for examples).
* `showConsoleCallLocation`: Set this option to `true` to append the source location of `console`
  calls to their output
* `preferences`: Set additional Firefox preferences in the debugging profile
* `installAddonInProfile`: Install the add-on by building an xpi file and placing it in the temporary
  profile that is created for the debugging session. This installation method is incompatible with
  the `reAttach` option and won't allow reloading the add-on while debugging, but it is necessary
  for debugging XUL overlays. By default, it is only used if `reAttach` isn't set to `true`.

### Diagnostic logging
The following example for the `log` property will write all log messages to the file `log.txt` in
your workspace:
```
...
    "log": {
        "fileName": "${workspaceRoot}/log.txt",
        "fileLevel": {
            "default": "Debug"
        }
    }
...
```

This example will write all messages about conversions from URLs to paths and all error messages
to the VS Code console:
```
...
    "log": {
        "consoleLevel": {
            "PathConversion": "Debug",
            "default": "Error"
        }
    }
...
```
 
## Troubleshooting
* Breakpoints that should get hit immediately after the javascript file is loaded may not work the
  first time: You will have to click "Reload" in Firefox for the debugger to stop at such a
  breakpoint. This is a weakness of the Firefox debug protocol: VS Code can't tell Firefox about
  breakpoints in a file before the execution of that file starts.
* If your breakpoints remain unverified after launching the debugger (i.e. they appear gray instead
  of red), the conversion between file paths and urls may not work. The messages from the 
  `PathConversion` logger may contain clues how to fix your configuration. Have a look at the 
  "Diagnostic Logging" section for an example how to enable this logger.
* If you think you've found a bug in this adapter please [file a bug report](https://github.com/hbenl/vscode-firefox-debug/issues).
  It may be helpful if you create a log file (as described in the "Diagnostic Logging" section) and
  attach it to the bug report.
