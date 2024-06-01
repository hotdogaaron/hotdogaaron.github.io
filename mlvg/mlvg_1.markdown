---
layout: default
title: Adventures in mlvg (Part 1)
description: Aaron Trythall's Website
---
## Introduction

mlvg is a cross-platform audio app/plugin framework created by Randy Jones of Madrona Labs. While I can't speak for the project, it appears to me to be a minimal, and somewhat more nimble, alternative to larger frameworks like JUCE.

This is not a tutorial. I am documenting my warts-and-all experience of compiling and running the mlvg demo application on a MacOS environment, with the goal of identifying snags in the process and making improvments in the mlvg README and project settings.

Writing instructions for open-source projects is really hard. As a developer, you've probably compiled and tested your app hundreds or thousands of times before it's time to write up instructions in the README. At that point, there are probably dependencies you've long since forgotten about, fixes you've made in your environment weeks ago, and a whole host of little problems that usually only crop up the very first time somone interacts with a project.

For someone with a lot of technical experience, these problems can be a minor roadblock or an annoying timewaster. But for someone just getting started in music development, these sorts of things can be completely bewildering, and require hours of troubleshooting and learning that takes you well away from working on cool music stuff.

The unfortunate truth is that you'll always run into stuff like this to some degree when working with source-code based projects that require compilation. Hopefully this helps shed some light on the process.

Let's dive in!

## mlvg README Instructions

	to build an app:
	----------------

	First, build and install madronalib as a static library using the instructions in the madronalib project.
    
	If you haven't already, update the submodules to mlvg:
	- git submodule update --init --recursive
    
	Now install the SDL2 framework. You can get a binary release from libsdl.org. On MacOS this will go into /Library/Frameworks. On Windows into the System directories. 
    
	Now use cmake to generate a project for a test application as follows:
	- mkdir build
	- cd build
	- for Mac OS: `cmake -GXcode ..`
	- for Windows: `cmake -G "Visual Studio 14 2015 Win64" ..` (substitute your compiler of choice)

## My Environment Details

Hardware: MacBook Pro 18,1  
OS: macOS 13.6.3  
Xcode: 15.2  

## Installation and Setup

I started by cloning the `mlvg` repository, and then cloned and updated the git submodules per the README. I did not notice the line from the README regarding `madronalib`, but luckily (?) I had previously compiled and installed the `madronalib` project while experimenting with the Soundplane software. Next, I installed SDL2 via Homebrew. Finally, I generated the project build folder using `CMake`.

Input:

	git clone https://github.com/madronalabs/mlvg && cd mlvg
	cd mlvg
	git submodule update --init --recursive
	brew install sdl2
	mkdir build && cd build
	cmake -GXcode ..

Cmake Output:

	-- The C compiler identification is AppleClang 15.0.0.15000100
	-- The CXX compiler identification is AppleClang 15.0.0.15000100
	-- Detecting C compiler ABI info
	-- Detecting C compiler ABI info - done
	-- Check for working C compiler: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang - skipped
	-- Detecting C compile features
	-- Detecting C compile features - done
	-- Detecting CXX compiler ABI info
	-- Detecting CXX compiler ABI info - done
	-- Check for working CXX compiler: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang++ - skipped
	-- Detecting CXX compile features
	-- Detecting CXX compile features - done
	madronalib headers should be in: /usr/local/include/madronalib
	madronalib library should be in: /usr/local/lib/madrona$<$<CONFIG:Debug>:-debug>
	mlvg destination: lib
	SDL2 headers should be in: /opt/homebrew/Cellar/sdl2/2.30.0/include/opt/homebrew/Cellar/sdl2/2.30.0/include/SDL2
	SDL2 libraries should be in:
	-- Configuring done (6.2s)
	-- Generating done (0.0s)
	-- Build files have been written to: /Users/aaron/dev/music/mlvg/build

## Okay, now what?

At this point we have generated an XCode project, but haven't actually built anything. At this point, a sane person may have double clicked on `build/mlvg.xcodeproj` and attempted to build in Xcode. I hate Xcode with a passion, and naively started trying to build on the command line with `xcodebuild`, and was immediately confronted with a gaggle of errors:

