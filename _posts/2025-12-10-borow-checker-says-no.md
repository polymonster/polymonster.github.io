---
title: 'Borrow checker says “No”! An error that scares me every single time!'
date: 2025-10-31 00:00:00
---

It’s Halloween and I have just been caught out by a spooky borrow checker error that caught me by surprise. It feels as though it is the single most time consuming issue to fix and always seems to catch me unaware. The issue in particular is “cannot borrow x immutably as it is already borrowed mutably” - it manifests itself in different ways under different circumstances, but I find myself hitting it often when refactoring. It happened again recently so I did some investigating and thought I would discuss it in more detail.  
 
The issue last hit me when I was refactoring some code in my graphics engine [hotline], I have been creating some content on YouTube and, after a little bit of a slog to fix the issue, I recorded a video of me going through the scenario of how it occurred and some patterns to use that I have adopted in the past to get around it. You can check out the video if you are that way inclined, the rest of this post will mostly echo what is in the video, but it might be a bit easier to follow code snippets and description in text.  
 
[video]
 
I have a generic graphics API, which consists of traits called [gfx]. This is there to allow different platform backends to implement the trait; currently I have a fully implemented Direct3D12 backend and I recently began to port macOS using Metal.
 
The gfx backend wraps underlying graphics API primitives; in this case we are mostly concerned about `CmdBuf` which is a command buffer. Command buffers are used to submit commands to the GPU. They do things like `draw_indexed_instanced` or `set_render_pipeline`, amongst other things. For the purposes of this blog post, what the command buffer does is not really that important, just that is does `do_something`, which at the starting point when the code was working is a trait method that takes an immutable self and another immutable parameter ie. `fn do_something(&self, param: &Param)`.
 
In the rest of the code base I have a higher level rendering system called `pmfx`. This is graphics engine code that is not platform specific but implements shared functionality. So where `gfx` is a low level abstraction layer, `pmfx` implements concepts of a `View` that is a view of a scene that we can render from. A `View` has a camera that can look at the scene and is then passed to a render function, which can build a command buffer to render the scene from that camera's perspective. The engine is designed to be multithreaded and render functions are dispatched through `bevy_ecs` systems, so a view gets passed into a render system but it is wrapped in an `Arc<Mutex<View>>`. 
 
I made a small cutdown example of this code to be able to demonstrate the problem I encounter, so let’s start with the initial working version:
 
```rust
use std::sync::Arc;
use std::sync::Mutex;
 
struct Cmd;
 
struct View {
    cmd: Cmd,
    param: Param
}
 
struct Param;
 
impl Cmd
{
    fn do_something(&self, param: &Param) {
        unimplemented!("");
    }
}
 
fn get_view() -> Arc<Mutex<View>> {
    unimplemented!();
}
 
fn main() {
    let view = get_view();
    let mut view = view.lock().unwrap();
 
    view.cmd.do_something(&view.param);
}
```
 
I tried to simplify it as much as possible so these snippets should compile if you copy and paste them, they won’t run thanks to `unimplemented!` macro (which I absolutely love using, it is so handy!) but we only care about the borrow checker anyway.
 
All we really need to think about is that a `Cmd` can `do_something` and it also gets passed in a `Param`, which is also contained as part of ‘view’. Coming from a C/C++ background I landed on my personal preference being procedural C code with context passing, so I tend to group things together into a single struct. It makes sense to me in this case and I wanted to group everything inside `View`, and we fetch the view from elsewhere in the engine.
 
So the code in the snippet compiles fine and I was working with this setup for some time. I began work on macOS and it turned out that the `do_something` method needed to mutate the command buffer so that I could mutate some internal state and make the Metal graphics API behave similarly to Direct3D12. This is common for graphics API plumbing.
 
The specific example in this case was that in Direct3D we call a function `bind_index_buffer` to bind an index buffer before we make a call to `draw_indexed`, but in Metal there is no equivalent to bind an index buffer. Instead you pass a pointer to your index buffer when calling the equivalent draw indexed. So to fix this, when we call `bind_index_buffer` we can store some extra state in the command buffer so we can pass it in the later call to `draw_indexed`.
 
In hindsight any method on the command buffer trait that does anything, like set anything or write into the command buffer, should take a `&mut self` because it is mutating the command buffer after all. In my case since I am calling through to methods on  `ID3D12CommandList`, which is unsafe code and does not require any mutable references.
 
In our simplified example, in order to store, state `do_something` now needs to change and take a mutable self: `do_something(&mut self, param: &Param)` it should be noted that `view` itself was already `mut`.
 
```rust
impl Cmd
{
    fn do_something(&mut self, param: &Param) {
        unimplemented!("");
    }
}

fn main() {
    let view = get_view();
    let mut view = view.lock().unwrap();
 
    view.cmd.do_something(&view.param);
}
```
Borrow checker now kicks in…my heart sinks. In the real code base not only did I have to modify a single call site, but I had hundreds of places where this error was happening, I made the decision here and now to make any methods that write to the command buffer also be mutable and make the mutability 
 
```
error[E0502]: cannot borrow `view` as immutable because it is also borrowed as mutable
  --> src/main.rs:30:28
   |
30 |     view.cmd.do_something(&view.param);
   |     ----     ------------  ^^^^ immutable borrow occurs here
   |     |        |
   |     |        mutable borrow later used by call
   |     mutable borrow occurs here
 
For more information about this error, try `rustc --explain E0502`.
error: could not compile due to 1 previous error
```
This is not the first time I have encountered this problem and I doubt it will be the last. There are a number of ways to resolve it and they aren’t too complicated. The frustrating thing is that it seems to occur always when you are doing something else and not just when you decide to refactor, so you end up having a mountain of errors to solve before you can get back to the original task. I suppose you could call it a symptom of bad design or lack of experience, but when writing code things inevitably change and bend with new requirements, and Rust throws these unexpected issues up for me more often than I find with C, and often the required refactor takes more effort as well. But that is the cost you pay, hopefully more upfront effort to get past the borrow checker means fewer nasty debugging stages later. So let’s look at some patterns to fix the issue!
 
