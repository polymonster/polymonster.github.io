---
title: 'Hot reloading c++ for rapid development with the help of fungos/cr'
date: 2021-02-06 17:46:00
---

It has been almost a year of living with covid-19 lockdown restrictions and during this period I have been productively coding at home and having a lot of fun with hot reloadable c++ to aid rapid development. When the first lockdown measures began the first task I undertook was getting this system working and integrated into my [game engine](https://github.com/polymonster/pmtech), it had been on my to-do list for a while, but with nowhere to go and not much else to do I created a workflow which has been enjoyable to use for the past year and has managed to produce some decent output. I'll go through some details of how it works in this blog and show some of the things I've created so far.

## Some Results

I posted a [demo video](https://www.youtube.com/watch?v=dSLwP4D8Fd4) on youtube a while ago of it in action which can best illustrate how it can be used to quickly develop and debug algorithms and effects.

I created this series of geometric animations posted on my [instagram](https://www.instagram.com/__shufb/) and archived the code from these live coding sessions [here](https://github.com/polymonster/live-coding):

I created the platonic solids in wireframe and solid from scratch in code with no reference and with mathematically precise results, just iterating and using the hot reloading to work things out:

![instagram](/images/posts/hot-reloading/platonic-wire.png)

And currently I am working on some procedural city generation, for which hot relaoding has been really useful in debugging the many edge cases I am encountering:

![procedural](/images/posts/hot-reloading/proc.png)

## Integrating CR

I can't take all the credit for this, the main component here is [fungos/cr](https://github.com/fungos/cr). This header-only library is easy to integrate and handles the hot reloading for us, all we have to do is build a host executable, which is called pmtech_editor in my case, which intialises cr and calls an update function each frame:

```c++
#define CR_HOST // required in the host only and before including cr.h
#include "cr/cr.h"

// init cr
cr_plugin ctx;
bool live_lib = cr_plugin_open(ctx, live_lib_path);

// on update
cr_plugin_update(ctx);
```

It's pretty simple to include and start using cr, but we need to build a dynamic library to open at runtime. I call this the live lib and it consists of a single function to implement for cr. I am using premake as my project generation tool and use the `SharedLibarary` project kind to create a dynamic library for either macOS, Windows or Linux:

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

The `on_load` function gets called each time we re-compile and reload the live lib and the update gets called each frame; cr will automatically detect when a change has occured to the dynamic library and reload it for us. That is all there is to it for a basic program setup. Going beyond this starts to introduce some more engine specific details and the requirement for some additional build settings which are different depending on which platform/compiler you want to use.

## Engine Specific Details

In the `cr_main` function I pass a `live_context` object which contains some pointers to systems of the game engine that have already been setup and remain persitent for the applications lifetime. The main thing I use is `ecs_scene` this is a container for a data-oriented structure of arrays (SoA) entity component system.  

In the live lib I can create entities in the `on_load` function and then manipulate and animate them inside the `on_update` function. The cr docs recommend to make as thin as possible host application, but pmtech_editor is a fairly large application and it intialises the rendering backend, platform setup and handles window creation so when we reload live_lib those things remain persistent. There is also a core entity component system update which applies hierarhical transforms, updates bounding volumes, and makes draw calls driven by a config driven render system. This allows the live_lib to simply add things to the ecs and get the benefits of the engine remaining running underneath:

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

    // add a primitive
    u32 new_prim = get_new_entity(scene);
    scene->names[new_prim] = primitive_names[p];
    scene->names[new_prim].appendf("%i", new_prim);
    scene->transforms[new_prim].rotation = quat();
    scene->transforms[new_prim].scale = vec3f::one();
    scene->transforms[new_prim].translation = pos[p] + vec3f::unit_y() * 2.0f;
    scene->entities[new_prim] |= e_cmp::transform;
    scene->parents[new_prim] = new_prim;
    instantiate_geometry(primitives[p], scene, new_prim);
    instantiate_material(default_material, scene, new_prim);
    instantiate_model_cbuffer(scene, new_prim);
}
```

When loading, the scene can be cleared and entities re-added by filling out their components in SoA fashion. Entities added here will be automatically rendered and updated by the core updates inside pmtech_editor. In the live lib update function I typically use my debug render api to help debug algorithms in realtime, manipluate entities or add ImGui calls to add UI to further tweak or debug issues:

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

The symbols for all of pmtech's engine features are inside the pmtech_editor exe, but we also want to call the same functions inside live lib. In order to do this we need to use dynamic symbol lookup to look for the symbols inside the host executable when the dynamic library is loaded at runtime. This requies different linker options depeding on your platform and compile, which I will cover next.

## Multi Platform Support

I enjoy jumping around on different platforms and machines, and also ensuring things work accross multiple platforms in as seamless and consistent way possible. For some it may be tedious, but for my job I have had to support many different platforms simultaneously over the years and it has become second nature to me now so I wanted to have cr working on macOS, Windows and Linux. I have successfully got cr working on my target platforms and have used it for extended periods on each of them. 

As mentioned before I use premake to generate my project files, so it is a simple as setting the poroject kind to `SharedLibrary` and supplying some linker flags. CMAKE or other prebuild tools provide the same cross-platform simplicity too or you could still do this through an IDE if you are that way inclined. The live lib project is set up to have include paths to be the same as the host executable, but it has no libraries linked to it because we want to dynamically link at run time. Until changes are made to the linker settings you will find linker errors for undefined symbols for anything you want to call inside the host exe.

### macOS

I use macOS as my primary platform so this was naturally the first platform for me to implement and it doesn't require much to get up and running. I am using xcodebuild / apple LLVM here and all you need to add is `-undefined dynamic_lookup` to the ld flags of the dylib you want to hot reload. In premake it can be supplied in the `linkoptions`. This is all that is required and the dylib will magically be able to call all the engine code.

### Linux

I am using GCC here and it is as simple as macOS but is slightly different. By default the `.so` file will try and dynamically lookup the symbols inside it's host exe when loaded but we need to export the symbols from the executable so they can be visible at runtime. This simply requires `-export-dynamic` to be set on the ld flags of the host executable.

### Windows

Windows requires the most work to get working and requires an extra build step, but with the help of a python script it isn't too much work. Symbol lookups for windows work differently to macOS and Linux, we need to generate a `.lib` file to link when building the live lib. This lib file contains the function stubs so at runtime the dll knows how to import the symbols. This is where it starts to get a little bit more engine specific again, pmtech is made of 2 dynamic libraries pen.lib (low level abstractions) and put.lib (higher level game engine)... these libs are linked into a single executable which provides the entry point and small amount of app configuration code. After compiling these libs we can generate a `.def` file from these libraries which contaion all the engine functions we may want to call and link the `.def` file when building the live lib.

I used this python script [lib2def](https://github.com/tapika/test_lib2def) which parses the contents of a `.lib` and extracts symbols we are interested in and creates a `.def` file with a single CLI call:

```shell
"py -3 libdef.py pen.lib put.lib -o pmtech.def
```

I added this step as a prebuild step of the live lib so it will always generate an up-to-date definition file, and finally as linker options to pass to MSVC `/DEF:pmtech.def` when building the live lib.

Depending on how many libs you link in you exe and which functions you want accessible from the live lib you will need to add them to the list of libs which need symbols exporting and run them through the python script.

## Workflow 

For each platform my workflow is slightly different because I also want to debug at the same time as hot reloading, so I am using visual studio, xcode and vscode on different platforms. All of my setups these days are single screen so I split the screen to allow a debugger or text editor and the running application to sit side by side and give myself a single key stroke to rebuild the live lib:

On Linux I used vscode to launch and debug the host application and also edit the code of the live lib, I have a small terminal window with the build command for the live lib setup as a single CLI invocation with assistance of my build sytem [pmbuild](https://github.com/polymonster/pmbuild):

![linux-wf](/images/posts/hot-reloading/linux-wf.png)

On macOS I use xcode to debug and launch the host executable and also build the live lib:

![mac-wf](/images/posts/hot-reloading/mac-wf.png)

On windows I use visual studio to launch and debug the host and have a powershell prompt to build the live lib because visual studio does not allow you to build another project inside a solution that is currently debugging, xcode wins in this respect:

![win-wf](/images/posts/hot-reloading/win-wf.png)

## Final Thoughts

This has been a huge increase in productivity for me and also it's just so easy to jump in and start experimenting with ideas. 

It is not all not all plain sailing... there are sometimes issues where the reloading stops working and requires an app restart. If the host crashes because of bad code in the live lib then recovering from this means building the live lib again so it doesn't cause a crash and relaunching the host, this process can be a little disruptive while in the middle of debugging or working on something and sometimes you can introduce mutiple crashes which might take multiple failures and attempts to reload. I found Windows so far to be the most unreliable platform, it can start failing much sooner than the others, after a few reloads it can sometimes revert back to linking an older dll version, but I haven't debugged it in great detail yet.

If you want to try out [fungos/cr](https://github.com/fungos/cr) or the integration in [pmtech](https://github.com/polymonster/pmtech), click on the links to visit the repositories on GitHub.