Input:

	cd build
	xcodebuild

Output [excerpt]

	In file included from /Users/aaron/dev/music/mlvg/source/widgets/MLPanel.cpp:6:
	In file included from /Users/aaron/dev/music/mlvg/source/widgets/MLPanel.h:8:
	/Users/aaron/dev/music/mlvg/source/common/MLWidget.h:94:9: error: cannot initialize object parameter of type 'ml::PropertyTree' with an expression of type 'ml::Widget'
	        setProperty(tail(msg.address), msg.value);
	        ^~~~~~~~~~~
	/Users/aaron/dev/music/mlvg/source/common/MLWidget.h:123:17: error: cannot initialize object parameter of type 'const ml::PropertyTree' with an expression of type 'ml::Widget'
	    return Path(getTextProperty("param")) == paramName;
	                ^~~~~~~~~~~~~~~
	/Users/aaron/dev/music/mlvg/source/common/MLWidget.h:136:8: error: cannot initialize object parameter of type 'const ml::PropertyTree' with an expression of type 'ml::Widget'
	    if(hasProperty("param"))
	       ^~~~~~~~~~~
	/Users/aaron/dev/music/mlvg/source/common/MLWidget.h:140:23: error: cannot initialize object parameter of type 'const ml::PropertyTree' with an expression of type 'ml::Widget'
	      Path paramName (getTextProperty("param"));                     ^~~~~~~~~~~~~~~
	[...]

Okay, so what happened here? Taking a look at one of the random errors (`error: cannot initialize object parameter of type 'ml::PropertyTree' with an expression of type 'ml::Widget'`), it seems that an object is being constructed using incompatible input parameters. This sort of error often arises when an installed version of a dependancy does not match the version the project was coded against.

Since the source of the error is the object type `ml::PropertyTree`, let's see where that is defined. First we'll search for PropertyTree in the mlvg codebase:

	$ grep ../source -rne PropertyTree

	# Output
	../source/common/MLAppView.h:104:  PropertyTree _drawingProperties;
	../source/common/MLAppController.h:10:#include "MLPropertyTree.h"
	../source/common/MLWidget.h:17:#include "MLPropertyTree.h"
	../source/common/MLWidget.h:31:// the Widget. To store them, Widget inherits from PropertyTree.
	../source/common/MLWidget.h:33:class Widget : public PropertyTree, public MessageReceiver
	../source/common/MLWidget.h:37:  Widget(WithValues p) : PropertyTree(p) {}
	../source/common/MLDrawContext.h:103:  PropertyTree* pProperties;

Okay, so it looks like it isn't actually defined within `mlvg`, which makes sense. But we can see it appears to be defined in `MLPropertyTree.h`  Let's see if that is from `madronalib`:

	find ../../madronalib -name MLPropertyTree.h

	# Output
	../../madronalib/source/app/MLPropertyTree.h

Jackpot! Remember when I said that I "luckily" had a previously compiled version of `madronalib`? Well, let's go ahead and update / rebuild that. 

[Note: I didn't really search these projects using `grep / find` -- `madronalb`/`mlvg` are fairly small and easy to parse visually or by poking around. That said, in a much larger codebase, using search tools will almost certainly save you a significant amount of time.]

	cd ../../madronalib
	git pull
	mkdir build && cd build
	cmake ..
	make
	sudo make install
	# make install is not included in the madronalib README instructions, should potentially be added

Okay, now that `madronalib` is updated, let's see what new horrors are in store.

## New Horrors

Input:

	cd ../../mlvg/build
	xcodebuild

Output:

	The following build commands failed:
	  Ld /Users/aaron/dev/music/mlvg/build/build/testapp.build/Debug/Objects-normal/x86_64/Binary/testapp normal x86_64 (in target 'testapp' from project 'mlvg')
	(1 failure)

This error message isn't super descriptive, but I noticed that we're building the `x86_64` target. Since I'm running on an `arm64` machine, let's do a quick sanity check.

First, we'll check to make sure the project is targeting `arm64` (If I had thought about this for more than a second, I'd probabaly just assume that it does, since Apple processors have been `arm64` based for many years at this point). Let's take a look at `CMakeLists.txt`

	 IF(APPLE)
	  SET(CMAKE_OSX_ARCHITECTURES "x86_64;arm64" CACHE STRING "Build architectures for Mac OS X" FORCE)
	  SET(CMAKE_OSX_DEPLOYMENT_TARGET "10.15" CACHE STRING "Minimum OS X deployment version")
	 ENDIF(APPLE)

