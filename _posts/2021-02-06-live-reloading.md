---
title: 'Hot reloading c++ for rapid development with the help of cr'
date: 2020-10-04 00:00:00
---

It has been almost a year of living with covid-19 lockdown restrictions and during this period I have been productivly coding at home and having a lot of fun with hot reloadable c++ to aid rapid development. When the first lockdown measures began the first task I undertook was getting this system working, it had been on my todo list for a while but with no where to go and not much else to do I created a workflow which has been enjoyable to use for the past year and has managed to produce some decent output. I'll go through some details of how it works in this blog and show some of the things I've created so far.

## Some Examples s

I posted a [demo video]() on youtube a while ago of it in action which can best illustrate how it can be used to develop and debug algorithms quickly.

I created this series of geometric animations for my instagram and archived the code from these live coding sessions here:

Next I created the platonic solids from scratch in code with no reference:

And currently I am working on some procedural city generation, for which hot relaoding has been really useful in debugging the many edge cases I am encountering:

## Integrating CR

I can't take all the credit for this, the main component here is [cr](), this header only library is easy to integrate and handles the hot reloading for us, all we have to do is build a host executable which is called pmtech_editor in my case, it intialises cr and calls an update each frame:

```c++
#define CR_HOST // required in the host only and before including cr.h
#include "cr/cr.h"

// init cr
cr_plugin ctx;
bool live_lib = cr_plugin_open(ctx, live_lib_path);

// on update
cr_plugin_update(ctx);
```

It's pretty simple to include and start using cr, but we need to build a dynamic library to open. I call this live_lib and it consists of a single function to implement for cr. I am using premake as my project generation tool and use the `SharedLibarary` project kind to create a dynamic library for either mac, windows or linux:

```c++
CR_EXPORT int cr_main(struct cr_plugin *ctx, enum cr_op operation)
{
    live_context* live_ctx = (live_context*)ctx->userdata;
    static live_lib ll;
    
    switch (operation)
    {
        case CR_LOAD:
            return ll.on_load(live_ctx);
        case CR_UNLOAD:
            return ll.on_unload();
        case CR_CLOSE:
            return 0;
        default:
            break;
    }
    
    return ll.on_update(live_ctx->dt);
}
```
The `on_load` function gets called each time we re-compile and reload live_lib and the update gets called each frame. That is all there is to it for a basic program setup, going beyond this starts to introduce some more engine specific details and the requirement for some additional build settings which are different depending on which platform you want to use.

## Engine Specific Details

In the `cr_main` function I pass a live_context object which contains some pointers to systems of the game engine. The main one to focus on here is `ecs_scene` this is a container for a data-oriented entity component system, in the live_lib I can create entities in the `on_load` function and then manipulate and animate them inside the `on_update` function. The cr docs recommend to make as thin as possible host application but pmtech_editor is a faily large application it intialises all the rendering backend and handles window creation so when we reload live_lib those things remain persistent. There is also a core entity component system update which applies transforms, updates bounding volumes and makes draw calls driven by a config driven render system. This allows the live_lib to simply add things to the ecs:

```c++
void on_load(live_context* ctx)
{
    // clear the scene we are passed in ctx
    scene = ctx->scene;
    ecs::clear_scene(scene);

    // add light
    u32 light = get_new_entity(scene);
    scene->names[light] = "front_light";
    scene->id_name[light] = PEN_HASH("front_light");
    scene->lights[light].colour = vec3f::one();
    scene->lights[light].direction = vec3f::one();
    scene->lights[light].type = e_light_type::dir;
    scene->transforms[light].translation = vec3f::zero();
    scene->transforms[light].rotation = quat();
    scene->transforms[light].scale = vec3f::one();
    scene->entities[light] |= e_cmp::light;
    scene->entities[light] |= e_cmp::transform;
}
```

When loading the scene can be cleared and entities added by filling out their components in SoA fashion, entites added here will be automatically rendered and updated by the core updates inside pmtech_editor. In the update function I use typically my debug render api to help debug algorithms in realtime, manipluate entities or add ImGui calls to further tweak or debug issues:

