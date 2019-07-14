# What it is

A sample project to check [`XcodeArchiveCache`](https://github.com/sweatco/xcode-archive-cache) in action.

# Before you start

Install `XcodeArchiveCache` using `bundle install` command.

Archive builds require signing, so you'll need to specify the team to sign `Test` target. Commit that change locally because we're going to use `git reset --hard` numerous times to test caching.

# Cachefile

`XcodeArchiveCache` has a simple DSL to describe what to put in the cache. That configuration is stored in a file named `Cachefile`. `cat Cachefile` will show you the configuration that our sample project uses:

```
workspace "Test" do
  configuration "release" do
    build_configuration "Release"
    xcodebuild_args "SOME_FLAG='1' -UseModernBuildSystem=NO"
  end

  derived_data_path "build"

  target "Test" do
    cache "Pods_Test.framework"
    cache "libStaticDependency.a"
  end
end
```

First, we need to tell the tool which workspace or project it should operate upon - that's done in either `workspace "<workspace name>"` or `project "<project name>"` part. Inside that main block we describe what we need to cache and the way to build cached products.

`configuration` parts are about the way we invoke `xcodebuild` to build cached products. You can have as many of those as you want, and specify the one to use with `--configuration` flag during `XcodeArchiveCache` invocation.

- `build_configuration` tells `XcodeArchiveCache` which build configuration should be used. By default, Xcode generates `Debug` and `Release`.
- `xcodebuild_args` are passed to `xcodebuild` - note that these are the same flags we passed to `xcodebuild` in "Build it" part.

`derived_data_path` is obviously the path where `xcodebuild` should store it's derived data during dependency builds.
`target` part defines which dependencies should be cached - `Test` is our main app's target, and it links `Pods_Test.framework` and `libStaticDependency.a`. They, and their direct and transitive dependencies are going to be cached.

# Build without cache

Simply run:

```
pod install && time xcodebuild -workspace Test.xcworkspace -configuration Release -destination generic/platform=ios -scheme Test -derivedDataPath build SOME_FLAG=1 -UseModernBuildSystem=NO -archivePath build/test.xcarchive archive | xcpretty
```

# Build using cache

Run:

```
git reset --hard && git clean -fdx && pod install && time xcode-archive-cache inject --configuration=release --storage="$HOME/build_cache"
```

Since it's the first time we run `XcodeArchiveCache`, our cache directory is empty, so `XcodeArchiveCache` is going to build every dependency and put products into cache. Run `git diff` - some targets vanished from project files, and those are the targets that were parts of build graphs for `Pods_Test.framework` and `libStaticDependency.a`. We replaced these targets with cached build products.

Let's check how cache affects app build time:

```
time xcodebuild -workspace Test.xcworkspace -configuration Release -destination generic/platform=ios -scheme Test -derivedDataPath build SOME_FLAG=1 -UseModernBuildSystem=NO -archivePath build/test.xcarchive archive | xcpretty
```

# Rebuild using cache

Run the same two commands once again. This time, cache directory contains some zipped build products, and `XcodeArchiveCache` is going to rely on them.

# Does it really work?

We've built our sample app using the cache, but does it really work? Since the app was archived, it's not going to run in a simulator - archive builds only produce ARM binaries. Still, we can install the app on a real device and check if it actually runs as intended. 

1. We need to create an `ipa`:

```
cd build/test.xcarchive/Products/Applications && mkdir Payload && mv Test.app Payload/Test.app && zip -r Test.ipa Payload && cp Test.ipa ~/Desktop && cd -
```

2. We need to install that `ipa` to a device: go to Xcode - Window - Devices and Simulators, select the device which you want to install the app to in the left pane, press "plus" button at the bottom, below "Installed Apps" table, and select `Test.ipa` that's on your Desktop.

![](https://i.ibb.co/yn0YZgP/Install-On-Device.png)

Finally, we can launch the app. Contents of the `UILabel` on top of the screen come from `StaticDependency`, which in turn takes these strings from its own dependencies - you can see it `viewDidLoad` method of `ViewController`. Tap the "Tap me" button - what does it say?