Okay, so on cursory inspection it looks like we're building for both Intel and Silicon architectures, most likely in a Fat/Universal binary.

Let's look further back in the output log and see if there's more detail:

	Ld /Users/aaron/dev/music/mlvg/build/build/testapp.build/Debug/Objects-normal/x86_64/Binary/testapp normal x86_64 (in target 'testapp' from project 'mlvg')
	[...]
	clang: error: no such file or directory: '/usr/local/lib/libmadrona-debug.a'

It looks like since we're building in Debug configuration for the testapp, we're expected to use a debug configuration of `madronalib`. Let's go back and rebuild in Debug livery. Note: I had to consult with the CMake documentation to refresh my memory on how to change build configurations.

	cd ../../madronalib
	rm -rf build && mkdir build && cd build
	cmake -DCMAKE_BUILD_TYPE=Debug ..
	make
	sudo make install

Now after rebuilding mlvg again, we see:

	Ld /Users/aaron/dev/music/mlvg/build/build/testapp.build/Debug/Objects-normal/x86_64/Binary/testapp normal x86_64 (in target 'testapp' from project 'mlvg')
	[...]
	ld: framework 'SDL2' not found
	clang: error: linker command failed with exit code 1 (use -v to see invocation)

So we're having trouble linking with SDL2. My first thought was -- well maybe I was being too fancy by installing with Homebrew -- let's SDL2 install per the README instructions in /Library/Frameworks/. So I went to libsdl.org, downloaded the MacOS release ( `SDL2-2.30.0.dmg` ), and extracted SDL2.framework to `/Library/Frameworks/SDL2.framework`. Next, we'll regenerate the build folder so `cmake` can pick up the new paths.

	cd ..
	rm -rf build && mkdir build && cd build
	cmake -GXcode ..
	xcodebuild

And the result is ... success! We have built the project and test app. Let's try running it.

## Failure to Launch

Looking at our build directory in Finder, we find `testapp` in the `Debug` directory. When I try running it by double clicking in Finder, a terminal window opens up that displays the following: 

	/Users/aaron/dev/music/mlvg/build/Debug/testapp ; exit;
	dyld[28057]: missing symbol called
	zsh: abort      /Users/aaron/dev/music/mlvg/build/Debug/testapp

	Saving session...
	...copying shared history...
	...saving history...truncating history files...
	...completed.

	[Process completed]

Very enlightening. Let's try running it from within Xcode. We'll select the `testapp` target and run it with the "Play" button.

[Screenshot 1]

And the result is ... Build failed. Let's rebuild the whole project from within XCode. We'll select the `ALL_BUILD` target and use the "Play" button again.

[Screenshot 2]

Success! Now we can switch back the the `testapp` target and try again. We have another failure, but with more useable information than from the command line:

[Screenshot 3]

Looking at the XCode console, we can see that the SDL2 Framework is not being found. We didn't have a problem at compilation, so this appears to be a runtime path problem.

