# Strider Extension Loader

[![Build Status](https://hosted.stridercd.com/51f050cf04568a0c00000008/strider-cd/strider-extension-loader/badge)](https://hosted.stridercd.com/Strider-CD/strider-extension-loader/)

[Strider](https://github.com/Strider-CD/strider) is an extensible CI
system, written in node. Strider extensions are simply NPM packages
with additional metadata contained in a file named
`strider.json`. This metadata tells Strider which JavaScript source
files should be loaded and initialized.

Hence, to install a new Strider extension, you can just `npm install`
it in your strider repositiory.

This is a small Node.JS library for loading Strider extensions.

## Strider Extensions

### Extension types

- runner: runs the jobs, like strider-docker-runner
- provider: gets the source code for a project, like strider-github or strider-hg
- job: effects the way a job runs, runs tests, sets up the environment, like strider-node or strider-sauce
- basic: does whatever you want

### Webapp vs Worker

There are two environments where plugins are loaded, webapp and worker.

#### Webapp environment

Effects the way the strider webapp works, how it looks, etc. You can
define templates, serve static files, listen to global strider events,
and other great things.

#### Worker environment

This code is loaded for each job that is run, by the process that is
running the job. This may be the same process as the webapp (as when
using `strider-simple-runner`), or it might be somewhere else
entirely. Accordingly, it is recommended that you net depend on
network connections unless absolutely necessary. In many cases, you
can pass a message up to the strider app and handle it in your
`webapp` code.

### Strider.json

To declare your npm package as a strider plugin, include a
`strider.json` in the base directory. Alternatively, you can have a
`strider` section to your `package.json`.

`strider.json` schema:

```javascript
{
  "id": "pluginid", // must be unique.
  "title": "Human Readable",
  "type": "runner | provider | job | basic", // defaults to basic
  "webapp": "filename.js", // loaded in the webapp environment
  "worker": "filename.js", // loaded in the worker environment
  "templates": {
    "tplname": "<div>Hello {{ name }}</div>",
    "tplname": "path/to/tpl.html"
  },
  // other configurations
}
```

Additionally, if there is a `/static/` directory, the files within it
will be accessible at the path `/ext/:pluginid`.

### Runner

Runner plugins do not get loaded in the worker environment.

#### Webapp

```javascript
module.exports = {
  config: {}, // mongoose schema. This will be per-project config
  create: function (emitter, options, callback) {  }
}
```

The runner object is expected to handle the following events:

- `job.new (job)` see strider-runner-core for a description of the `job` data
- `job.cancel (jobid)` if the runner has the specified job, either
  queued or in process, stop it and fire the `job.canceled` event

Runners are only expected to handle a job if `job.project.runner.id`
identifies it as belonging to this runner.

The runner object is expected to emit the following events:

- `browser.update (eventname, data)` this is for proxying internal
  `job.status` events up to the browser
- `job.queued (jobid, time)`
- `job.done (data)`

#### Extra config

- `panel` see the job plugin section

### Provider

Provider plugins that need an ssh keypair are encouraged to use the `privkey` and `pubkey` that are defined for each project. They are attributes on the `project` object.

#### Webapp

```javascript
module.exports = {
  // mongoose schema for project-specific config
  config: {
    /* `url: String` and `display_url: String` are automatically defined. */
  },
  // mongoose schema for user-level config (like a github OAuth token) and/or cache
  userConfig: {},
  // optional; used by services such as github, bitbucket, etc.
  listRepos: function (userConfig, done(err, repos)) {}
}
```

The `repos` that are returned by `listRepos` are a list of objects
which, when activated, will be the provider config for the project. As
such, it is required to have a `url` that is unique, and it should
also define a `display_url` where appropriate. All other config is up
to you.

```javascript
{
   // this has to be unique; it's how we identify whether a project
   // has already been configured.
   url: 'http://example.com/unique/url.git',
   // optional - linked to from the project page
   display_url: 'http://example.com/path/to/repo',
   // everything else is up to you.
}
```

##### Worker

If just a function is exposed, it is assumed to be "fetch".

```javascript
module.exports = {
  // get the source code for a project. This is where the real work gets done.
  //   dest: the path to put things
  //   context: contains runCmd, io (for event passing)
  fetch: function (dest, userConfig, config, job, context, done) {
  },
}
```

##### Extra Config

Use `panel` for provider config.

- `inline_icon` you can also define a `24x24` icon for the
  `display_url` links. If this is not a path, it is assumed to be the
  name of an icon from `FontAwesome` and will be loaded as such.

### Job

#### Webapp

```javascript
{
   config: {}, // mongoose schema, if you need project-specific config
   // Define project-specific routes
   //   all routes created here are namespaced within /:org/:repo/api/:pluginid
   //   req.project is the current project
   //   req.accessLevel is the current user's access level for the project
   //      0 - anonymous, 1 - authed, 2 - admin / collaborator
   //   req.user is the current user
   //   req.pluginConfig() -> get the config for this plugin
   //   req.pluginConfig(config, cb(err)) -> set the config for this plugin
   routes: function (app) {
   },
   // Define global routes
   //   all routes namespaced within /api/:pluginid
   //   req.user is the current user
   globalRoutes: function (app) {
   },
   // Listen for global events
   //   all job-local events that begin with `plugin.` are proxied to
   //   the main strider eventemitter, so you can listen for them here.
   //   Other events include `job.new`, `job.done` and `browser.update`.
   listen: function (emitter) {
   }
}
```

#### Worker

If only a function is exposed, it is assumed to be the `init(config,
job, cb)` function.

Autodetection rules are only used when a project has no plugins
configured.

```javascript
module.exports = {
  // Initialize the plugin for a job
  //   config: the config for this job, made by extending the DB config
  //           with any flat-file config
  //   job:    see strider-runner-core for a description of that object
  //   cb(err, initializedPlugin)
  init: function (config, job, cb) {
    return cb(null, {
      // string or list - to be added to the PATH
      path: path.join(__dirname, 'bin'),
      // any extra env variables. Will be available during all phases
      env: {},
      // Listen for events on the internal job emitter.
      //   Look at strider-runner-core for an
      //   enumeration of the events. Emit plugin.[pluginid].myevent to
      //   communicate things up to the browser or to the webapp.
      listen: function (emitter) {
      },
      // For each phase that you want to deal with, provide either a
      // shell command [string] for a fn(context, done(err, didrun))
      env: 'nvm install ' + (config.version || '0.10'),
      prepare: 'npm install',
      test: function (context, done) {
        checkSomething(context, function (shouldDoThings) {
          if (!shouldDoThings) {
            // Send `false` to indicate that we didn't actually run
            // anything. This is so we can warn users when no plugins
            // actually do anything during a test run, and avoid false
            // positives.
            return done(null, false);
          }
          doThings(function (err) {
            done(err, true);
          });
        });
      },
      cleanup: 'rm -rf node_modules'
    });
  }
  // this is only used if there is _no_ plugin configuration for a
  // project. See gumshoe for documentation on detection rules.
  autodetect: {
    filename: 'package.json',
    exists: true
  }
}
```

#### Extra config

##### Icon

Job plugins can also define an `icon` in the `strider.json` object,
which is the path to a 48x48 image that will be shown on the project
configuration page when a user is enabling plugins.

##### Config Panel

If the plugin requires special configuration, it can also define a `panel` object in `strider.json`, which looks like:

```javascript
"panel": {
  "src": "path/to/file.html",
  "controller": "NameOfCtrl"
}
```

Define the angular controller in `/static/project_config.js`, which
will be loaded.

See [strider-webhooks](https://github.com/Strider-CD/strider-webhooks)
for an example of a custom config panel.

### Basic

This is where you do whatever you want. It will not be listed in the
UI anywhere automatically, so user configuration will require your own
ingenuity. If the need arises, we might expose some kind of config on
the system level to strider administrators, but not at the moment.

#### Worker

You can listen for events, but you shouldn't run any tests or interact
with the source code in any way. For that, write a `job` plugin.

```javascript
module.exports = function (context, job, done) {
}
```

#### Webapp

```javascript
module.exports = function (context, done) {
}
```

<!-- the following hasn't yet been updated -->

### API

The `context` object passed to each build hook function provides the
following public API:

**Functions**

- `forkProc` - Fork a process and run a UNIX command in it,
  automatically harvesting STDIO. **note** You must use `shellWrap` on
  your command before executing it.
- `striderMessage` - Add a log message to the job output.
- `shellWrap` - Wrap a shell command for use by `forkProc`.

**Data**

- `events` - EventEmitter which may be used for inter-extension
  co-ordination. This EventEmitter is reset on each job run.
- `workingDir` - Absolute path to the root of the current code repository.
- `jobData` - JSON object containing the job details.

The `callback` function passed to each build hook accepts a status value. This
status value is modeled on the UNIX exit code status. 0 means the build hook
completed successfully, non-zero means the buildhook failed.

### Webapps

Another type of extension adds endpoints and content to Strider.

This is achieved with a webapp module specified in strider.json which
exposes a function as follows:


```javascript
module.exports = function(ctx, cb) {

  // Add routes to strider:
  ctx.route.get("/foo/bar", function(req, res, next){
    // This is an express3 route
  })

  // you can use authentication middleware and param validation:

  ctx.route.post("foo/bar"
    , ctx.auth.requireUser
    , ctx.middleware.require_params(["url"])
    , function(req, res, next){
      res.send("Hi", req.currentUser.user.email)
    })

  //  you can register 'blocks' to be inserted at
  //  specific points in existing pages. Any element with a class
  // with the 'StriderBlock_' prefix can be specified here:

  ctx.registerBlock('HeaderBrand', function(context, cb){
    // context has a lot of useful stuff on it:

    var email = context.currentUser.user.email

    // You can do some async processing here, but bear in mind
    // you'll be blocking the page load.

    cb(null, "<h1>FooStrider</h1>");
  })


  // [Note] ctx.registerPanel as seen in the sauce webapp is a legacy method
  // that will eventually disappear and should be rewritten as:
  // ctx.registerBlock("ProjectConfigPanel", foo)

  cb(null) // No errors in extension
}
```

#### Templates

Because writing a bunch of `registerBlock` calls for simple pieces of template
overrides is a little tedious, you can also use the following shortcut in your
strider.json:

```javascript
{"templates": {
    "HeaderBrand" : "<h1>An HTML String</h1>"
  , "FooterTOS" : "./path/to/TOS.html"
  }
}
```
These are either inline strings or paths to static HTML. There is no templating
available for these at present.

*Note* If more than one override is specified for a block, then the first one
will be used. At the moment this means that extensions can squash each other.
If you want to simply 'append' to a block, use the `registerBlock` method
and make sure that you prefix the html you return with:
`ctx.content` which will contain either the default html, or the content from
previous extensions.

#### Static Files

If you have additional static files, you can create a `static` directory in
your extension. These files will be available at `/ext/$modulename/...`
