# Mike's Addressables Training Manual

This doc was originally published Jan 13, 2020 against Addressables 1.16.15

# Table of Contents

* [Welcome](#welcome)
* [Use Cases](#use-cases)
* [Content Builds and Build Artifacts](#content-builds-and-build-artifacts)
* [Content Update Builds](#content-update-builds)
* [Settings](#settings)
* [Questions and Answers](#questions-and-answers)

# Welcome

So! You start looking into Addressables, and...aren't sure what to do next. Maybe you installed the package and thought...what now? Maybe you tried to read the official docs and became confused. Maybe you started asking yourself whether it's you or the system. This happened to me, and I made this manual to help people understand the system and encourage them to adopt it instead of giving up on it.

## What is This?

The goal of this manual is to get you up to speed more quickly than you could with only the official docs, source code, or experimenting. It's good for getting started or when you get stumped. It's not comprehensive.

My main goal is to encourage other game devs to use Addressables by reducing the friction of learning the system. I know from experience that Addressables has tons of value out of the box, but a lot of it is hidden behind layers of complexity. I wrote a worse version of this system for Hearthstone, and I'd like to avoid doing that again. I'd like to help others avoid doing that as well because doing that doesn't provide much value for your project.

This doc was inspired by [Amazon Web Services In Plain English](https://expeditedsecurity.com/aws-in-plain-english/)

# Use Cases

The official docs [have a page about this](https://docs.unity3d.com/Packages/com.unity.addressables@1.16/manual/index.html#why-use-addressable-assets), but it focuses on optimizing asset load times.

Here are some business and productivity use cases.

## 1. Decouple assets from code (in editor and builds)

You can use Addressables purely for improving the organization of your workflows and for making cleaner builds. Out of the box, Unity has a very tight coupling between a project's code and assets, and this coupling continues (in a different form) in builds. This is a strength of Unity for early-stage projects, but tends to become a hindrance for larger scale projects because most teams fail to realize when these transitions happen, so they fail to create guardrails at project transition points. In my experience, many devs struggle to realize or articulate this phenomenon, and when you hear them say "Unity sucks for large projects" or similar it's usually a symptom of the aforementioned problem.

Addressables can actually be the guardrails you need to help you scale, and you can use it purely for that purpose without the other benefits described below. Addressables can become the contract between content creators and engineering via assigning an ```Address``` to assets and grouping them into AssetGroups. So engineers load assets by Address (or Label), and content creators can organize their assets however they want as long as the Address (or Label) still points to the assets that the code expects to load.

So this use case is primarily meant for scaling up your project's processes. If this is the only use case you care about, this guide won't help you much vs the official docs. The official docs teach you how to use things like ```Address```, ```Label```, ```AssetGroup```, ```AssetReference``` etc. The only "gotcha" is that you need to make sure that in your Unity Preferences (NOT Project Settings) you've flagged the Addressables system to ```Build Addressables on Player Build```.

## 2. Deploy OTA content to reduce app store build size

You can use Addressables to reduce the size of your builds by deploying assets to a server instead of shipping them with your builds. The main use case for this is reducing app sizes for mobile app stores so you can (hopefully) stay underneath app store cellular download limits. This use case is possible because your assets get transformed into asset bundles, which are basically a type of asset that Unity can load at runtime (normally, assets in Unity must be baked into your builds).

## 3. Deploy updated content OTA

Addressables has a workflow specifically intended for deploying content updates to live apps. It's important to note that this is a distinct workflow from the previous use case of reducing build sizes. You can't just use the workflow from the previous use case and overwrite the files you wrote last time to achieve a content update.

# Content Builds and Build Artifacts

In order for the Addressables runtime to work, you have to make a content build. For me, the hardest thing to learn about Addressables were the content build artifacts. What gets generated, where they go, what you're expected to do with them, etc. Additionally, the documentation and the artifacts themselves don't make it obvious what you're supposed to do after a content build.

First, a note about the term "artifact." This is a build engineering term, which is basically a fancy term for "output file." Although the official docs don't use it, I use it because it's an industry standard term. It's also nice for its flexibility - you don't have to list out every file type that a build can produce if you just say "artifacts" since some build systems can output dozens to hundreds of different file types.

## What is a content build?

A content build is the collection of artifacts that let the Addressables runtime load your Addressable assets. Content builds are platform specific since Addressables is, in simple terms, a build-and-load management system for asset bundles, and asset bundles are platform specific.

When testing in the editor, you don't actually need to care about what a content build is. Addressables defaults to loading assets from your project directly, just like other types of Unity assets.

When testing player builds, you need to care because your player will throw exceptions when loading Addressable assets if you don't make a content build first.

**By default, content builds are not triggered automatically. You have to trigger them either via the UI or via an editor script.** 

## Making a content build

* From the UI
  * Open the Addressables Groups window
  * Build > New Build > Default Build Script
* From an editor script
  * Call `Addressables.BuildPlayerContent()`

## Making Addressables work in a player build

1. Make a content build
1. Make a player build
1. [If you have remote content] Deploy Remote artifacts to your Remote Load Path

**Again, content builds are not made automatically.** You can automate it by writing an editor script to build content before building the player. It's not an intuitive default behavior, but the reasoning behind it is that you can run each one of the above steps independently if you know exactly what changes you made. So, you can speed up iteration time on builds by only running the step where something changed.

## What artifacts get generated by a content build?

Here's a table. Note that "System Files" is a term I made up, not an official term:

<table>
  <tr>
    <td>Artifact Type / Deploy Location</td>
    <td>Local</td>
    <td>Remote</td>
  </tr>
  <tr>
    <td>System Files</td>
    <td>
      <li>catalog.json (content catalog)</li>
      <li>settings.json (profile variables, etc)</li>
      <li>link.xml (preserves assembly contents for scripts attached to bundles)</li>
    </td>
    <td>
      <li>catalog_&lt;timestamp&gt;.json (content catalog)</li>
      <li>catalog_&lt;timestamp&gt;.hash (CRC for the content catalog)</li>
      <br>
      Note: these files are only used for content update builds
    </td>
  </tr>
  <tr>
    <td>Asset Bundles</td>
    <td>
      <li>A special AssetGroup called Built In Data</li>
      <li>Any AssetGroups with Build and Load Paths set to local paths, typically LocalBuildPath and LocalLoadPath</li>
    </td>
    <td>
      <li>Any AssetGroups with Build and Load Paths set to remote paths, typically RemoteBuildPath and RemoteLoadPath</li>
    </td>
  </tr>
</table>

### Special case: addressables_content_state.bin

This artifact type doesn't fit neatly into the chart above. It's a special case that is used as the basis for content updates. This is covered more in the Content Update Builds section.

## Where are these artifacts generated?

**Local artifacts**

* `Library/com.unity.addressables/aa`
* [If you make a player build] `Assets/StreamingAssets`
  * Note that files get copied here only when you make a player build AND there are already files in `Library/com.unity.addressables/aa` from a prior content build. See Questions section for more on that.

**Remote artifacts**

Your selected profile's Remote Build Path, which defaults to `<YourProjectPath>/ServerData/`

**addressables_content_state.bin**

* `Assets/AddressableAssetsData/<PlatformName>`
* Also `Library/com.unity.addressables/<PlatformName>` but it's not clear why. The documentation only mentions the `Assets/` location.

## Which artifacts do I need to care about?

**In a nutshell, anything in the Remote section, because you are responsible for deploying them after your content is built.**

Here's the more detailed breakdown:

**Local Artifacts**: I refer to these as prepackaged since they are similar to assets in Resources folders. They don't need any special deployment steps after your player build is done.

**Local System Files**: These are also prepackaged. They are basically configuration that Addressables needs to locate artifacts (local or remote) and remote system files. Additionally, these files also enable support for injecting custom configuration and behavior at startup (follow the breadcrumbs of InitializationObjects and the CacheInitialization class to see an example of that).

**Remote Artifacts**: These are not immediately useful after an Addressables content build. They just get copied to the Remote Build Path after a content build, which is just a path on your hard drive. You are responsible for deploying the files in that path to the Remote Load Path after the content build is complete.

**Remote System Files**: These are what enable the Addressables runtime to load OTA content. These are pretty much just catalog files, and catalog files are what tell the Addressables runtime where remote artifacts are located. When you initialize Addressables at runtime, you need to tell it to update its content catalog with a remote content catalog (which is deployed just like any other remote artifacts), and by doing that, you enable the Addressables runtime to find where the remote bundles are. You can generate a remote content catalog by checking a box in the AddressableAssetSettings inspector.

**Asset Bundles**: For the most part, Unity can only load assets at arbitrary paths if they are packaged into Asset Bundles. That's why Addressables outputs these.

Things to remember:

* You organize your project assets into AssetGroups
* Each AssetGroup is a ScriptableObject with its own settings
* The AssetGroup's *Build and Load Paths* settings are what determine whether the group is Local or Remote
* AssetGroup settings are also what determine whether a group becomes one or many Asset Bundles

# Content Update Builds

Things to remember:

* An **addressables_content_state.bin** is a requirement for making content update builds. They're produced by content builds. This file is not intended to be deployed to users.
* **addressables_content_state.bin** is placed in `Assets/AddressableAssetsData/<PlatformName>/`
* In order for content to be able to be updated, you need to flag its AssetGroup as Can Change Post Release.
* **When you make a content update build, first check the box that says Build Remote Catalog.** This setting can be found on the AddressableAssetSettings inspector.

There are 2 ways to make a content update build:

* From the UI
  * Open the Addressables Groups window
  * Build > Update a Previous Build
  * Pick the appropriate addressables_content_state.bin i.e. `Assets/AddressableAssetsData/<PlatformName>/addressables_content_state.bin`
* From an editor script
  * If you want to show a file picker for addressables_content_state.bin, first call: `ContentUpdateScript.GetContentStateDataPath(true)`
  * **Note**: If you don't want to show a file picker (e.g. a build server), then **don't** call this. You'll need to hard code or pass in a path to addressables_content_state.bin
  * Next, call: `ContentUpdateScript.BuildContentUpdate(AddressableAssetSettingsDefaultObject.Settings, path)`

The output of a content update build is basically the Remote Deploy column in the artifact table above.

You'll want to look at the official docs for the full picture on this topic. This section is more here for summarizing and clarifying the official docs since they have some errors (e.g. referring to steps 4-6 that don't exist) and can be difficult to understand. 

Official docs: https://docs.unity3d.com/Packages/com.unity.addressables@1.16/manual/ContentUpdateWorkflow.html#how-it-works

# Confusing Terminology

[Naming is one of the 2 hardest problems in computer science.](https://skeptics.stackexchange.com/questions/19836/has-phil-karlton-ever-said-there-are-only-two-hard-things-in-computer-science)

Here are some terms that caused me a lot of confusion and what they mean.

**Local**: Whenever you see this term in Addressables it basically means prepackaged files. Another way to think of it is "deployed with the player build." Try not to think of it in terms of client-server when you're reasoning about all of the configuration, just replace the word "Local" with "files that are prepackaged with your player" and you'll have a much easier time. It also helps to realize that content builds produce a blend of Local and Remote artifacts, so don't get stuck thinking that you're making a "Local only" or "Remote only" build. The one exception to this is making "content update" builds, where every file is intended to be deployed Remotely.

**Build Cache**: This is basically your `Library/com.unity.addressables/aa` folder. For some reason the Build Cache **does not include the addressables_content_state.bin** so if you Clean your Build Cache, not everything in `Library/com.unity.addressables` will be deleted.

**Build Script**: This is confusing because as a first time user you don't necessarily care about build scripting. This isn't custom build scripting though! This is fundamental to Addressables. Build Scripts are what Addressables uses to make content builds, content builds produce the artifacts that the Addressables runtime needs. Content builds are composed of 2 things: Asset Bundles which are your assets transformed from AssetGroups into a runtime-friendly format, and System Files, such as the content catalog, which contain config and Asset Bundle paths.

**Play Mode Script**: Probably the most confusing term, made worse by the fact that your Play Mode Script options are the same as your Build Scripts. This is an **editor-only concept** and the term "Play Mode" refers to Unity's Play Mode in the editor. Setting a Play Mode Script basically tells the Addressables runtime to simulate one of the Build Scripts while your editor is in play mode. One of the BuildScripts is called "Simulate Groups" which makes things really confusing! That one basically lets you simulate network delays as if you were in a deployed build even though you're still in the editor.

**AssetGroupSchemas**: These are fields on your AssetGroups (remember, AssetGroups are just ScriptableObjects in your Asset tree) and are the primary mechanism that Build Scripts use to determine how to transform your AssetGroups into Asset Bundles when a content build is made. Part of what the build process does is loop through each AssetGroup and get its GroupSchemas to help determine how the group should be transformed into bundles.

There are 3 pre-provided GroupSchemas out of the box, two of which (```BundledAssetGroupSchema``` and ```ContentUpdateGroupSchema```) are added to AssetGroups by default and will get you pretty much exactly what you need. There's plenty to learn and configure with ```BundledAssetGroupSchema``` so, even though the GroupSchemas system is intended to be user-extensible (via subclassing), practically speaking, you probably won't need to make your own since the pre-provided ones cover a lot of use cases.

It's worth noting that ```BundledAssetGroupSchema``` has a ```Bundle Mode``` which lets you determine whether to put all assets in a group into one bundle vs individual bundles, and I called it out because it's a field you'll likely tweak regularly depending on the group.

**Profiles**: Profiles are intended to be used in a similar way to Configurations in Visual Studio or Schemes in Xcode. They have a custom syntax that you can look up in the official docs https://docs.unity3d.com/Packages/com.unity.addressables@1.16/manual/AddressableAssetsProfiles.html.

Profiles define a set of variables, and these variables are referenced by AssetGroups. **These 2 concepts together are how you specify whether an AssetGroup gets built into Local or Remote artifacts!** For example, if a group's LoadPath uses the RemoteLoadPath profile variable, then it's a Remote bundle. Here's a simplified sequence of how the build system uses these:

*For each AssetGroup Foo:*

1. BundlePath = Foo.LoadPath (**the LoadPath references a Profile var**) + Foo.Name + a hash value
1. Convert the AssetGroup into a bundle or bundles(!) depending on the settings of the AssetGroups' AssetGroupSchemas
1. Write the bundles to disk using BundlePath
1. Write the BundlePath into the content catalog

Honestly, I simplified this a bit. It's a bit more complex because AssetGroups don't reference profile variables directly. They reference AssetGroupSchemas, which reference profile variables.

# Settings

There are A LOT of settings in Addressables, so I'm going to call out the ones you'll really need to care about when getting started.

## AddressableAssetSettings

This is a ScriptableObject that contains settings, *some* of which are version controlled. It's quite confusing that there's no clear way to tell which settings are version controlled and which are not.

* **Profile in Use**: Changes AddressableAssetSettings. This is the profile that gets used when you make a content build either by calling Addressables.BuildPlayerContent() or clicking Build > New Build > Default Build Script in the Addressables Groups window.
  * **Warning: Because this setting changes the ScriptableObject and this object is intended to be source-controlled, you'll need to be very careful to change the setting back to its default value instead of committing it when you're done testing Addressables.**
* **Build Remote Catalog**: Changes AddressableAssetSettings. You need to check this before you make a content update build.
* **Send Profiler Events**: Does not change AddressableAssetSettings. This enables you to profile the system from the Addressables Event Viewer. This actually changes a file in your `Library/com.unity.addressables` folder, it turns out, effectively making this a user-specific setting.

## CacheInitializationSettings

This is another ScriptableObject. It's hard to find.

Assets > Create > Addressables > Initialization > Cache Initialization Settings

This gives you some control of asset bundle caching behavior when your app starts up. It doesn't actually do anything until you drag the file that was created into the ```InitializationObjects``` slot of your ```AddressableAssetSettings```.

I wouldn't start messing with this until you find you have issues with asset bundle cache size in production. I wanted to call this out partially because finding this file is difficult and partially because it follows a very interesting, very extensible (and undocumented?) pattern which is the InitializationObjects.

InitializationObjects allow you to run logic and save state at content build time which you can then consume at runtime initialization. Using this pattern, I implemented my own InitializationObject that allowed me to detect whether the selected profile used for a build had any remote urls and, if not, the runtime would clear the asset bundle cache on initialization.

## AssetGroups

Every AssetGroup's ScriptableObject has settings. The AssetGroupSchemas on the AssetGroups are what determine which settings you see. These settings affect how the group is converted into asset bundles by the build system.

By default, when you create an AssetGroup, it will have the ```BundledAssetGroupSchema``` and ```ContentUpdateGroupSchema``` and here are some of their more important settings:

* **Build and Load Paths section**: This is what determines whether your bundle will be local or remote
* **Advanced Options > Compression**: See the guidance at https://docs.unity3d.com/Packages/com.unity.addressables@1.16/manual/AddressableAssetsGettingStarted.html#building-for-multiple-platforms
* **Bundle Mode**: This is where you'd start to get into download optimization. This basically determines whether your group gets converted into multiple asset bundles or a single asset bundle. This is effectively your toggle for controlling size vs number of downloaded files.
* **Bundle Naming**: Addressables generates REALLY LONG file paths in its artifacts, so you might have to mess with this if the runtime starts failing to load files. Or you can shorten your file paths in your project. This is a pretty well known sharp edge of the system.
* **Content Update Restriction**: This determines whether the group gets written into your addressables_content_state.bin as content that can be updated after you've already shipped your player builds.

## Other files

For the most part, you can ignore everything else.

I'm not sure what AssetGroupTemplates are. DefaultObject seems to be some kind of hack that exists for some of the built-in Addressables ScriptableObjects to link back to the AddressableAssetSettings. There might be more I'm forgetting or not aware of...

# Questions and Answers

These are just questions I posed to myself, role-playing as a person learning the system, so that I could force myself to come up with a satisfying answer.

### **Q: What is the difference between the terms "content" and "assets"?**

A: As far as Addressables is concerned, they're almost synonyms. Usually the word content is used as a direct synonym for asset bundles.

### **Q: What are asset bundles? Why do I need to care about them?**

A: An asset bundle is an asset file that the Unity runtime can load dynamically from disk so that you can use the asset in your game without building it in as part of your player build. Unlike assets in your Resources folder, asset bundles are more flexible because they can be moved, copied, downloaded, etc independent from a player build.

So why do you care to learn what these are? Because the downside of Addressables is that you have to do the moving, copying, downloading, etc of asset bundles yourself for Remote bundles. Addressables solves the problem of building and managing asset bundles as part of its build system. It even solves downloading them at runtime from a server. But, it **doesn't** solve the problem of deploying bundles to a server.

### **Q: Why does Addressables create/use the StreamingAssets folder?**

A: This is a bit weird, and honestly a bit of a hack if you ask me. `StreamingAssets` itself is a confusing name, but it helps to know the etymology. The naming is a Unity thing, not an Addressables thing. This is a special folder that was created in the early days of Unity, originally intended for deploying movie files with Unity player builds. They chose the name "StreamingAssets" because the intention was for this to be the place where you put large media files that the Unity engine would have to stream into memory at runtime.

Why is this relevant to Addressables? Because of a side effect. Unity copies any files in this folder **verbatim** into a similar location in player builds, meaning that you can use the `Application.streamingAssetsPath` property as a consistent base file path in both the editor (e.g. when Addressables writes paths into its catalog) and at runtime (e.g. when the Addressables runtime needs to load Local files). If you see `{UnityEngine.AddressableAssets.Addressables.RuntimePath}` in Profile variables, that's basically an alias for `Application.streamingAssetsPath`.

That's it! That's the only reason Addressables uses this folder! It's kind of annoying that the Addressables BuildScripts don't delete the `StreamingAssets` folder if they are the only thing using it, but oh well.

### **Q: What version control ignore rules do I need?**

A: Here's a Git example, but you should be able to adapt this to any VCS:

* `/Assets/StreamingAssets/aa/`
* `/Assets/StreamingAssets/aa.meta`
* `/Assets/AddressableAssetsData/**/addressables_content_state.bin*`
* `/ServerData`

I highly recommend putting placeholder files in `Assets/StreamingAssets` and in each platform subfolder of `AddressableAssetsData` (e.g. `AddressableAssetsData/OSX`, `AddressableAssetsData/iOS`, etc) otherwise your teammates will get complaints about meta files for empty folders when they startup your Unity project.

### **Q: How do I test in the editor?**

A: This is affected by your Play Mode Script, which defaults to "Use Asset Database (fastest)" so everything is loaded as if `AssetDatabase.LoadAsset()` were being called.

If you want to test a content build in the editor, you'd first make a content build, then choose the Play Mode Script "Use Existing Build (requires built groups)" and remember this is platform specific, so you'd have to do this again if you switch platforms.

### **Q: What are the different Play Mode Scripts and what do they do?**

A: See this:

<table>
  <tr>
    <td>Name</td>
    <td>Script</td>
    <td>Usage</td>
  </tr>
  <tr>
    <td>Use Asset Database (Fastest)</td>
    <td>BuildScriptFast.cs</td>
    <td>Load directly from your project instead of a content build. Addressables runtime load calls will all basically become AssetDatabase.LoadAsset()</td>
  </tr>
  <tr>
    <td>Simulate Groups (advanced)</td>
    <td>BuildScriptVirtual.cs</td>
    <td>Lets you simulate network loading delays when the Addressables runtime loads bundles. Weirdly, the settings for the loading delays are not exposed in a UI anywhere, so I wouldn't consider this is a well-supported feature (yet?).</td>
  </tr>
  <tr>
    <td>Use Existing Build (requires built groups)</td>
    <td>BuildScriptPackedPlayMode.cs</td>
    <td>Simulates what a player build would do, except using the Library folder, which is where local artifacts end up </td>
  </tr>
</table>

### **Q: How do I test Addressables in player builds, especially for different platforms?**

A: This involves setting up Profiles. You have a few options:

* Just use a single Profile that works all the time. Thankfully the Default profile uses some clever syntax to target the current user-selected platform, so this should mostly work out of the box. You'd want to change your RemoteLoadPath to wherever your remote bundles are deployed.
* Create a set of profiles per-platform (iOS, Mac, Windows, etc) using the values of the UnityEditor.BuildTarget enum, e.g. you could have a Profile named iOS with RemoteLoadPath: https://myurl.com/mygame/environment/iOS

### **Q: How do you specify which profile to use when you make a content build?**

A: You can set this in the AddressableAssetSettings `Profile In Use` field.

When testing locally, keep in mind the warning above about changing this setting and the fact that it's source controlled. I believe it was an oversight or error in judgment on the authors' part to have the *selected value* be source controlled and coupled to the default value (seems fine for the default value to be source controlled).

When using Unity Cloud Build, you can set a selected profile name in the Addressables section of a UCB config.

Pro tip: spend some time making the "Default" profile just work for all possible combinations of platforms, build servers, and teammates' workflows. Not only will this avoid the sharp edge of the selected value being source controlled, it will also be less overhead for your team to deal with when testing on multiple platforms. If you're responsible for setting up this system, and you don't do that, it's only a matter of time before your team will let you know about it. If you do manage to make the Default work, you probably won't hear anything about it. Build engineering is a thankless job!

### **Q: How do you specify which groups should be local vs remote? Why should a group be local vs remote?**

A: This is basically a call you have to make on what content you want to ship as your initial app download (local content) vs what content you're OK with downloading (remote content). There are lots of strategies here that go way beyond the scope of this doc.

### **Q: What should I put for the Remote Load Path? How do I deploy my Remote artifacts to a server?**

A: This is a massive topic. This is basically "how do I put files on a server so that my game can download them."

The best advice I can give is to sign up for and use Unity's Cloud Content Delivery (CCD) service. It has nice integrations with Unity Cloud Build so you can really save yourself and your team from some undifferentiated heavy lifting.

If you don't use that, then the next best advice is "use a major cloud provider." Each cloud provider has file storage services that make this easier than running your own web server and enabling file downloads.

* AWS: S3
* Azure: Blob storage
* GCP: Cloud Storage

Cloud providers also have CDN services that you probably want to put in front of their file storage services, for better performance. For AWS CloudFront is typically what people put in front of S3, so typically clients will download from a CloudFront URL, which points to an S3 bucket as its "origin."

If you're feeling very DIY, you can run a local web server and use a public-facing URL by using ngrok.

### **Q: What Addressables artifacts need to be uploaded to CCD?**

A: Handled automagically if you use the CCD integration in UCB.

If you use the CCD CLI, then any files that are in the Remote Build Path for your current profile. Files that end up in the Remote Build path are the remote content catalog (which is actually 2 files: a .hash and a .json file) if you have that enabled and any remote asset bundles. See this https://docs.unity3d.com/Packages/com.unity.addressables@1.16/manual/AddressablesCCD.html

### **Q: When/how should we upload Addressables artifacts to CCD from a UCB build?**

A: See the Enable Cloud Content Delivery service header in https://docs.unity3d.com/Manual/UnityCloudBuildAddressables.html