First, I checked `testapp` using `otool -L` to see its library dependencies:

	$ otool -L Debug/testapp
	# Output
	Debug/testapp (architecture x86_64):
		[...]
	Debug/testapp (architecture arm64):
		/System/Library/Frameworks/Cocoa.framework/Versions/A/Cocoa (compatibility version 1.0.0, current version 24.0.0)
		/System/Library/Frameworks/Metal.framework/Versions/A/Metal (compatibility version 1.0.0, current version 341.35.0)
		/System/Library/Frameworks/MetalKit.framework/Versions/A/MetalKit (compatibility version 1.0.0, current version 161.0.0)
		/System/Library/Frameworks/CoreAudio.framework/Versions/A/CoreAudio (compatibility version 1.0.0, current version 1.0.0)
		/System/Library/Frameworks/CoreServices.framework/Versions/A/CoreServices (compatibility version 1.0.0, current version 1226.0.0)
		/System/Library/Frameworks/AppKit.framework/Versions/C/AppKit (compatibility version 45.0.0, current version 2487.30.104)
		@rpath/SDL2.framework/Versions/A/SDL2 (compatibility version 3001.0.0, current version 3001.0.0)
		/usr/lib/libc++.1.dylib (compatibility version 1.0.0, current version 1600.157.0)
		/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1336.61.1)
		/System/Library/Frameworks/CoreFoundation.framework/Versions/A/CoreFoundation (compatibility version 150.0.0, current version 2202.0.0)
		/System/Library/Frameworks/CoreGraphics.framework/Versions/A/CoreGraphics (compatibility version 64.0.0, current version 1774.2.3)
		/System/Library/Frameworks/Foundation.framework/Versions/C/Foundation (compatibility version 300.0.0, current version 2202.0.0)
		/usr/lib/libobjc.A.dylib (compatibility version 1.0.0, current version 228.0.0)

The entry `@rpath/SDL2.framework/Versions/A/SDL2 (compatibility version 3001.0.0, current version 3001.0.0)` indicates that this framework is expected to be found using a runtime path (rpath), rather than a fixed location on disk. There are typically some default paths that are searched at runtime, s well as any search paths that are embedded within the binary itself, which are set at build time. Let's check the binary with `otool -l` (note the lowercase) to see if there are any rpaths baked in:

	otool -l Debug/testapp

This command yields a lot of output, but what we are looking for is an entry labeled `LC_RPATH`. I don't see anything in the output, but let's double check:

	otool -l Debug/testapp | grep RPATH
	# Output
	# nada

Since there is no configured RPATH entry, we have to rely on the default runtime search paths. One would assume that `/Library/Frameworks` would be included in the default search paths, but that is apparently not the case. Next, we'll try dropping it next to the binary -- that's usually a safe place to expect an rpath to resolve.

	sudo cp -r /Library/Frameworks/SDL2.framework Debug

After running again, we get:

    dyld[93742]: Library not loaded: @rpath/SDL2.framework/Versions/A/SDL2
      Referenced from: <2C2D005D-459B-3B18-916C-B6D3D27F76D4> /Users/aaron/dev/music/mlvg/build/Debug/testapp
      Reason: tried: '/Users/aaron/dev/music/mlvg/build/Debug/SDL2.framework/Versions/A/SDL2' (code signature in <BB19FDFC-751D-363E-B90C-6D2E2338924A> '/Users/aaron/dev/music/mlvg/build/Debug/SDL2.framework/Versions/A/SDL2' not valid for use in process: library load disallowed by system policy)

It looks like our old friend GateKeeper. We can go into System Settings -> Privacy and Security to enable this.

[Screenshot 4]

Finally, we'll run `testapp` from Xcode again, and finally have a running test application:

[Screenshot 5]

## What's Next

Well, you can see that getting a test `mlvg` application up and running can be a bumpy process. I plan to propose some additions to the README, and look into the CMake configuration for some options to smooth things over.

Off the top of my head, here are some things I want to look at:

- README Additions
	- Add note about `madronalib` debug configuration
	- Add note about Homebrew installation of SDL2
	- Add instructions for building project
		- GUI build vs `xcodebuild`
		- Document XCode / Hardware version(s) last tested with
	- Add note about Gatekeeper Exception for SDL2
- Project Changes
	- Look into proper options for command line build with `xcodebuild`
	- Investigate adding `madronalib` as a git submodule
	- Investigate statically linking SDL2
	- Investigate adding RPATH for dynamic linking to SDL2

I'll probably follow this up with another entry discussing and documenting these possible changes in more detail.