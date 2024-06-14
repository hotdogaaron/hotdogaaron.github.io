---
layout: page
title: Adventures in madronalib (Part 1)
---

## Introduction

`madronalib` is a C++ framework for audio/DSP applications developed by Randy Jones of Madrona Labs.

Much like my [previous writeup](./mlvg_1) on `mlvg`, this is a document of my experience getting the `madronalib` VST Plugin example up and running in my environment. This is not a tutorial, but will hopefully provide some insight to others who may be working with the library.

## My Environment Details

Hardware: MacBook Pro 18,1  
OS: macOS 14.5  
Xcode: 15.4  

## madronalib README Instructions

    Madronalib can be built with the default settings as follows:

        mkdir build
        cd build
        cmake ..
        make
        
    This will create a command-line build of all the new code.

## Building madronalib

First we'll build the `madronalib` libraries using the README instructions. Things proceed smoothly, please note that I additionally installed the libraries using `make install`.

    mkdir build && cd build
    cmake ..
    make
    # Install library in default location of /usr/local/lib
    sudo make install

Next up we'll try running one of the example non-VST applications to ensure everything is working as expected:

    bin/SineExample.app
    zsh: permission denied: bin/SineExample.app

That fails because MacOS app bundles can't be run directly from the command line. `.app` bundles are reallly just directories -- so we'll see if we can run the actual Unix executable embedded within:

    bin/SineExample.app/Contents/MacOS/SineExample
    [rtaudio] Found: 2 device(s)
        Device: 0 - Apple Inc.: MacBook Pro Microphone
        Device: 1 - Apple Inc.: MacBook Pro Speakers

    Stream latency = 70 frames
    sample rate: 48000

    Running ... press <enter> to quit (buffer frames = 512).

Success!

## Building the synth plugin

Next up, let's try to build the synth plugin VST example project. The README instructions for synth-plugin:

    to build:
    ---------

    First, build and install madronalib as a static library using the instructions in the madronalib project.

    Download the VST3 SDK from Steinberg.

    *MacOS*

    You'll also need to download the CoreAudio SDK from Apple. It's easiest if you put the CoreAudio SDK in a directory next to the VST SDK---this way the VST SDK should find it automatically. The version of the CoreAudio SDK you want comes in a folder "CoreAudio" and has four subfolders: AudioCodecs, AudioFile, AudioUnits and PublicUtility. A current link: https://developer.apple.com/library/archive/samplecode/CoreAudioUtilityClasses/CoreAudioUtilityClasses.zip

    To make the VST and AU plugins, first create an XCode project for MacOS using cmake:

    - mkdir build
    - cd build
    - cmake -DVST3_SDK_ROOT=(your VST3 SDK location) -DCMAKE_BUILD_TYPE=Debug -GXcode ..

    Cmake will create a project with obvious placeholders (llllCompanyllll, llllPluginNamellll) for the company and plugin names. 

    Then, open the project and build all. Links to VST3 plugins will be made in ~/Library/Audio/Plug-Ins/VST3. The au component will be copied to ~/Library/Audio/Plug-Ins/Components.

## Installing dependencies

First, we must download and extract the VST3 and CoreAudio SDKs. Per the instructions, we can choose an arbitrary path for these dependencies and set this location in the `cmake` command invocation.

I chose `synth-plugin/external`. I chose to download and extract using the command line, but you can of course manually download and extract using a browser and the MacOS archive utility.

    # in madronalib/examples/synth-plugin
    mkdir external && cd external
    # wget installed from Homebrew (https://brew.sh)
    wget https://www.steinberg.net/vst3sdk
    wget https://developer.apple.com/library/archive/samplecode/CoreAudioUtilityClasses/CoreAudioUtilityClasses.zip
    unzip vst3sdk
    unzip CoreAudioUtilityClasses.zip
    # OCD Cleanup
    rm vst3sdk CoreAudioUtilityClasses.zip

This yields:

    VST_SDK
    CoreAudioUtilityClasses

## First build attempt