```c++
void on_update(live_context* ctx)
{
    // pushes a debug line to draw
    dbg::add_line(p0, p1);

    // translate entity
    scene->transforms[entity].translation += vec3f(1.0f);

    // imgui ui
    ImGui::Begin();
    // ...
}
```

The symbols for all of pmtechs engine features are inside the pmtech_editor exe, but we also want to use them inside live_lib in order to do this we need to use dynamic symbol lookup which has some slight differences on each platform which I will cover next.

## Multi Platform Support

I enjoy jumping around on different platforms and machines and also ensuring things work accross multiple platforms in as seamless and consistent way possible. For some it may be tedious but for my job I have had to support many different platforms and it has become second nature to me now. I have successfully got cr working with pmtech on Windows, macOS and Linux and have used it for extended periods on each of these platforms I will detail the steps required for each, as mentioned before I use premake to generate my project files so it is a simple as setting the poroject kind to `SharedLibrary` and supplying some linker flags. The live_lib project is setup to have include paths to be the same as the host executable pmtech but it has no libraries linked to it because we will dynamically link at run time.

### macOS

I use macOS as my primary platform because I have a MacBook Pro and can work on it from anywayhere so this was naturally the first platform for me to implement and it doesnt require much to get up and running. I am using xcodebuild / apple LLVM here and all you need to add is `-undefined dynamic_lookup` to the ld flags of the dylib you want to hot reload. In premake it can be supplied in the `linkoptions`. This is all that is required and the dylib will magically be able to call all the engine code.

### Linux

I am using GCC here and it is as simple as macOS but is slightly different. By default the .so file will try and dynamically lookup the symbols when loaded but we need to export the symbols from the executable so they can be dynamically looked up at runtime. This simply requires `-export-dynamic` to be set on the ld flags of the host executable.

### Windows

Windows requires the most work to get working and requires an extra build step, but with the help of some other tools it isn't too much work. Symbol lookups for windows work differently to macOS and Linux, we need to generate a `.lib` file to link when building live_lib. This lib file contains the function stubs so at runtime the dll knows how to import the symbols. This is where it starts to get a little bit more engine specific again, pmtech is made of 2 dynamic libraries pen.lib (low level abstractions) and put.lib (higher level game engine)... the names of these libs are linked into a single executable which provides the entry point and small amount of app configuration code. After compiling these libs we can generate a `.def` file from these libararies which contaion all the engine functions we may want to call and link the `.def` file when building the live_lib.

I used this python script lib2def which parses the contents of a `.lib` and extracts symbols we are interested in and creates a `.def` file with a single CLI call:

```shell
"py -3 libdef.py pen.lib put.lib -o pmtech.def
```

I added this step as a prebuild command to building the live_lib so it will always generate an up-to-date definition file, and finally as linker options to pass to MSVC `/DEF:pmtech.def`:

## Workflow 

For each platform my workflow is slightly different because I also want to debug at the same time as hot reloading, so I am using visual studio, xcode and vscode on different platforms. All of my setups these days are single screen so I split the screen to allow a debugger or text editor and the running application side by side:

On Linux I used vscode to launch and debug the host application and also edit the code of the live_lib, I have a small terminal window with the build command for the live_lib setup as a single CLI invocation with assitence of my build sytem pmbuild.

On macOS I use xcode to debug and launch the host executable 

On windows I use visual studio to launch and debug the host and have a powershell prompt to build the live_lib.

## Final Thoughts

This has been a huge increase in productivity for me and also it's just so easy to jump and start experimenting ideas, it's not all plain sailing... there are sometimes issues where the reloading stops working and requires and app restart. If the host crashes because of bad code in the live_lib then recovering from this means building the live_lib again which doesnt crash and rebuilding the host so can sometimes get a bit fiddly. I found Windows so far to be the most unreliable, it can start failing much sooner than the others, after a few reloads it seems to revert back to an older dll version, I have also seen what looked like the on_update and on_load had been loaded from different dlls. I havent debugged these issues much yet but I will take a look at improving it.

If you want to try out cr or the integration in pmtech, click on the links to visit the repositories on GitHub.




