---
title: 'Not so modern c++'
date: 2021-03-27 17:41:00
---

There have been a lot of changes and additions to the c++ standard since I started using c++ in the early 2000's, the language itself has started to look quite different. Yet I am still using a very stripped down subset of c++ actually more c-style and only reaching for c++ and modern c++ with careful consideration. Why is this? Because I have never really coveted new language features, I have always been more focused on the 'details' like the maths behind graphics effects or physics mechanics and not so much about the 'glue' of how it all fits together. I worked on and shipped games pre c++11 and saw effective techniques that worked well and stuck with them, these techniques provide good performance even in debug builds.

For a while I was quite anti-c++, this was from [optimising CPU code](https://www.polymonster.co.uk/blog/dod-ecs) that was heavily object orientated and very bad in terms of L1/L2 cache miss rate. This lead me to data oriented design and to a rejection of object orientated programming for which c++ took the brunt of it. But just because c++ is an object oriented programming language doesn't mean you have to use classes and objects. In 2014 I started working on a new game engine using c++ but writing c-style code with a c++ complier and only using some select c++ features. My main goal was to make all of my code as small, simple and fast as possible... but still with flexibility and extendability.  

## Still C

People may be encouraged to use `std::array` instead of plain c arrays or dynamic memory, honestly I have never typed the words `std::array` until just now. I have seen insane uses of `std::vector` to store and process image data. In debug [stl is slower than using c arrays](https://blog.demofox.org/2016/09/26/is-code-faster-than-data-switch-statements-vs-arrays/) and while in release stl containers may work out to be zero cost abstractions I still don't like the extra bloat in debug builds, especially in hot code that is processing large sets of data. Having a fast or at least a usable debug build is a great strength and I have seen many times over heavily stl dependent code bases can be extremely slow in debug. I am still using  raw mallocs and c arrays in loads of places, years ago I wouldn't touch stl at all, now I use it sparingly in places it works for me.

### Stripped Down Data Structures

I use these simple, custom [data structures](https://github.com/polymonster/pmtech/blob/master/core/pen/include/data_struct.h) in my engine. But have implemented many others like these and sometimes 'in-place' directly inside another system. They come with a usage warning though they are bespoke for my use cases. For instance the ring buffer might overwrite it's tail and cause a crash if its not large enough and there is no error checking for that. I don't want that cost either, I promise by the time the product ships that the ring buffer does not overwrite itself, other people may want or need stronger guarantees of safety and I understand that. This "fly by the seat of your pants" approach might not be for everyone but you can remove a lot of bloat by living on the edge. I don't like much error handling, simply assert when something goes wrong, fix the assert and continue... the final product will not hit those asserts, I promise.

### Some STL

Nowadays where I start to need more algorithmic functionality I will use stl, I find `std::set` underrated and useful in many situations. In my engine at work we use stl more than I do at home, this is because more people work on the project and other people are more comfortable with that. I tend to stick with that and this led me to enjoy using stl because it's quicker to work with than using my stripped down data structures in many scenarios. For places that need performance I can optimise the stl version by replacing with something more specialised if we need it. And for small data sets I don't really worry too much for the performance these days on the hardware I am targeting.

### Structure of Arrays

This is a common data oriented design technique to improve cache efficiency but there are more layers to it which could become more expensive when using stl or polymorphic programming techniques:

Let say we want an array of `entities` where an `entity` has a transform consisting of `pos, rot, scale` and we want to translate the entities position using a function.

If we go by the object oriented book and use c++ we could do something like this:

```c++
class entity
{
public:
    vec3f pos;
    vec3f scale;
    quat  rot;
}
std::vector<entity> entities;

void translate_aos(vec3f offset)
{
    for(auto& entity : entities)
    {
        entity.pos += offset;
    }
}

```

This would be the idiomatic way to describe an array of entities using c++ and object oriented programming. This data layout is called "array of structures" because we have an array of entities, which is a class (structure). If we switch to using "structure of arrays":

```c++
struct soa 
{
    vec3f* pos;
    quat*  rot;
    vec3f* scale;
}
soa entities;

void translate_soa(vec3f offset)
{
    for(u32 i = 0; i < num_entities; ++i)
    {
        entities.pos[i] += offset;
    }
}
```

Here we have a structure of arrays, the `pos`, `rot` and `scale` components are separate arrays. Because the translate_soa function only uses the `pos` array we don't need to pollute our caches with `rot` and `scale` as we do in the aos version because of the data layout. But in addition to making sure we use our cache efficiently we also want to have low overhead when applying the offset to pos, with c++ the tendency might be to use `std::vector` instead of dynamic memory for the soa components but the cost of `std::vector::operator[]` is more expensive in debug, and iterating with a raw pointer [is faster](https://www.asawicki.info/news_1691_efficient_way_of_using_stdvector) I'm not saying don't use stl, it just has some performance caveats to be aware of. These details are important if you want to have millions of entities.

## Pure(ish) free functions

Often with c++ the tendency is to make everything object oriented for the sake of it and I have seen some crazy attempts to make everything an object when other approaches might be more suited. I still like to use free functions and procedural API's they can be much nicer than a singleton or using static member functions. Here is an example one if you don't know, it's just a function and you can call it anywhere you like as long as you can see the function declaration or forward declaration:

```c++
namespace api
{
    void function();
}

void main()
{
    // just call it from anywhere, no objects in sight
    api::function();
}
```

Pure functions stipulate that:

`1. The function return values are identical for identical arguments (no variation with local static variables, non-local variables, mutable reference arguments or input streams).`  

`2. The function application has no side effects (no mutation of local static variables, non-local variables, mutable reference arguments or input/output streams).` 

A simple example pure function:

```c++
int add_integer(const int a, const int b)
{
    return a + b;
}
```

Functions like this are not technically pure but for many years I thought were:

```c++
void update_camera(camera* cam, const mat4& view, const mat4 proj)
{
    cam->view = view;
}
```

Because the cam argument is a `mutable reference argument` it fails to satisfy the second requirement of a pure function. However this style of function with all arguments passed to the function can still be a useful tool. They are easy to follow and at a glance when paired with const correctness you can quickly see what arguments may be mutated and what arguments are const.

If I receive a pull request with pure or pure-ish functions it's much easier to reason about and I can see exactly what is going on, I feel more confident approving these just because its nice and encapsulated. They also make testing, optimisation and refactoring easier because the everything is self contained in the function.

The problem is that with pure free functions we don't have a way to guarantee we haven't mutated any global or static state when we call a function (static analysis can help). We have these functions we call pure but someone at anytime could just come along and add a static variable and break the contract. But as I previously mentioned I tend to live on the edge and if I and the team say these functions are pure we know that respect it and rely on that and trust each other.

## Hidden Implementations  

Private in earlier c++ is ironically not very private at all, it's like getting naked in your bedroom window with curtains open... nicely encapsulated in your home you may feel safe but everyone can see all the nitty gritty details.

```c++
#include "massive_header_file.h"

class MySecretClass
{
    void public_function();
private:
    SomeMassiveClass m_private; // (lol)
}
```

In order to get this to compile we need to include `massive_header_file.h` so we can see the declaration of `SomeMassiveClass` so we have all that baggage I just made up but we also expose that in headers and make it hard for ourselves to decouple to interface and implementation. Hidden implementations can help us to hide more includes inside cpp files and restrict the number of compilation units header files (and subsequently structures and classes) are visible in. This helps reduce code dependency complexity and improve compile times.

When working on multi-platform code we may want a unified header for all platforms and to completely hide implementation details, so how do we do this with older c++?

### Ditch 'Single Header Library'

The single header only c++ library is a common pattern these days and it is useful, I use libraries which employ this approach but do not write my own code in this style It is not really what I like to see in a production environment that you have more control over. 

Some header may contain a macOS, Windows and Linux implementation in a single file separated by `#ifdef`'s but for some projects you may need to support many more platforms and I think that the complexity and size of the libraries warrant more consideration.

Large files with a lot of preprocessor macros for porting can quickly escalate, if you have multiple compile time branches in the file they sometimes can become hard to spot and your file is polluted with many platforms you may not currently be focusing on.

For this reason I find separating interface and implementation with a header file containing an opaque interface and cpp files containing the implementation separated per platform. There are a couple of approaches I like to use for this:

### Use premake/cmake

For multi platform development I use premake to generate solutions and projects. Previously I had toiled with Visual Studio configurations which always end up cumbersome to use. For projects where Visual Studio can be used for multiple platforms (PlayStation, Xbox, Windows) I generate a solution per platform and do not use the "excluded from build" flag on files, instead only relevant files for a platform are included in it's solution.

### Proceduracl c-style

Mentioned earlier in the post, the free function, procedural style API. My engine uses this for graphics abstraction among other things. A single header file has the renderer interface [renderer.h](https://github.com/polymonster/pmtech/blob/master/core/pen/include/renderer.h) and then per platform files supply the implementation [renderer_dx11.cpp](https://github.com/polymonster/pmtech/blob/master/core/pen/source/dx11/renderer_dx11.cpp), [renderer_metal.mm](https://github.com/polymonster/pmtech/blob/master/core/pen/source/metal/renderer_metal.mm), [renderer_vulkan.cpp](https://github.com/polymonster/pmtech/blob/master/core/pen/source/vulkan/renderer_vulkan.cpp).

```c++
// renderer.h interface
void renderer_set_index_buffer(u32 buffer_index, u32 format, u32 offset);
void renderer_set_constant_buffer(u32 buffer_index, u32 unit, u32 flags);
void renderer_set_structured_buffer(u32 buffer_index, u32 unit, u32 flags);
void renderer_update_buffer(u32 buffer_index, const void* data, u32 data_size, u32 offset = 0);
// ...

// renderer_dx11.cpp
void direct::renderer_set_index_buffer(u32 buffer_index, u32 format, u32 offset)
{
    DXGI_FORMAT d3d11_format = to_d3d11_index_format(format);
    ID3D11Buffer* buf = _res_pool[buffer_index].generic_buffer.buf;
    s_immediate_context->IASetIndexBuffer(buf, d3d11_format, offset);
}

// renderer_vulkan.cpp
void direct::renderer_set_index_buffer(u32 buffer_index, u32 format, u32 offset)
{
    VkBuffer buf = _res_pool.get(buffer_index).buffer.get_buffer();
    vkCmdBindIndexBuffer(_ctx.cmd_bufs[_ctx.ii], buf, offset, to_vk_index_type(format));
}

```

In the cpp files I use anonymous namespaces or static to keep as much code as possible accessible inside that single compilation unit only. I try minimising the use of `#ifdef`, having per platform implementation cpp files helps a lot here. Sometimes you may have multiple platforms or sub-platforms such as GLES inside an openGL implementation where having another cpp file might seem like overkill. I try here to keep preprocessor macros at the top of the cpp file and minimise the amount of interleaved code and pre-processor macros where possible, sometimes you need a big old `#ifdef` in there but minimising them can help code readability a lot.

The previous example uses an opaque `u32` for resource types (`buffer_index`) which are looked up inside `_res_pool` this system isn't very strongly typed and it's not an approach I have stuck with. Instead now I would suggest something a bit more "c++" for a graphics API abstraction layer:

### Pure Virtual Interface

This allows multiple graphics contexts and runtime graphics backend selection:

```c++
// interface header
 namespace gfx
{
    class Context
    {
    public:
        virtual Shader  createShader(const ShaderDesc& desc) = 0;
        virtual Buffer  createBuffer(const BufferDesc& desc) = 0;
        virtual Texture createTexture(const TextureDesc& desc) = 0;
    }
}

// d3d11.cpp
struct D3DShader
{
    ID3D11VertexShader*     m_vertexShader = nullptr;
    ID3D11PixelShader*      m_pixelShader = nullptr;
    ID3D11ComputeShader*    m_computeShader = nullptr;
}

class D3D11Context
{
    Shader createShader(const ShaderDesc& desc) override
    {
        D3DShader* shader = new D3DShader();
        shader->m_type = desc.m_type;

        //..
        return (Shader)shader;
    }

    void destoryShader(Shader shader) override
    {
        D3DShader* d3dshader = (D3DShader*)shader;
        // delete internal
        d3dshader->m_vertexShader->Release();
        delete d3dshader;
    }
}

// factory function, you could extern this or something
Context* createD3D11Context()
{
    return (Context*)new D3D11Context();
}
```

Where `Shader`, `Buffer` and `Textures` opaque types and can be cast to the internal implementation when passed to a platform specific function but act as an opaque handle to the outside world but they can specialise themselves internally to try and make all graphics API's behave in a similar way.

## Async API's

When porting or helping to optimise projects sometimes you have a codebase which is struggling on a single CPU core and there are idle CPU cores on your hardware doing nothing. Lets multi-thread something! easy! well, sometimes it can be and there may be low-hanging fruit but sometimes it can be hard to find stuff that can be run async when a code base has been heavily designed around serial execution.

I first used this approach to async openGL CPU driver overhead onto another thread but have since used it in many places. You can write something to code generate you the boiler plate. I'll do a quick example using the procedural API of the renderer in [pmtech](https://github.com/polymonster/pmtech). It uses the ring buffer data structure I previously mentioned in my stripped down data structures.

```c++
// render commands we want to store and defer to another thread
struct renderer_cmd
{
    u32 command_index;
    u32 resource_slot;
    u64 frame_index;

    // union of "commands"
    union {
        u32                              command_data_index;
        draw_cmd                         draw;
        draw_indexed_cmd                 draw_indexed;
        draw_indexed_instanced_cmd       draw_indexed_instanced;
        // ..
    };

    renderer_cmd(){};
};
static ring_buffer<renderer_cmd> cmd_buffer;

// allow running in single or multi-threaded (useful for debugging)
#if PEN_SINGLE_THREADED
#define add_cmd(cmd) exec_cmd(cmd)
#else
#define add_cmd(cmd) cmd_buffer.put(cmd)
#endif

// call to set_index_buffer from main thread stores args in cmd_buffer
void renderer_set_index_buffer(u32 buffer_index, u32 format, u32 offset)
{
    renderer_cmd cmd;

    cmd.command_index = CMD_SET_INDEX_BUFFER;
    cmd.set_index_buffer.buffer_index = buffer_index;
    cmd.set_index_buffer.format = format;
    cmd.set_index_buffer.offset = offset;

    add_cmd(cmd);
}

// if running multi threaded cmd buffer is consumed on another thread and calls the 'direct' API function
void exec_cmd(const renderer_cmd& cmd)
{
    // switch statement to handle commands
    switch (cmd.command_index) {
        case CMD_SET_INDEX_BUFFER:
            direct::renderer_set_index_buffer(cmd.set_index_buffer.buffer_index, cmd.set_index_buffer.format,
                cmd.set_index_buffer.offset);
        break;

        // handle more cases...
    }
}

// renderer_dx11.cpp..
void direct::renderer_set_index_buffer(u32 buffer_index, u32 format, u32 offset)
{
    DXGI_FORMAT d3d11_format = to_d3d11_index_format(format);
    ID3D11Buffer* buf = _res_pool[buffer_index].generic_buffer.buf;
    s_immediate_context->IASetIndexBuffer(buf, d3d11_format, offset);
}
```

This approach is more geared towards older CPU's with a few cores but it can be applied to many API's, if you need to read data back from the API you can buffer results into a double or triple buffer and read them at a safe time swapping the buffer each frame as long as you OK with a frames latency or so.

## Code Generation

With template meta-programming you can do some crazy and cool things for sure and once you start it can be seductive like a jedi being drawn to the dark side...  Code generation using external processes has yielded good results and allowed me to do things not possible with templates and resulted in simpler code.

I made this [python library](https://github.com/polymonster/cgu) which can parse and breakdown c-style languages to provide information about structures and functions, listing member variables, functions and arguments. Not quite as in depth as clangs AST but from a c structure declaration I have written tools which automatically generate c++ to json and back again serialisation and an automatic ImGui property browser. This tool called `serj` taking inspiration from the amazing `serde` in rust. `serj` is not open source but I can show a quick example:

```c++
// declarations
struct File
{
    std::string filepath;
};

struct Model
{
    File m_modelFilepath = {};
    File m_meshFilepath = {};

    std::string m_pipeline = "";
};
```

Running the code generation `serj` outputs the following:

```c++
struct Model
{
    File        m_modelFilepath;
    File        m_meshFilepath;
    std::string m_pipeline = "";

    bool propertyBrowser(const char* suffix = nullptr)
    {
        bool changed = false;
        ImGui::Separator();
        ImGui::PushID(suffix?suffix:"0");
        ImGui::Text("Model %s", suffix?suffix:"");
        changed |= m_modelFilepath.propertyBrowser("m_modelFilepath");
        changed |= m_meshFilepath.propertyBrowser("m_meshFilepath");
        changed |= ModelSettings::m_pipeline.showUi("Pipeline",m_pipeline);
        ImGui::PopID();
        return changed;
    }

    void deserialize(const json& j)
    {
        if(j.find("m_modelFilepath") != j.end())
        {
            m_modelFilepath.deserialize(j["m_modelFilepath"]);
        }

        if(j.find("m_meshFilepath") != j.end())
        {
            m_meshFilepath.deserialize(j["m_meshFilepath"]);
        }

        if(j.find("m_pipeline") != j.end())
        {
            m_pipeline = j["m_pipeline"].get<std::string>();
        }
    }

    void serialize(json& j) const
    {
        m_modelFilepath.serialize(j["m_modelFilepath"]);
        m_meshFilepath.serialize(j["m_meshFilepath"]);
        if(isNonDefault(m_pipeline,(std::string)"")) j["m_pipeline"] = m_pipeline;
    }
};
```

Code generation can reduce the amount of boilerplate a coder has to write and can do things that the language may not be able to do with meta programming. I extended `serj` to be able to swizzle "structure of arrays" to "array of structures" for more robust and extendible serialisation.

Here it is important to note I'm aiming for robust serialisation and not performance, serialising to json means we can merge files but it isn't the fastest. `serj` could also generate a binary serialisation code which would give speed and could be used in a final product once the serialisation layout has been nailed down.

## Scoped Enum

I always found enum handling to be a poor area in c++ I hoped `enum_class` would sort me out but I preferred my older school alternative because it can be used as an int or as bit-mask as well. 

```c++
namespace e_shader_type
{
    enum shader_type
    {
        vertex,
        fragment,
        geometry,
        compute
    };
}
typedef e_shader_type::shader_type ShaderType;
```

This gives you nice scope so you can type `e_shader_type::` and get auto-complete in most IDE's. You can typedef `ShaderType` to make a stronger type to pass around. You can `using namespace e_shader_type` and then `flags |= vertex` to minimise the code a bit and I do like keeping code narrow.

You can also can use this for flags and bit-masks, so everything is nicely group together and consistent.

```c++
namespace e_texture_usage
{
    enum texture_usage : u32
    {
        shader_read = 1<<0,
        shader_write = 1<<1,
        render_target = 1<<2,
        depth_stencil = 1<<3
    };
}
typedef u32 TextureUsage;

void func()
{
    u32 m_flags = e_texture_usage:shader_read | e_texture_usage:render_target;
}
```

## That's it

Just a few examples here, to be honest sometimes I just need loops and ALU ops, pointers and pointer arithmetic... having to add more code and layers of abstraction comes at a cost and I try and keep this as bare-metal as possible certainly for performance critical code. I start to use more generic programming techniques where the benefits of extendability, code re-use and ease of use work for me. In tools and build pipelines I am more inclined to worry less about performance focus on usability but also in the main update I like to keep my ear to the ground.
