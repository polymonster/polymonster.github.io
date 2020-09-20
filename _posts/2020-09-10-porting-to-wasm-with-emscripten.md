---
title: 'Porting a c++ game engine to the web with emscripten'
date: 2020-09-18 00:00:00
---

I embarked on this journey because I wanted to make my c++ game engine [pmtech](https://www.github.com/polymonster/pmtech) runnable in a browser for quick and easily accessible live demos. Overall the process took about 5 weeks of work, totalling 20 days, I spent some weekend days and a few hours in the evenings. Nothing too intense and the process was quite enjoyable, relaxing dare I say!

You can see the live WebAssembly / WebGL demos [here](https://www.polymonster.co.uk/webgl-demos) for yourself, and the first real-world example I wanted a WebGL demo for is available in my [maths](https://github.com/polymonster/maths) library.

### Motivation

The engine has over 40 examples and unit tests covering mostly graphics capabilities and rendering techniques, they are quick and easy to build and are running on Windows, macOS, iOS and Linux. But the native platform requires users to clone the pmtech repository and build the examples, having them inside a web browser would be the ultimate showcase.


### In Steps Emscripten

Emscripten is an LLVM to WebAssembly compiler and it also generates WebGL from OpenGL and boilerplate code so that the end result is a html file you can load in a browser running your c++ code. I had known about emscripten for a while and had seen other projects referencing it, so I was keen to try it out and see if it was as good as it sounds.

Because pmtech was already cross platform, I had an OpenGLES 3.0 compatible rendering backend I was using for Android and iOS, I am using posix threads for a few platforms and had general unix and stdlib code hanging around too. Emscripten supported all of these as well as supporting gnu make files which nicely can be output by premake, which I am using as my build configuration tool.

The first steps to get up and running were very simple, just a case of adding a few bits of code `premake5.lua` to include the various cpp files I needed and some emscripten flags:

``` lua
linkoptions{
    "-s USE_PTHREADS=1",
    "-s FULL_ES3=1",
    "-s MIN_WEBGL_VERSION=2", 
    "-s MAX_WEBGL_VERSION=2"
}
```

### Entry Point 

With all the shared code from other platforms, there is just a single file [os.cpp](https://github.com/polymonster/pmtech/blob/master/core/pen/source/web/os.cpp) to implement for emscripten. This module is a procedural api which handles the program entry, input/os message pumps, render context creation and submission and so on.

The first step as always was just to put a simple print "Hello World"! inside `int main`. This went smoothly and my macros and code to handle printing worked right out of the box, I was also able to easily compile my third-party libs from source (Bullet Physics, ImGui, etc.) with no need for any changes. I then moved onto getting the OpenGL context created via SDL which required only a small amount of code,

I encountered my first problem when Safari could not create a WebGL context, this is due to lack of support for ES3, and for the time being I am relying on some ES3 features so I tried other browsers and found that Chrome, Firefox and Microsoft Edge all worked ok, so I left Safari for the time being and I will be revisiting it at a later date.

I used Chrome primarily for the rest of development because I later encountered issues with pthreads which I will go into more detail on later in this post.

After the initial hook in I wanted to start including more code, this required the os api functions to be implemented; until you start linking emmake allows undefined symbols so as I tried building new example projects I could see which functions were necessary.

I implemented mouse and keyboard events from SDL and window resize handling, as well as a few getter functions to obtain the window size and so forth. Once this was done the samples were all running, or so I thought...

### Simple Examples Running.

The pmtech examples start with minimal code and slowly increase in complexity; the earlier samples simply render a triangle or load a texture so that the functionality can be tested in isolation. As the samples become more complex they increase in both complexity on the GPU and CPU. More render passes are applied and more draw calls are made and they make use of more advanced GPU capabilities.

I noticed some issues with shadow maps, depth texture sampling, multi-sampling and at first chose to ignore these issues because I was suffering from intermittent crashes coming from bad memory access, unaligned reads and a few other things which I could not put my finger on. I dug a little deeper into debugging with emscripten and added the following to `premake5.lua`:

```
configuration "Debug"
    buildoptions { 
        "-g4", 
    }
    linkoptions { 
        "-g4", 
        "--source-map-base http://localhost:8000/web/",
        "-s STACK_OVERFLOW_CHECK=1", 
        "-s SAFE_HEAP=1", 
        "-s DETERMINISTIC=1" 
    }
```

This gave me a nice call stack and decent information to find the root of the problem. It was related to threading and this made me change my approach for the emscripten platform in pmtech.

### Threading

pmtech has a threading model that contains a main thread that is responsible for window handling, input and graphics api calls, and a user thread for game code and logic which can asynchronously submit graphics api calls through a lockless ring buffer. There are also audio and physics threads, which also have an async ring buffer and work in the same way.

First I began using pthreads but I did encounter some issues; while my posix semaphore code did compile I was unable to create a semaphore, 0 was always returned. I remedied this by implementing my own semaphore with an `std::atomic<u32>`, a while loop and a sleep. This got things up and running but threads were causing instability and the use of atomics also created other problems where the samples would not run in certain browsers.

The crashes I was seeing in complex samples with high draw call counts were because the ring buffer for rendering commands was overwriting it's tail, causing the crashes to happen while commands were overwritten whilst in flight. Previously I did assert on detecting this but it had been removed, and while it may sound a bit reckless to allow the ring buffer to overwrite itself, the design of the system is such that the ring buffer is not dynamically re-sizing, locking or performing any checks for performance reasons and you can configure on a project by project basis how big the ring buffer is.

All of the other supported pmtech platforms work fine this way, with plenty of room in the ring buffer and the main thread constantly consuming items to prevent an overwrite happening. The reason it was struggling on emscripten is because the threads are not truly running asynchronously and calls to `usleep` or similar yield a thread to allow others to execute... Even with a large ring buffer I would still run into problems which led me to realise the user thread was building up too many commands until it yielded for these to be dispatched.

I decided I would just make pmtech be able to run single-threaded and see how that would fare, because I also discovered that Safari and Firefox did not work due to issues with atomics and array buffers respectively, so I just decided for maximum compatibility I could implement single-threading quickly and more efficiently than relying on emscriptens pthreads.

Here is the anatomy of a pmtech thread prior to porting to emscripten, it gets called once and then has it's own internal tight loop which is broken out of when we want to shutdown:

```c++
void* user_setup(void* params)
{
    // setup code
	
    //..
	
    for(;;)
    {
        // update loop
    }
	
    // shutdown code
}
```

I wanted to maintain the ability to still have the multi-threaded support and didn't want to change too much code, so via some macros I came up with this:

```c++
#if PEN_SINGLE_THREADED
#define pen_main_loop(function) pen::jobs_create_single_thread_update(function);
#define pen_main_loop_exit()
#define pen_main_loop_continue() return true
typedef bool loop_t;
#else
#define pen_main_loop(function) for(;;) { if(!function()) break; }
#define pen_main_loop_exit() return false;
#define pen_main_loop_continue() return true;
typedef bool loop_t;
#endif

void* user_setup(void* params)
{
    // setup code
	
    pen_main_loop(user_update);
}

loop_t user_update()
{
    // called each frame
	
    if(exit)
    {
        user_shutdown();
        pen_main_loop_exit();
    }	
	
    pen_main_loop_continue();
}

void user_shutdown()
{
    // clean up memory!
}

```

When running in single-threaded mode the update function is registered via `pen_main_loop` using `jobs_create_single_thread_update` . All registered single thread update functions are called from inside the `emscripten_request_animation_frame_loop` once per frame. 

When running in multi-threaded mode the code ends up being almost the same as before but with the loss of the ability to have local stack objects declared inside user_setup for the life of the program. This caveat generated the most work I had to do on the project because I had a number of samples which needed the code refactoring to support this. It was a simple process, all I had to do was move variable declarations into static scope inside an anonymous namespace... it was just a bit of leg work to get it all done.

```c++
namespace
{
    struct vertex
    {
        f32 x, y, z, w;
    };

    u32 s_vertex_buffer = 0;

    void* user_setup(void* params)
    {
        // ..
    	
        s_vertex_buffer = pen::renderer_create_buffer(bcp);
    }
    
    loop_t user_update()
    {
        // ..
		
        pen::renderer_set_vertex_buffer(s_vertex_buffer, 0, stride, 0);
    }
}
```

Switching to single-threaded mode fixed my issues with crashes, and to my delight some of the more complex samples such as stencil_shadows worked straight away, which I had been unable to test previously.


### WebGL Issues

I started to dig into why some of the samples weren't working. I already had OpenGL samples running on different platforms but I did hit some WebGL specific issues and I found WebGL to be a little more pedantic than other OpenGL implementations when it came to sampler parameters.

 - I have not implemented any of the compute shader functionality, I know it is possible in certain versions of Chrome or Edge but for the time being I havent looked into it because I am primarily working on macOS which does not have any browsers which support compute.The samples  basic_compute and global_illumination are incomplete for WebGl as a result.
 - I disabled MSAA render target support due to the lack of `glTexImage2DMultisample`, but there is MSAA on the backbuffer. I know `glTexStorage2DMultisample` can be used but I still need to look further into this. The msaa_resolve is incomplete due to this, and there are a few other samples which will just be missing MSAA for the time being.
 - I  encountered issues with transform feedback, and the macOS OpenGL 3.0 sample is also broken, it is working on Windows and Linux and I need to dig a little deeper into that one. The sample vertex_stream_out is incomplete because of this.
 - The issue that gave me the biggest run-around was actually self inflicted. I was seeing strange behaviour sampling from `Sampler2DShadowArray` within loops and I was also seeing the same undefined behaviour on macOS/OpenGL 4.0. I initially chalked it up to Apple's deprecation of OpenGL and I skirted around the issue for a little while until later I found a bug where `glUniform1i` was not being correctly set to the correct location for shadow samplers. The issue was not present GLSL 400+ platforms which allow for assigning sampler locations within the shader and that is why I did not see this on Windows or Linux.
 - Sampling a depth texture with `GL_LINEAR` did not work on `Sampler2D` and resulted in zero being return from any texture calls in glsl. But linear filtering does work on `Sampler2DShadow` and `GL_COMPARE_REF_TO_TEXTURE`, linear filtering works for both sampler types on OpenGL3.0+ on macOS, Windows and Linux.  Linearly interpolating depth values that are not comparison ops does not technically makse sense, so WebGL flagging this is up is actually correct, it just causes some issues of how to handle this in the most robust manner because it would be easy to write new code on Metal or Direct3D and accidentally use a linear filter with no adverse effects. I created additional sampler objects with `GL_NEAREST` filtering to fall back to, to maintain parity with other rendering API's.
- I has some troubles with loading `GL_TEXTURE_2D_ARRAY` with mip-maps from .dds. This was also present in my other OpenGL implementations but was nice to fix it. The main problem is because of how OpenGL uses `glTexImage3D` for array textures. The fix was to use `glTexSubImage3D` to supply image data in slice/mip/mip.. order instead of mip/slice/slice.

In doing this port, I actually fixed a many small issues in the OpenGL rendering backend of pmtech, which has been left un-used for a while because I switched to Metal, Direct3D and Vulkan... WebGL has given OpenGL a new lease of life and I'm actually quite happy about that. 

### Publishing

With almost all samples now working, I was ready to hit the web. I tried pushing to my GitHub Pages but the .data files generated from emscripten were too large.

Until this point all of the data for all  samples goes into a `/data/` folder and that is around 400mb. GitHub has a limit of 100mb for file sizes. I trimmed down some of the large textures (some beefy 4k HDR ones) and got the data size down to something reasonable, but all of the simple samples had a relatively long load time and I wanted to be as lean as possible. I used the pmtech tests to generate metadata about which data files each sample need by printing access to `fopen`. I simply read this output file and pass to the emscripten link option `--preload-file`.

I created the [landing page](https://www.polymonster.co.uk/webgl-demos) for the demos and a much stripped down `shell.html` with a download progress readout to make the page feel responsive. The thumbnail images on the landing page were generated by running the pmtech examples on macOS with Metal and downsampled using ffmpeg and laid out in an image grid using code-generated html.

### Final Thoughts

I managed to get every pmtech sample running aside from global_illumination, basic_compute (both requie compute shaders), vertex_stream_out (requires transform feedback), the shadow maps sample has the omission of omni directional shadow maps which require texture cube arrays and the two audio focused samples also are not yet running... I will be looking into WebAudio soon. 

Overall I was very impressed that the majority of features worked and the parity with OpenGL and other platforms is amazing. In release mode with optimizations performance is great, not quite as fast as the native implementations but very good and excellent for showcasing work!

I still have some missing features to implement and will be continuing to work supporting this platform for the forseable future but I thought now was good ime to post about while my thoughts are still fresh.

The final demos can be found [here](https://www.polymonster.co.uk/webgl-demos). You can contact me via any of the various social channels linked on this site, if you are interested check out my [GitHub](https://github.com/polymonster) and if you enjoyed the post I will be sharing more insights from the development of pmech and my other repositories.








