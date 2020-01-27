# magisk-recovery

This repo is the result of my need to run magisk in core only mode to recover from an incompatible module. This repo contains pre-built versions of magisk that run in core only modes.

To use it...

1) Restore your stock `boot.img`, so that you can boot your phone
2) Open magisk manager settings and set the update channel to `https://raw.githubusercontent.com/ammmze/magisk-recovery/master/custom.json`
3) On magisk home screen refresh it and it should show the `20.299` version as being the latest one. This is really Magisk 20.3+ (Built at commit fbe776db0bdfcbf3b1052381ca43382a61175aa5) but patched to boot in Core Only
4) Install this custom magisk as normal and your phone should be able to boot with magisk in core only mode
5) Disable/uninstall any magisk module(s) causing problems
6) Set the update channel back to stable in magisk manager
7) Install the magisk update using the Direct Install method

## Disclaimer

To cover my butt...you're doing these things at your own risk. I'll not be held responsible should something go sideways.

## Building Magisk to run Core Only

This is for those who may have a fudged up magisk module and can no longer boot into their system and are unable to manually disable/remove the module(s) via the methods described [here](https://www.didgeridoohan.com/magisk/MagiskModuleIssues#hn_Disablinguninstalling_modules_manually). 

In my scenario, the new command (`adb wait-for-device shell magisk --remove-modules`) did not work and my phone is running the latest Android operating system and TWRP has not yet been updated to allow me to mount `/data` and disable/remove the module.

I really did not want to factory reset, so the last option before a factory reset was...

> Another option is to build your own custom version of Magisk that has Core Only Mode enabled by default, patch the device boot image with the custom Manager and then flash the patched image to your device.

Great! That seems reasonable. Time to figure out how to do that!

I jump on over to the [Magisk github](https://github.com/topjohnwu/Magisk) and reading through the readme, topjohnwu has clearly described what is needed to build and how to build it. I'm a developer, so I've already got Python 3, Java, and Android SDK setup. Just needed to set the `ANDROID_HOME` and `ANDROID_NDK_HOME` environment variables, and setup the `config.props` file. And then was able to build it. Then just had to figure out what changes to make and how to patch my boot image with my custom zip.

The gist of the process is this:

1) Build a patched version of the magisk zip that is forced to core only mode
2) Flash the stock boot image
3) Change the update channel to a custom one that points to your patched magisk zip
4) Install the custom patched magisk
5) Remove/disable the problem module
6) Upgrade back to the release version of magisk

### Clone the repo

First you'll want to clone the repo and switch to the latest release tag (which currently is `v20.3+`)...

```bash
git clone https://github.com/topjohnwu/Magisk.git
cd Magisk
git checkout v20.3+
```

### Environment variable locations

For me those environment variables look like this...

```bash
export ANDROID_HOME=/Users/trogdor/Android/sdk
export ANDROID_NDK_HOME=/Users/trogdor/Android/sdk/ndk/20.3+.5594570
```

> You'll want to make sure there are no spaces in those paths. Initially I had a space in one of the directory names and at some point the build blew up on me.

### Setup `config.props`

As instructed in the readme, I copied the `config.props.sample` to `config.props` in the root of the repo we cloned. Then you'll need to set the `version`, `versionCode`, `appVersion`, and `appVersionCode`.

The `version` and `versionCode` should be lower than the version of the magisk you ultimately want to install but still relatively close to the version we want to install or Magisk Manager may throw fits about the version of Magisk being too old for the version Magisk Manager we are running (as of now, Magisk Manager requires Magisk 18+). This is to allow for a seamless upgrade to the actual release version when we are complete. If you set these to a higher value that the release version we want to install at the end, then we'd have to uninstall our custom magisk then re-install with the release version.

The `appVersion` and `appVersionCode` are basically irrelevant as we don't really need a custom Magisk Manager build for this. We just need to patch the `boot.img` using a custom `magisk.zip`. 

In the end, here's what my `config.props` looked like:

```properties
# The version name and version code of Magisk
version=20.299
versionCode=19999

# The version name and version code of Magisk Manager
appVersion=7.3.5
appVersionCode=243

outdir=out

# Whether use pretty names for zips, e.g. Magisk-v${version}.zip, Magisk-uninstaller-${date}.zip
# The default output names are magisk-${release/debug/uninstaller}.zip
prettyName=false

```

### Updating the code to load core only

We are going to update the `native/jni/core/bootstages.cpp` to have it run in core only mode. 

> Note: In magisk manager it will probably look like it booted normally (i.e. core mode is not enabled), but none of the modules would have actually been loaded. This is just because the flag that magisk manager uses to detect whether or not we're supposed to boot into core mode is not actually set. Our code changes are simply forcing it to boot in core only mode.

