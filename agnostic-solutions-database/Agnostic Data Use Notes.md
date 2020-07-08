# Agnostic Data Use Notes

This is a collection of solutions data for use in new Morphic implementations. The goals of this collection are described at https://issues.gpii.net/browse/GPII-4515 and in more detail in the 
document https://docs.google.com/document/d/1gDZXuJldLMR6rhQoo6nkeLK2NegUBV1g/edit# . 

Data for each solution is held in a separate file, with translations between solution settings in an accompanying document named "-translations". Information about generic or common settings is held in generic.yaml.

# Fields in main solutions entry

## contexts

The `contexts` block describes what the required context is for the solution to run. Currently only one type of context
is supported, namely `OS`. The context block is **mandatory**.

**Example Context**:

```snippet
"contexts": {
    "OS": [
        {
            "id": "win32"
        }
    ]
}
```

## settingsHandlers

The `settingsHandlers` block is unique and one of the most important blocks in the solutions registry entry. It consists
of zero or more settingsHandler entries, each keyed by an arbitrary name (that is unique within this solutions
settingsHandlers block). Inside each settingsHandler entry, the properties for that settingsHandler is provided. The
entries in the settingsHandlers block can be referred to from the lifecycle blocks of the solutions registry entry. The
settingsHandlers block is mandatory, but can be empty.

## launchHandlers:

The `launchHandlers` are very similar to the `settingsHandlers` block in both form, functionality and implementation,
but have a different area of responsibility. As the name suggests, rather than being responsible for modifying settings,
they are responsible for the 'launch' state of the application. That is, they are responsible for any actions related to
stopping or starting the application, and detecting whether it is running.

There are two main difference from settingshandlers: (1) internally, launch handlers only have one setting (`running`)
which can be true or false depending on the (desired) state of solution and (2) launch handlers do not get their
settings from the users NP directly, rather they get the value for `running` from the matchmaker. This is because the
decision of which applications to run/stop/update on login depends on what is available on the system and what the
matchmaker decides best works for the user.

On a technical level, launch handlers work exactly as settings handlers, in that they have two methods `get` and `set`
for getting and setting the current run-state of and application, respectively. They ignore all "settings" passed in the
payload, except for the `running` setting, which should be a boolean value. An implementation of a launch handler should
support 3 actions: reading the current run-state of an application (i.e. the `get` call), starting an application (i.e.
when `set` is called with a `true` value) and stopping an application (i.e. when `set` is called with a `false` value).

## Lifecycle Blocks: configure, restore, start, stop, update and isRunning

Lifecycle blocks describe what should happen when the system needs to configure, start, update, etc., an application.
Neither of these blocks are mandatory as the system will infer their content in case they are not specified.

### configure and restore

These blocks describe how to configure and restore a solution, that is:

* `configure`: Configure the solution with the users setting (e.g. on login)
* `restore`: Restore the settings of the system from before the user logged in

Each of these lifecycle blocks allow the same content - which is an array with entries that are either references to
settingsHandlers blocks or customized lifecycle blocks. To reference a settingsHandler block, the keyword
`settings.<blockname>` is used, where `<blockname>` should be replaced with the name of a settingsHandler block. The
meaning of referencing a settingsHandler is telling the system that the users preference set will be applied to that
solution via the referenced settingshandler. Alternative to referencings setting and restoring settings, arbitrary
lifecycle actions are allowed - the syntax for this is an object that contains at least a `type` key for the function to
call and any further key/value pairs that are needed by the type.

If the `configure` and/or `restore` blocks are omitted from a solution entry, they will be inferred as containing
references to all the solutions settingshandlers (if any).

**Example blocks**:

```snippet
"configure": [
    "settings.myconf"
],
"restore": [
    "settings.myconf"
]
```

### start, stop and isRunning

These blocks all have to do with the run-state of a solution. Their meanings are the following:

* `start`: Launch/start the solution
* `stop`: Stop/kill the solution
* `isRunning`: Detect whether the application is currently running

