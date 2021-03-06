node-webkit-updater [![NPM version][npm-image]][npm-url]
=======
This is [node-webkit](https://github.com/rogerwang/node-webkit)-updater.

```
npm install node-webkit-updater
```

It gives you low-level API to:

1. Check the manifest for version (from your running "old" app).
2. If the version is different from the running one, download new package to a temp directory.
3. Unpack the package in temp.
4. Run new app from temp and kill the old one (i.e. still all from the running app).
5. The new app (in temp) will copy itself to the original folder, overwriting the old app.
6. The new app will run itself from original folder and exit the process.

You should build this logic by yourself though. As a reference you can use [example](app/index.html).

Covered by tests and works for [linux](http://screencast.com/t/Je2ptbHhP), [windows](http://screencast.com/t/MSTKqVS3) and [mac](http://screencast.com/t/OXyC5xoA).
### How to run the tests
```
git clone git@github.com:edjafarov/updater.git
npm run deps
npm test
```

## Quick Start
```javascript
var gui = require('nw.gui');
var pkg = require('../package.json'); // Insert your app's manifest here
var updater = require('node-webkit-updater');
var upd = new updater(pkg);
var copyPath, execPath;

/* Args passed when new app is launched from temp dir during update */
if(gui.App.argv.length) {
    copyPath = gui.App.argv[0];
    execPath = gui.App.argv[1];
}

if(!copyPath) {
    /* Checks the remote manifest for latest available version and calls the autoupgrading function */
    upd.checkNewVersion(function(error, newVersionExists, manifest) {
        if (!error && newVersionExists) {
            upgradeNow(manifest);
        }
    });

    /* Downloads the new version, unpacks it, replaces the existing files, runs the new version, and exits the old app */
    function upgradeNow(newManifest) {
        var newVersion = upd.download(function(error, filename) {
            if (!error) {
                upd.unpack(filename, function(error, newAppPath) {
                    if (!error) {
                        upd.runInstaller(newAppPath, [upd.getAppPath(), upd.getAppExec()],{});
                        gui.App.quit();
                    }
                }, newManifest);
            }
        }, newManifest);
    }
} else {
    /* Replace old app, Run updated app from original location and close temp instance */
    upd.install(copyPath, newAppInstalled);
    var newAppInstalled = function(err) {
        if(!err) {
            upd.run(execPath, null);
            gui.App.quit();
        }
    }
}
```


## API

As a reference you can use the [example](https://github.com/edjafarov/updater/blob/master/app/index.html).

<a name="new_updater"></a>
###new updater(manifest)
Creates new instance of updater. Manifest could be a `package.json` of project.

Note that compressed apps are assumed to be downloaded in the format produced by [node-webkit-builder](https://github.com/mllrsohn/node-webkit-builder) (or [grunt-node-webkit-builder](https://github.com/mllrsohn/grunt-node-webkit-builder)).

**Params**

- manifest `object` - See the [manifest schema](#manifest-schema) below.  

<a name="updater#checkNewVersion"></a>
###updater.checkNewVersion(cb)
Will check the latest available version of the application by requesting the manifest specified in `manifestUrl`.

The callback will always be called; the second parameter indicates whether or not there's a newer version.
This function assumes you use [Semantic Versioning](http://semver.org) and enforces it; if your local version is `0.2.0` and the remote one is `0.1.23456` then the callback will be called with `false` as the second paramter. If on the off chance you don't use semantic versioning, you could manually download the remote manifest and call `download` if you're happy that the remote version is newer.

**Params**

- cb `function` - Callback arguments: error, newerVersionExists (`Boolean`), remoteManifest  

<a name="updater#download"></a>
###updater.download(cb, newManifest)
Downloads the new app to a template folder

**Params**

- cb `function` - called when download completes. Callback arguments: error, downloaded filepath  
- newManifest `Object` - see [manifest schema](#manifest-schema) below  

**Returns**: `Request` - Request - stream, the stream contains `manifest` property with new manifest and 'content-length' property with the size of package.  
<a name="updater#getAppPath"></a>
###updater.getAppPath()
Returns executed application path

**Returns**: `string`  
<a name="updater#getAppExec"></a>
###updater.getAppExec()
Returns current application executable

**Returns**: `string`  
<a name="updater#unpack"></a>
###updater.unpack(filename, cb, manifest)
Will unpack the `filename` in temporary folder.
For Windows, [unzip](https://www.mkssoftware.com/docs/man1/unzip.1.asp) is used.

**Params**

- filename `string`  
- cb `function` - Callback arguments: error, unpacked directory  
- manifest `object`  

<a name="updater#runInstaller"></a>
###updater.runInstaller(appPath, args, options)
Runs installer

**Params**

- appPath `string`  
- args `array` - Arguments which will be passed when running the new app  
- options `object` - Optional  

**Returns**: `function`  
<a name="updater#install"></a>
###updater.install(copyPath, cb)
Installs the app (copies current application to `copyPath`)

**Params**

- copyPath `string`  
- cb `function` - Callback arguments: error  

<a name="updater#run"></a>
###updater.run(execPath, args, options)
Runs the app from original app executable path.

**Params**

- execPath `string`  
- args `array` - Arguments passed to the app being ran.  
- options `object` - Optional. See `spawn` from nodejs docs.  

---

## Manifest Schema

An example manifest:

```json
{
    "name": "updapp",
    "version": "0.0.2",
    "author": "Eldar Djafarov <djkojb@gmail.com>",
    "manifestUrl": "http://localhost:3000/package.json",
    "packages": {
        "mac": {
           "url": "http://localhost:3000/releases/updapp/mac/updapp.zip"
        },
        "win": {
           "url": "http://localhost:3000/releases/updapp/win/updapp.zip"
        },
        "linux32": {
           "url": "http://localhost:3000/releases/updapp/linux32/updapp.tar.gz"
        }
    }
}
```

The manifest could be a `package.json` of project, but doesn't have to be.

### manifest.name

The name of your app. From time, it is assumed your Mac app is called `<manifest.name>.app`, your Windows executable is `<manifest.name>.exe`, etc.

### manifest.version
[semver](http://semver.org) version of your app.

### manifest.manifestUrl
The URL where your latest manifest is hosted; where node-webkit-updater looks to check if there is a newer version of your app available.

### manifest.packages
An "object" containing an object for each OS your app (at least this version of your app) supports; `mac`, `win`, `linux32`, `linux64`.

### manifest.packages.{mac, win, linux32, linux64}.url
Each package has to contain a `url` property pointing to where the app (for the version & OS in question) can be downloaded.

### manifest.packages.{mac, win, linux32, linux64}.execPath (Optional)
It's assumed your app is stored at the root of your package, use this to override that and specify a path (relative to the root of your package).

This can also be used to override `manifest.name`; e.g. if your `manifest.name` is `helloWorld` (therefore `helloWorld.app` on Mac) but your Windows executable is named `nw.exe`. Then you'd set `execPath` to `nw.exe`

---

## Troubleshooting

If you get an error on Mac about too many files being open, run `ulimit -n 10240`

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md)

[npm-url]: https://npmjs.org/package/node-webkit-updater
[npm-image]: https://badge.fury.io/js/node-webkit-updater.png