Now back to `bootstages.cpp`, you'll see a block of code that looks like this...

```cpp
    // Core only mode
    if (access(DISABLEFILE, F_OK) == 0)
        core_only();
```

We just want to remove that `if (access(DISABLEFILE, F_OK) == 0)` (which is basically just checking to see if the file that says to disable modules exists) line so that it looks like this:

```cpp
    // Core only mode
    core_only();
```

### Building

Once you have the environment variables set and the `config.props` setup, we're ready to build. Just in the root of the repo we cloned run...

```bash
./build.py all
```

This builds everything (including a new magisk manager). Technically we could probably get what we need with `./build.py binary && ./build.py zip`.

Once the build is successful, you should have a `magisk-debug.zip` in the `out` directory of the repo we cloned.

### Installing your custom build

Before we can install our custom build, we need to get our phone booted with magisk uninstalled. This can be done by flashing your stock `boot.img` back to your phone.

Now that we have a custom build of magisk, we just need to patch our `boot.img` with it. This can be done fairly easily with magisk manager by giving it a custom channel to look for updates.

Essentially the channel url we give to Magisk Manager should be a url to a JSON file that tells what version is the most recent version and where to downloaded it from. Easiest thing to do is to just grab the current `stable.json` from [here](https://github.com/topjohnwu/magisk_files/blob/master/stable.json) and update it to point to our `magisk-debug.zip`.

I copied the `stable.json` to `out/custom.json` (i.e. same directory where `magisk-debug.zip` was created). Then I updated the `magisk.version`, `magisk.versionCode`, and `magisk.link` to 1) match the version info from `config.props` and 2) the url where I plan to host this. Technically, it would probably be a good idea to update the `magisk.md5` to match the `md5sum` of the `magisk-debug.zip`, but I found it wasn't necessary.

Here's what my `custom.json` looks like:

```json
{
  "app": {
    "version": "7.3.5",
    "versionCode": "243",
    "link": "https://github.com/topjohnwu/Magisk/releases/download/manager-v7.3.5/MagiskManager-v7.3.5.apk",
    "note": "https://raw.githubusercontent.com/topjohnwu/Magisk/59fd38bbf810c81076a1f9b16bc5a0071581b9e7/app/src/main/res/raw/changelog.md"
  },
  "uninstaller": {
    "link": "https://github.com/topjohnwu/Magisk/releases/download/v20.3+/Magisk-uninstaller-20191011.zip"
  },
  "magisk": {
    "version": "20.299",
    "versionCode": "19999",
    "link": "http://192.168.0.117:8000/magisk-debug.zip",
    "note": "https://raw.githubusercontent.com/topjohnwu/magisk_files/2d7ddefbe4946806de1875a18247b724f5e7d4a0/notes.md",
    "md5": "088fb6545e7042a4939bda7d21423b8b"
  }
}
```

Now to serve up this `custom.json` and `magisk-debug.zip`, the easiest thing to do is to make sure your phone and computer are on the same network, go to the `out` directory and then run `python3 -m http.server`. At this point, you should have an `http server` running and (assuming no firewalls in your way) you should be able to access your `custom.json` at http://ip-address-to-your-computer:8000/custom.json (in my case this was http://192.168.0.117:8000/custom.json ... but yours will vary depending on your computers IP address on the network). Next you'll want to make sure you can access that from your phone by going to that url in your phone's browser.

Once your files are accessible via URL from your phone, then just open Magisk Manager, go to Settings, go down to the `Update Channel` and select `Custom`. When prompted for the url, put the url to that `custom.json` file. Then go to the Magisk home screen and refresh it, you should see `Magisk is not installed` and the latest version should be the version and version code we put in our `custom.json`. 

Now that Magisk Manager sees our custom build, you can go ahead and patch your boot image and flash it as you would a normal magisk install. Once installed, you should be able to boot up your phone and see magisk is installed and while it may look like the modules are enabled, they were not actually loaded. We are effectively running in core only mode. At this point go ahead and remove/disable the module(s) that are causing problems.

Once the problem module(s) are disabled/removed, set the Channel in Magisk Manager back to Stable and then you should see an update available. Proceed with the update, doing a `Direct Install` and then you should be back to running the release version of Magisk instead of our custom one. You're now ready to re-install your module(s) and hope that they don't break again.

### Thoughts for improvement in Magisk

Maybe someday Magisk could start building a normal version and core-only version that could be used in scenarios like this, so it would just be a matter of flashing stock boot, switching the channel to a core only channel, remove/disable the module, then install stable again.