We'll start by calling `cmake` using the `VST3_SDK_ROOT` definition per the README. I specified this parameter as an absolute path by prepending `$(pwd)` to represent the full path to the current working directory. Some build systems are happy with relative paths, but I've found it is usually best to specify full absolute paths where possible to avoid unexpected behavior.

Also note that we are building in a debug configuration by setting `-DCMAKE_BUILD_TYPE=Debug`. I did not check whether this build configuration requires `madronalib` to be built and installed in debug format as I already have a debug version installed from a previous build process.

    mkdir build && cd build
    cmake -DVST3_SDK_ROOT=$(pwd)/../external/VST_SDK -DCMAKE_BUILD_TYPE=Debug -GXcode ..

Errors:

    CMake Error at VST3_SDK.cmake:14 (include):
    include could not find requested file:

        SMTG_Global
    Call Stack (most recent call first):
    CMakeLists.txt:89 (include)

    CMake Error at VST3_SDK.cmake:15 (include):
        include could not find requested file:

        SMTG_AddVST3Library
    Call Stack (most recent call first):
    CMakeLists.txt:89 (include)
    [...]

We are greeted with a series of about 15 similar errors. As these are all errors stemming from VST3 SDK related includes, it's likely that we are not using the correct path for `VST3_SDK_ROOT`.

Let's take a look at the `VST_SDK` directory:

    ls ../external/VST_SDK
    # Output
    # VST3_Project_Generator	vst3sdk

It looks like Steinberg packages some extra junk with the SDK. We could update `VST3_SDK_ROOT` to point to the `vst3sdk` subdirectory, but since the README indicated that the `CoreAudio` classes are expected to be found next to the VST3 SDK, let's move it up to the `external` directory:

    mv ../external/VST_SDK/vst3sdk ../external/

## Second attempt

After running `cmake` again, we see the following error:

    CMake Error at external/VST_SDK/vst3sdk/cmake/modules/SMTG_AddSMTGLibrary.cmake:224 (message):
    [SMTG] PROJECT_VERSION is not defined for target: "llllpluginnamellll".  To
        fix this, set the VERSION option of the most recent call of the cmake
        project() command e.g.  project(myPlugin VERSION 1.0.0)
    Call Stack (most recent call first):
        external/VST_SDK/vst3sdk/cmake/modules/SMTG_AddSMTGLibrary.cmake:238 (smtg_target_check_project_version)
        external/VST_SDK/vst3sdk/cmake/modules/SMTG_AddVST3Library.cmake:171 (smtg_target_make_plugin_package)
        external/VST_SDK/vst3sdk/cmake/modules/SMTG_AddVST3Library.cmake:247 (smtg_add_vst3plugin_with_pkgname)
        CMakeLists.txt:91 (smtg_add_vst3plugin)

The relevant line here is `PROJECT_VERSION is not defined for target: "llllpluginnamellll"`. First, we'll want to see where `llllpluginnamellll` is defined. Let's start by checking the project's `CMakeLists.txt`

Looking for a `project` declaration, we find:

    #--------------------------------------------------------------------
    # project and version
    #--------------------------------------------------------------------

    set(CMAKE_CONFIGURATION_TYPES "Debug;Release" CACHE STRING "" FORCE)

    project(llllPluginNamellll)

    set(VERSION_MAJOR "0")
    set(VERSION_MINOR "1")
    set(VERSION_PATCH "0")
    set(VERSION "${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}")

The error message noted that we are mising a `PROJECT_VERSION` definition, and suggest this can be defined as part of the `project()` declaration: `project(myPlugin VERSION 1.0.0)`.

We define the project without an explicit `VERSION`, and later set the cmake variable `VERSION` explicitly.