Take
 
The one I actually went for in this case was using `std::mem::take`. We take the `CmdBuf` out of view so we no longer need to borrow a ‘view’ to use `cmd`, and then when finished return the cmd into ‘view’. It is important to note here that `CmdBuf` needs to derive default in order for this to work, as when we take the `cmd` in `view` will become `CmdBuf::default()`
 
```rust
#[derive(Default)]
struct Cmd;
 
// .. 
 
fn main() {
    let view = get_view();
    let mut view = view.lock().unwrap();
 
    // take cmd out of view
    let mut cmd = std::mem::take(&mut view.cmd);
 
    // the immutable and mutable references are now split
    cmd.do_something(&view.param);
 
    // return the cmd into view
    view.cmd = cmd;
}
```
 
This approach is the simplest I could think of at the time because any existing code using `view.cmd` doesn't need updating, everything stays the same and we just separate the references. In this case it was easy to derive the default for  `CmdBuf`.You need to remember to set the `cmd` back on `view` here, which could be a pitfall and cause unexpected behaviour if you didn't.
 
Clone
 
If you can’t easily derive default on a struct there are some other options. If the struct is clonable or you can easily derive a clone, you can clone to achieve a similar effect.
 
```rust
#[derive(Clone)]
struct Cmd;
 
// ..
 
fn main() {
    let view = get_view();
    let mut view = view.lock().unwrap();
 
    // clone cmd
    let mut cmd = view.cmd.clone();
 
    // the immutable and mutable references are now split
    cmd.do_something(&view.param);
}
```
 
Cloning might be considered a heavier operation than ‘take’ depending on the circumstances, but this method has the same benefit as the take version whereby unaffected code that is using `cmd` elsewhere doesn’t need to be changed.
 
RefCell
 
Another approach would be to use `RefCell` this allows for interior mutability and again we do not need to worry about default or clone.
 
```rust
use std::cell::RefCell;
 
struct Cmd;
 
struct View {
    cmd: RefCell<Cmd>,
    param: Param
}
 
fn main() {
    let view = get_view();
    let mut view = view.lock().unwrap();
 
    // borrow ref cell
    let mut cmd = view.cmd.borrow_mut();
 
    // the immutable and mutable references are now split
    cmd.do_something(&view.param);
}
 
```
 
Option (Take/Swap)
 
We also need to update any code that ever used `view.cmd` and do the same. Not ideal but it allows us to get around the need for a default or clone. I have had to resort to this in other places in the code base.
 
There are more options; quite literally `Option` here can help. If we make `cmd` an `Option<CmdBuf>` then this gives us the ability to use `None` as the default and we can use the `std::mem::take` approach. We can also use `std::mem::swap` and swap with `None`. Swapping works similar to ‘take’, where we take mem and swap with the default.
 
```rust
struct Cmd;
 
struct View {
    cmd: Option<Cmd>,
    param: Param
}
 
fn main() {
    let view = get_view();
    let mut view = view.lock().unwrap();
 
    // clone cmd
    let mut cmd = std::mem::take(&mut view.cmd);
 
    // the immutable and mutable references are now split
    cmd.as_mut().unwrap().do_something(&view.param);
 
    // return the cmd to view
    view.cmd = cmd;
}
```
 
The `Option` approach also requires more effort as we need to now take a reference and unwrap the option and update any code that ever used `view.cmd` to do the same. Not ideal, but it allows us to get around the need for a default or clone, and if your type is already optional then this will fit easily.
 
Interior Mutability
 
There is one final approach that could save a lot of time, and that would be to not change the `do_something` function at all in the first place. That is to keep it as `do_something(&self, param: &Param)`. So how do we mutate the interior state without requiring the self to be mutable?
 
This can be done with `RefCell` in single threaded code or `RWLock` in multithreaded code. Since we already looked at `RefCell` I will do an example of `RWLock`.
 
```rust
struct Cmd {
    interior: Arc<RwLock<u32>>
}
 
impl Cmd
{
    fn do_something(&self, param: &Param) {
        // we now mutate the interior, locking and writing in a thread say way
        let interior = self.interior.try_write().and_then(|mut interior| {
            *interior = 1;
            Ok(())
        });
    }
}
 
fn main() {
    let view = get_view();
    let view = view.lock().unwrap();
 
    // code at the call site can stay the same as the original
    view.cmd.do_something(&view.param);
}
```
 
Decisions
 
I decided to make the mutability explicit to the trait and that was based on how the command buffers are used in the engine, in other places I have taken other approaches favouring interior mutability. For this case a view can be dispatched in parallel with other views, but the engine is designed such that 1 thread per view and no work happens to a single view on multiple threads at the same time. Command buffers are submitted in a queue in order and dispatched on the GPU.
 
Here it made sense to me to avoid locking interior mutability for each time we call a method on a `CmdBuf` and it works with the engine's design. We lock a view at the start of a render thread, fill it with commands and then hand it back to the graphics engineer for submission to the GPU. The usage is explicit, we just needed to appease the borrow checker!
 
I hope you enjoyed this article, please check out my YouTube channel for more videos or more articles on my blog, let me know what you think and if you have any other strategies or approaches I would love to hear about them. I would also like to hear about compiler and borrow checker errors you find particularly time consuming or frustrating to deal with.