Similar to the configuration related blocks, each of these lifecycle blocks allow the same content - which is an array
with entries that are either references to launcHandler blocks or customized lifecycle blocks. To reference a
launchHandler block, the keyword `launchers.<blockname>` is used, where `<blockname>` should be replaced with the name
of a `launchHandler` block. Internally, when referencing a launchHandler, different things will happen depending on
which lifecycle block the reference is from. A reference from `start` or `stop` will call the launch handlers `.set`
method with a `running` value of `true` or `false`, respectively. This should have the effect of starting or stopping
the process. In case of a reference from `isRunning`, a call will be made to the launch handlers `.set` method.
Alternative to referencing launch handler blocks, arbitrary lifecycle actions are allowed - the syntax for this is an
object that contains at least a `type` key for the function to call.

None of these blocks are mandatory. If one is omitted from the solution registry entry, it will be inferred as
containing references to all launchHandlers specified for that solution (if any).

**Example blocks**:

```snippet
"start": [
    "launchers.myLauncher"
],
"stop": [
    "launchers.myLauncher"
],
"isRunning": [
    "launchers.myLauncher",
    {
        "type": "gpii.runCheckers.myCustomChecker",
        "command": "applicationChecker.exe /n myApplication"
    }
]
```

### update

The `update` block works very similarly to the lifecycle blocks. It describes what should happen when the configuration
needs to be updated (e.g. due to preferences set changes, PSP adjustments, etc).

The format of the `update` block allows for the same entries as the other lifecycle blocks - that is: arbitrary
lifecycle action blocks and references to `settings.<blockname>` and `launchers.<blockname>`. Unlike for the other
lifecycle blocks, the `update` block furthermore allows references to the `start`, `stop` and `configure` blocks. This
is done by putting a string with the name of that block. When the system encounters one of these references, the entries
of that block will be run.

**Example block**:

```snippet
"configure": [
    "settings.myconf"
],
"update": [
    "configure",
    {
        "type": "gpii.launch.exec",
        "command": "my_application --refresh"
    }
]
```

In the above example, the process of updating the application settings would consists of running the contents of the
`configure` block (that is `"settings.myconf"`), followed by a custom lifecycle actions.

If the `update` block is omitted, it will be inferred by the system. What the inferred content will be depends on the
solutions' liveness. If any of the settingsHandlers have a `liveness` value of less than "live", the inferred content
will be `[ "stop", "configure", "start" ]`, i.e. a cycle of stopping, configuring and starting the application. If all
settingsHandlers are "live", that means that it supports settings being updated live and a value of `[ "configure" ]` is
inferred.

### isInstalled:

This directive is used to detect whether a solution is installed. If any of these blocks evaluate to `true` (implicit
**OR**), the application is considered to be installed.

**Example Entry**:

```snippet
"isInstalled": [
    {
        "type": "gpii.reporter.fileExists",
        "fileName": "${{registry}.HKEY_CURRENT_USER\\Software\\Texthelp\\Read&Write10\\InstallPath}\\ReadAndWrite.exe"
    }, // IMPLICIT OR BETWEEN THESE BLOCKS
    {
        "type": "gpii.packageKit.find",
        "name": "orca"
    }
]
```

# Fields in translations entry

* **capabilitiesTransformations**: Transformations from common terms to application specific settings can be defined
  here. These will enable the framework to automatically translate common terms from a user's preference set into
  application settings. Any common terms listed here, will automatically be added to the `capabilities` of the solution.
* **inverseCapabilitiesTransformations**: This block describes transformations from application settings to common
  terms. If this block is present, the transformations specified will be used by the framework to deduce common terms
  based on any application specific settings in the users preference set. If this key is not present, the framework will
  attempt to do the inversion itself, based on the `capabilitiesTransformations`. If this block is present, but empty,
  the system will make no attempt to automatically invert the `capabilitiesTransformations`.
