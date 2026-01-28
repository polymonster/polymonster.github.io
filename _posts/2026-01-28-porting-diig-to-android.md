---
title: 'Porting diig from iOS to Android in less than 2 weeks'
date: 2026-01-28 00:00:00
---

I recently decided to port my music app ‘diig’ to Android since I had some requests from friends and other potential Android users. The app was originally designed and built for iOS using all of my own code and no external frameworks. The whole thing took around 2 weeks to port, not full days work but just a few hours here and there. 2 weeks to port an entire app, 99% feature parity with iOS.

I began work while on the train travelling back to visit my parents and decided to ‘raw dog’ some relaxing ‘casual coding’ on the way. This part of the process was about as relaxing as Super Hans’ infamous ‘relaxing bit of crack’ line in Peep Show. The code base for [diig](https://github.com/polymonster/diig) is already multi-platform since the backend uses my game engine [pmtech](https://github.com/polymonster/pmtech). The engine already supported a number of platforms but not Android, however since Android is built on Linux, I already had a lot of functionality I could reuse from my Linux backend. I have a well tested OpenGL/WebGL and GLES rendering backend and FMOD for cross platform audio for the time being. I knew code wise there were only a few gaps that needed filling in.

## Premake

Premake made getting started very easy. I think it is a very underrated and overlooked tool. I don’t know how CMake became the de facto gold standard for project generators. Premake makes project configuration easy in lua scripts, which I find more flexible than CMake. 

The lua config setup allows you to specify project level, compiler and linker settings. With variables you can generically handle multiple platforms and configurations easily. My existing configs already had code paths for Win32, macOS, iOS and Linux with multiple rendering backends like Direct3D11, OpenGL and Metal. So a good portion of setting up Android was just plumbing through another combination of platform and config settings. 

Premake outputs CMake and gradle build files. Any useful setting that is required in gradle files needs to be passed from premake. I had to add some new settings and propagate information set via premake into the gradle or CMake files. This is a necessary step, it makes it a little bit more painful in the setup but it serves you much more over time, because it ensures your projects can be generated the same on other machines.

The lua configs for multi-platform configuration were pretty easy to extend and add Android, although the code does feel a little bit spaghetti-like since platforms and features have been tacked on over a long period of time now. I decided just to add another bunch of stuff onto the Jenga tower and not get sidelined refactoring. I would like to rewrite the scripts and could do a neater job, but it doesn't really add much value to what I am trying to achieve here.


## Android SDK

The real pain of the whole process was the Android platform itself. It’s just quite fiddly to get it into a good state. The build system uses gradle, CMake and ninja. There is the SDK, the NDK (Native Development Kit) for C/C++ and the JDK for Java. All of these dependencies have their own versions and breaking changes happen frequently. 

I thought first I would quickly plumb through a platform path for Android in my premake configs and get straight to fixing up compile errors. This was wishful thinking! I had to spend the best part of a 3-hour train journey fighting with all of the various parts of the Android build system to massage all the dependencies into a place where they worked together. Plus having to modify and remove deprecated functionality since I encountered breaking changes from a necessary SDK update.

I was doing this on a train with mobile hot-spot internet on my phone and ended up with a 50gb+ install of various dependencies. This is one problem that will not go away. It works as of now, but as time goes on this process repeats as versions of the various dependencies update. It’s not just a case of updating an SDK and then fixing the deprecated parts - you have multiple components, and you have to individually manage and then ensure they are compatible. You suffer for a day to fix it all up once every 6 months or at the time where you come back to the project after a while away.

## Android Studio

Android Studio itself is not the nicest IDE or debugger, but it is handy to have a graphical debugger and not just debugging from the command line, so I suffer through the issues. I do find the interface to be very noisy, there are a lot of pop ups and dialogs, squiggles, indents, inline hints, and long error messages. As you are working you can see the intelli-sense update and the elements subtly shift and move in the UI. It infuriates me that the shortcut keys for debug stepping and continue are different to visual studio and vscode, and that all of the shortcuts feel different and non standard, this adds another layer of friction. I could spend time configuring it, but when you want to get stuck in to some work in a short time period you don't want to waste time configuring keyboard shortcuts.

Feedback of error messages in Android Studio also seem to be way more verbose and annoying that any other IDEs, for example when you get a C++ compiler error, the end of the massage has tons of unnecessary verbal spew about  java exceptions and you have to scroll back through the log to find the real errors. This makes things especially more stressful when you get confusing build errors you are un-familiar with. Logcat is also particularly stressful, it outputs so much information you have to sift through to find your own errors buried under a mountain of irrelevant info, if you filter it you worry you are missing some critical extra info.

## Entry Point / Program Structure

For the entry point and core interaction with the SDK, Android uses Java or Kotlin code. Any C++ code needs to be compiled into a shared library. You can use an NDK only approach, but I am familiar with the Java setup having used it in the past, and it does make some things easier since the NDK and the C versions of the API are badly documented and there are more Java examples.

Android requires an Activity which represents the application flow. You implement methods such as `onCreate` or `onPause` and `onResume`. These are invoked by the OS when you start your app. I also implement a wrapper of the `SurfaceView` that handles the creation of an EGL context and OpenGL surface for rendering.

The engine consists of 2 C++ static libraries which are linked to the `diig.so` shared library. Android is a bit more awkward than other platforms because ordinarily you would build an executable that links the 2 other C++ libs. In this case the executable is Java and we load the C++ dynamically at launch.

## Compiler Errors

The next step on a journey to porting is to get around any compiler errors. This part of the process is actually where I start to feel more comfortable. Mostly this is inside C++ files and most of the errors are expected or in my own code, which gives me agency to fix it.

I have had to do a lot of porting for work so this part comes quite naturally. First the project will need tweaking a bit, making sure include paths are set or all the right files have been added to compilation. Then I tend to `ifdef` out problematic code that I don’t currently need and look at it later, to focus on a small subset of the code base. I try to use `Ifdefs` for platform specific functionality sparingly and split things into file-per-platform. Some `ifdefs` go into shared code like OpenGL or the shared posix implementation. But once the project was set up, I didn't have a great deal of legitimate compiler issues because of how much existing code was reused.

## Linker Errors

Linker errors will occur for missing symbols that do not have an implementation for the Android platform. Most of these were to be expected and it is an easy fix. Here I just make a function stub for any of the missing symbols, that is an empty function that just returns a default value if applicable.

There were a few tricky linker issues to solve involving the audio system. FMOD has its own native libs and they need to be copied into a subdirectory of the Android studio project called `jniLibs`. To do this I added a copy step in premake, which copies the files during premake project generation. FMOD also requires some calls to `loadLibs` to load the C++ code and a call to `FMOD_Android_JNI_Init`. This took me a little time to figure out since I had cryptic error messages, but persistence always prevails and I got there in the end.

## Development

Getting to this point took a good few days, but this was the fun part, or the place I wanted to be. With the code compiling and running it was time to slowly, one function at a time implement the missing functionality of the stub functions. I used these small [examples](https://github.com/polymonster/pmtech/tree/master/examples/code) as unit tests to isolate functionality. 

The first step was to get the `empty_project` sample working that just logs something to the console, this required implementing the logging macro since printf does not display in logcat. After that it was a straightforward process to try rendering the `basic_triangle` to make sure OpenGL was working OK. I moved onto the `imgui_example` to make sure I could use the UI. `play_sound` to test FMOD, which is important since this is a music app. Finally `input_example` to hook in the input and touch events. Once these samples were working I had all of the core functionality for diig and the app should “just work”.

In total I ended up adding 871 lines of C++ code for the `os` module, 151 lines of C++ for Android filesystem related code, and 487 lines of Java code for the core activity. This was the bulk of it for the entire backend. Modifications were required in a few places for platform specific quirks in FMOD, OpenGL. There were also 100 or so lines of lua code for premake.

## JNI

To interoperate between Java and C++ code the Java Native Interface is used. I have to use JNI to pass information from the Java side, such as touch and keyboard (OSK) events from Java where they originate, and through to the C++ code the rest of the app code base calls. I also have to interop in both directions. Calling C from Java is quite simple, you just need to use `public static native`. Going the other direction is a little bit more work:

```C++
   void os_clear_clipboard_string()
   {
       auto env = get_jni_env();
       if(env)
       {
           jmethodID method = env->GetMethodID(s_android_context.m_activity_class, "clearClipboardString", "()V");
           env->CallVoidMethod(s_android_context.m_activity_object, method);
       }
   }
```

When calling Java from C++ you have to get the method by name, but also provide the signature for the types. Then there are a host of functions you can call such `CallVoidMethod`, `CallBooleanMethod` and so on for each type. It's pretty simple but also easy to make a mistake and get the signature wrong or call the wrong typed call. It doesn’t take much effort but the “plumbing” adds up, so I try to have a minimal amount of these wrapper functions.

## A Loose End

There is one loose end that is still to clean up even after writing this post. It annoyed me when I encountered it and it is just the way things are. When the app backgrounds on Android, upon return the whole app boots again from the start. I had to do this because upon background and return the EGL Context (OpenGL) is lost and that means all GPU resources need to be recreated. iOS does not have this same behaviour and the OS magically sorts it out for you. I was being lazy and just didn't get round to thinking about a strategy for it yet. I have had to do this kind of thing before for my job. I just really cannot be bothered with this sort of menial work imposing itself on this project which I wanted to be light and fun, it all too soon starts feeling like a job and I suppose depending on how far I want to take it I will have to get round to fixing this, but for the time being I chose to ignore it.

## Google Play Store

The final boss was automation and delivery to the Google Play store. First I had to pay £20 to actually set up an account. The one off fee is certainly better than Apple's yearly £80 developer fee, but my bank blocked the transaction and I had to jump through some additional hoops to make sure I didn't accidentally pay it twice. Then you have to do identity verification where I had to send my driving license, passport photo, a bank statement, and the blood of my first unborn child.

I went about setting up a GitHub action to automate the publishing. This [Action](https://github.com/r0adkll/upload-google-play) is helpful for handling Google Play. I hooked it up, ran it and the upload failed. I persisted until I discovered my account was not verified yet, so I had to wait for a few days for that to happen. After verification I  tried again, still failing to upload. It turns out you need to first push a build manually from the Google Play console, do that and run a build… still fails. At this point the error was regarding the json key not being valid to upload. 

This was the most frustrating part of the entire process, the Google documentation was out of date. The way you enable auth or generate a token for Google Play upload had changed and the documentation had not. The error message was not very clear so I tried many times adding permissions to various accounts. I tried using Copilot, it gaslighted me time and time again. I persisted until I finally found this [article](https://help.radio.co/en/articles/6232140-how-to-get-your-google-play-json-key) that described the new steps necessary to generate a json key. And success, cloud based build and release with the push of a tag!

The automation to Google Play has been rock solid and reliable since using it, more so than the equivalent for iOS. At some point Apple started enforcing that build machines were registered and linked to your developer account to be able to push to TestFlight. This means you can’t use a cloud GitHub action runner and instead I need to use my own machine as a self-hosted runner. This adds extra admin and unnecessary friction to the whole process, whereas Android just looks after itself.

## Onwards

Porting is a game of persistence, it can be a slog at times, but if you keep on persisting, fixing the errors one by one, starting small, building outward you always get there in the end. 

It can be a bit of a rollercoaster, the dopamine hits hard when some existing code “just works” or it goes smoothly because of well planned abstractions that were set in place years ago, the feeling of relief when you manage to work around some obtuse error from dependencies you have never heard of… just to be hit by crushing anxiety at the new error that appears in its place. 

After the initial steps of friction, the setup has been incredibly fun to use and all of the extra effort required for setting up and configuring something to be not just multi-platform but seamlessly multi-platform makes me able to just dip in and out and do little bits of work flexibly. I can work on my PC and target Android with a dual monitor and more desk space. Or on macOS and target Android or iOS on my laptop, on the go or more casually. 

The diig app is available in closed beta for Android or iOS. If you would like to try it out please contact me for an invite.