CMake [documentation](https://cmake.org/cmake/help/latest/variable/PROJECT_VERSION.html) defines `PROJECT VERSION` as the "Value given to the `VERSION` option of the most recent call to the `project()` command, if any."

Thus it appears we should change `set VERSION(...)` to `set PROJECT_VERSION(...)`.

## Third attempt

After running `cmake` again, we see:

    CMake Error at external/VST_SDK/vst3sdk/cmake/modules/SMTG_Bundle.cmake:117 (set_target_properties):
        set_target_properties called with incorrect number of arguments.
    Call Stack (most recent call first):
        external/VST_SDK/vst3sdk/cmake/modules/SMTG_Bundle.cmake:168 (smtg_target_set_bundle)
        CMakeLists.txt:104 (smtg_set_bundle)

Let's take a look at our usage in `CMakeLists.txt`:

    # Line 10
    if(SMTG_MAC)
        smtg_set_bundle(${target} INFOPLIST "${CMAKE_CURRENT_LIST_DIR}/mac/Info.plist" PREPROCESS)
        target_link_libraries(${target} PRIVATE "-framework CoreAudio" )
    elseif(SMTG_WIN)
        target_sources(${target} PRIVATE resource/llllpluginnamellll.rc)
    endif()

Let's also take a look at `SMTG_Bundle.cmake`. The file opens up with some usage documentation:

    Configures the macOS bundle
    The usual usage is:
    ```cmake
    smtg_target_set_bundle(yourplugin
        BUNDLE_IDENTIFIER "com.yourcompany.vst3.yourplugin"
        COMPANY_NAME "yourplugin"
    )
    ```

We can see that the calling convetion in the example and our usage is different. That said, looking through the file, it appears that both arguments `INFOPLIST` and `PREPROCESS` are valid options.

Looking more closely, we can see that our cmake file is calling `smtg_set_bundle` rather than `smtg_target_set_bundle`. Looking for that function definition, we find:

    #[=======================================================================[.md:
        Deprecated since 3.7.4
    #]=======================================================================]
    function(smtg_set_bundle target)
        message(DEPRECATION "[SMTG] Use smtg_target_set_bundle instead of smtg_set_bundle")
        smtg_target_set_bundle (${target})
    endfunction()

It appears that synth-plugin is using an older / deprecated version of this function.

I first tried changing `madronalib`'s usage to `smtg_target_set_bundle` without making any additional changes. This resulted in a bunch of new errors related to the function.

As a test, I decided to try downgrading to a version of the VST3 SDK prior to the deprecation.

## Downgrading VST3 SDK

First, I went to Steinberg's website to see if older releases were available as bundled downloads. Nothing was available, so it looks like I'll need to pull an older version from their git repository.

I browsed the VST3 repository on GitHub, and looked through the releases until I found the release prior to 3.7.4. This gave me a git tag name I could reference in my `git clone` command (note that the `branch` option also accepts tag names).

Also note the use of the option `--recurse-submodules`, which is helpful in this case as the VST3 SDK repository includes several other git repositories as submodules. When cloning, these submodules are not automatically pulled down during a standard `clone` invocation.

After cloning to the directory `vst3-v3.7`, we can update our `cmake` invocation to reference this older VST SDK version.

    # from madronalib/examples/synth-plugin/external
    git clone git@github.com:steinbergmedia/vst3sdk --branch v3.7.3_build_20 --recurse-submodules vst3-v3.7
    cd build
    cmake -DVST3_SDK_ROOT=$(pwd)/../external/vst3-v3.7 -DCMAKE_BUILD_TYPE=Debug -GXcode ..

Finally, we get through the full `cmake` generation process without error, and now have an XCode project we can try building.

## The home stretch

It's finally time to build! We'll invoke XCode on the command line using `xcodebuild`:

    xcodebuild -project llllPluginNamellll.xcodeproj

Errors:

    /Users/aaron/dev/madronalib/examples/synth-plugin/build/llllPluginNamellll.xcodeproj: warning: The macOS deployment target 'MACOSX_DEPLOYMENT_TARGET' is set to 10.10, but the range of supported deployment target versions is 10.13 to 14.5.99. (in target 'ZERO_CHECK' from project 'llllPluginNamellll')
    note: Run script build phase 'Generate CMakeFiles/ZERO_CHECK' will be run during every build because the option to run the script phase "Based on dependency analysis" is unchecked. (in target 'ZERO_CHECK' from project 'llllPluginNamellll')
    ** BUILD FAILED **

This looks relatively simple. `synth-plugin` was probably written some time ago, and targets an older version of MacOS than the range currently supported.

From CMakeLists.txt (Line 12)

    IF(APPLE)
        SET(CMAKE_OSX_ARCHITECTURES "x86_64;arm64" CACHE STRING "Build architectures for Mac OS X" FORCE)
        SET(CMAKE_OSX_DEPLOYMENT_TARGET "10.10" CACHE STRING "Minimum OS X deployment version")
    ENDIF(APPLE)

I changed `CMAKE_OSX_DEPLOYMENT_TARGET` to "10.13" per the error message and rebuilt for some new errors.

At first the error appeared to be:

    /Users/aaron/dev/madronalib/examples/synth-plugin/build/llllPluginNamellll.xcodeproj: warning: User-supplied CFBundleIdentifier value 'com.steinberg.vst3.again' in the Info.plist must be the same as the PRODUCT_BUNDLE_IDENTIFIER build setting value ''. (in target 'llllpluginnamellll' from project 'llllPluginNamellll')

I started to look into this for a few minutes, before realizing that this output was just a warning. As is often the case with compiler/toolchain output, the actual error was somewhat buried further up in the output:

    In file included from /Users/aaron/dev/madronalib/examples/synth-plugin/source/pluginController.cpp:8:
    In file included from /Users/aaron/dev/madronalib/examples/synth-plugin/source/pluginController.h:12:
    In file included from /Users/aaron/dev/madronalib/examples/synth-plugin/source/pluginProcessor.h:11:
    In file included from /usr/local/include/madronalib/mldsp.h:7:
    In file included from /usr/local/include/madronalib/MLDSPOps.h:38:
    In file included from /usr/local/include/madronalib/MLDSPMath.h:27:
    /usr/local/include/madronalib/MLDSPMathSSE.h:32:10: error: 'MLPlatform.h' file not found with <angled> include; use "quotes" instead
    #include <MLPlatform.h>
            ^~~~~~~~~~~~~~
            "MLPlatform.h"

This error indicates that the header file `MLPlatform.h` was not found on the header search paths specified by the system and/or the project, as it was included using the angle bracket (`<>`) format.

Headers for a library are typically grouped together in a subdirectory (`madronalib` in this case). When this folder is placed somewhere in a project's include search path, these headers can be included by referencing their subfolder in the `include` statement, i.e. `include <madronalib/MLPlatform.h>`.

In this specific case, the recommendation to use double quotes may be more appropriate, since this is an internal reference between headers and they are located relative to each other on the filesystem. 

As the file with this non-functioning `include` statement is within the installed `madronalib` headers themselves, this will likely need to be corrected within the `madronalib` library by its maintainer(s).

In the meantime, we can work around this by explicitly adding `/usr/local/include/madronalib` to the target's include path in our project .

I added the line `target_include_directories(${target} PUBLIC "/usr/local/include/madronalib")` to `CMakeLists.txt`. This must be added after the target `llllpluginnamellll` has been created within the CMake file.

After a little experimentation, it appears that the function `smtg_add_vst3plugin` actually creates the CMake target, so we'll add it just after the line `smtg_add_vst3plugin(${target} ${llllpluginnamellll_sources})`.

After rebuilding, we see:

    /Users/aaron/dev/madronalib/examples/synth-plugin/source/pluginProcessor.cpp:113:17: error: use of undeclared identifier 'make_unique'; did you mean 'std::make_unique'?
    _synthInput = make_unique< EventsToSignals >(_sampleRate);
                    ^~~~~~~~~~~
                    std::make_unique

This one looks simple, it's just a missing `std::` namespace declaration in front of a `make_unique` call.

I'm not sure how this snuck through -- perhaps it was added without compiling and running the test application, or the toolchain version used during testing inferred the `std::` namespace without complaint.

In any case, we'll edit the usage to `std::make_unique` and recompile.

And the result is ... succes! Now we can move on to testing within a plugin host.

## Don't fear the Reaper

It's finally time to test out the compiled plugin. First we'll copy it to `/Library/Audio/Plug-Ins/VST3` so our DAW can find it.

    sudo cp -r VST3/Debug/llllpluginnamellll.vst3 /Library/Audio/Plug-Ins/VST3/

I tested with Reaper. I added a new track with a virtual instrument, and it looks like our plug-in was picked up on the VST search path -- so far so good!

![Screenshot 1](images/madronalib_reaper_1.png)  

After loading the plugin, we see:

![Screenshot 2](images/madronalib_reaper_2.png)  

Finally, I try to play a few notes using the built-in virtual midi keyboard -- but I'm not hearing anything.

I check my audio settings, and it looks like my audio configuration has changed since I last ran Reaper. After changing the settings to the default OS audio devices, Reaper immediately crashes:

![Screenshot 3](images/madronalib_reaper_3.png)  

Seeing that Reaper went down so hard when audio was turned on, I worry that there's a problem in the audio code or VST3 SDK usage that's causing a segmentation fault.

The head of the crash report contains some useful information:

    Crashed Thread:        6  com.apple.audio.IOThread.client

    Exception Type:        EXC_CRASH (SIGABRT)
    Exception Codes:       0x0000000000000000, 0x0000000000000000

    Termination Reason:    Namespace SIGNAL, Code 6 Abort trap: 6
    Terminating Process:   REAPER [11254]

We can see that the crash was caused by a `SIGABRT` exception (Abort Signal). This is a relief, as abort signals are often (always?) intentionally sent from the source code. We can see that the exception originated in Thread 6, so let's take a look at that further down in the crash report:

    Thread 6 Crashed:: com.apple.audio.IOThread.client
    0   libsystem_kernel.dylib        	       0x182d7ea60 __pthread_kill + 8
    1   libsystem_pthread.dylib       	       0x182db6c20 pthread_kill + 288
    2   libsystem_c.dylib             	       0x182cc3a30 abort + 180
    3   libsystem_c.dylib             	       0x182cc2d20 __assert_rtn + 284
    4   llllpluginnamellll            	       0x121fb9648 Steinberg::Vst::llllpluginnamellll::PluginProcessor::processSignals(Steinberg::Vst::ProcessData&) + 172 (pluginProcessor.cpp:340)
    5   llllpluginnamellll            	       0x121fb8f48 Steinberg::Vst::llllpluginnamellll::PluginProcessor::process(Steinberg::Vst::ProcessData&) + 68 (pluginProcessor.cpp:65)
    6   REAPER                        	       0x100d55624 void VST_HostedPlugin::VST3_Process<double>(double**, double**, int) + 2356
    7   REAPER                        	       0x100ba0a3c VST_HostedPlugin::ProcessSamples(int, double*, int, int, int, double, midi_List*, bool*, double, double, double, bool, bool, int, bool) + 7200
    8   REAPER                        	       0x100bb3230 FxDsp::processFxDsp(int, double*, int, int, int, int, double, midi_List*, double, bool, double, double, double, double, int, bool) + 1560
    9   REAPER                        	       0x100be03f8 FxChain::ProcessChainDsp(FxDsp*, int, FxChain::proc_ctx const*, double*, midi_List*, int, int&, int&) + 508
    10  REAPER                        	       0x100be01d8 FxChain::ProcessChainForContext(FxChain::proc_ctx const&, int, double*, midi_List*) + 796
    11  REAPER                        	       0x100be0ff0 FxChain::ProcessChain(int, double*, int, int, int, int, double, midi_List*, double, bool, int) + 2148
    12  REAPER                        	       0x100922750 MediaTrack::RenderSamples_nocache(double, long long, double*, int, int, double, MediaTrack* const*, int, int*, bool, int, int, bool*, SyncSMP_Context*) + 21944
    13  REAPER                        	       0x100917fac MediaTrack::RenderSamples(double, long long, int, Track_RS_Output*, int, double, MediaTrack* const*, int, midi_List*, int, int, int, int, int, MediaTrack::Track_SendRec*, bool*, SyncSMP_Context*) + 1528
    14  REAPER                        	       0x10091ffbc MediaTrack::RenderSamples_nocache(double, long long, double*, int, int, double, MediaTrack* const*, int, int*, bool, int, int, bool*, SyncSMP_Context*) + 11812
    15  REAPER                        	       0x100995a58 ProcessProject(ReaProject*, int, int) + 1772
    16  REAPER                        	       0x100994a84 audiostream_onsamples(double**, int, double**, int, int, int) + 4504
    17  REAPER                        	       0x100a06548 audioStreamer_CoreAudio::onsamples(AudioBufferList const*, float*) + 420
    18  REAPER                        	       0x100a07448 caInproc(unsigned int, AudioTimeStamp const*, AudioBufferList const*, AudioTimeStamp const*, AudioBufferList*, AudioTimeStamp const*, void*) + 168
    19  CoreAudio                     	       0x185629454 HALC_ProxyIOContext::IOWorkLoop() + 9508
    20  CoreAudio                     	       0x1856267f8 invocation function for block in HALC_ProxyIOContext::HALC_ProxyIOContext(unsigned int, unsigned int) + 108
    21  CoreAudio                     	       0x1857ac5c4 HALC_IOThread::Entry(void*) + 88
    22  libsystem_pthread.dylib       	       0x182db6f94 _pthread_start + 136
    23  libsystem_pthread.dylib       	       0x182db1d34 thread_start + 8

Tracing the stack from the top down, we can see the `abort` instruction on line 2, followed by an `assert_rtn`. This is followed on the next line by a reference to code within our plugin project, specifically line 340 of `pluginProcessor.cpp`. Let's take a look:

    assert(processSetup.symbolicSampleSize == kSample32);

It looks like our plugin is asserting that the sample size should be 32 bits -- but apparently this is not the case. This assertion is failing, and causing the `abort` signal at runtime.

First I searched for the `processSetup` object in the plugin project source files. If it is being defined within our code, perhaps we can change the sample size there. It doesn't look like this object is explicitly created in our code, so it appears that this object is being created by the VST3 code.

I then searched through `pluginProcessor.cpp` for `kSample32`, and found:

    tresult PLUGIN_API PluginProcessor::canProcessSampleSize(int32 symbolicSampleSize)
    {
    if(symbolicSampleSize == kSample32)
        return kResultTrue;
    
    // we support double processing
    if(symbolicSampleSize == kSample64)
        return kResultTrue;
    
    return kResultFalse;
    }

It would appear that our plugin supports both 32-bit and 64-bit sample sizes. Based on this, the strict assertion of only 32-bit sample sizes may be uneccesary. I decided to comment out the assertion on line 340 and recompile as a test. After loading the VST again in Reaper, we finally have a working synthesizer plugin that makes sound!

Finally, as a sanity check I changed the assertion to `assert(processSetup.symbolicSampleSize == kSample32 || processSetup.symbolicSampleSize == kSample64);` and tested again to confirm that only sample sizes of 32 or 64 were being used.

At some point I'd like to determine where the default sample size is actually being set, but at this moment I'm satisfied with the results.

## Summary

I ran into quite a few issues along the way to a working plugin. It looks like the example is a little out of date with the `madronalib` library, which is often the case with little example projects like this.

That said, we ended up with a nice little list of items to change:

- Correct CMake `VERSION` / `PROJECT_VERSION` declaration
- Update use of `smtg_set_bundle` to `smtg_target_set_bundle` to support newer VST3 SDK Versions
- Update OSX Deployment Target
- Update `MLPlatform.h` inclusion in `MLDSPMathSSE.h`
- Correct missing namespace declaration on `make_unique`
- Determine correct sampleSize behavior