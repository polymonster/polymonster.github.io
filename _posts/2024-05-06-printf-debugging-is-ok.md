---
title: 'printf debugging is OK'
date: 2024-05-06 00:00:00
---

I stopped going on Twitter a while ago because it has the tendency to evoke rage, as it is designed to do. But every now and then I check back in - it can be useful sometimes for keeping up with graphics research, gamedev news and some people do post nice things, like sharing projects they are working on, so there is something to pull me back from time to time.

After checking the other day I saw this debate going around about not using an IDE or debugger, just using ‘notepad’ to write code. I looked in the comments, people arguing about who was right and all the usual toxic vibes, and it reminded me of some earlier occasions of people discussing the same topic.

It feels like the same old debate has been going on for a long time now, it’s packaged differently each time, but I don’t really know why people get so wound up about things. The main arguments are “if you need to use a debugger you’re an idiot and you don’t understand the code you are writing” (that’s not an actual quote but there was a similar take along those lines). Then there is “If you can’t use a debugger you’re an idiot”. The hating on the ‘printf’ crew is omnipresent.

At the risk of poking a hornet's nest, I just wanted to share some thoughts and ideas on this subject in a balanced way, because I don’t think there needs to be an ultimate solution here. We need to debug code and there are tools out there to help us, some are more useful than others in certain situations, but at the end of the day do whatever you need to do to fix those bugs.

## Debuggers

I use a debugger regularly, I will launch most work in C++ from Visual Studio or Xcode and preferably run in a debug build. I know for some people this is often a terrible UX because of the performance of debug builds, so a prerequisite here is fast debug builds. This is hard to retrofit but having a usable debug build is useful. Once running I can use the debugger break and step if I need to, and if I encounter a crash then there is a nice call stack I can look through in more detail.

I have noticed that it is extremely common for graduate and junior software engineers to have little to no debugging knowledge or experience. It’s not something that seems like it is taught at university and I have also been told stories of teachers imposing their usage of VIM and esoteric debugging strategies upon the students. For the record I am not a VIM user (another topic that ends up in polarising debates). I find using a mouse and 2 finger typing works for me.

The moment when you show someone how to use a hardware breakpoint or a watchpoint and find a bug immediately is like seeing the lightbulb appear on top of their head, a whole world of possibility opening in front of them, or the dismay of the wasted hours trying to catch some dodgy logic through layers and layers of object oriented spaghetti.

Some of those argue about using only ‘notepad’ and no debugger because they can dry run their code on paper and they “don’t write bugs”, but I find it difficult to understand how they work within a larger team project or codebase . A lot of bugs and issues I have ever had to fix were not in code I wrote myself, they were in legacy systems, colleague’s code, or in open source code (and some hard as nails bugs to track too!) that had been just lifted into a project. If you believe in the impending AI coding apocalypse then human engineers may merely be around to debug and fix issues with AI generated code. So yeah, being able to write perfect code yourself is one thing, but using a debugger to debug existing code in a large complex project shouldn’t be a thing of shame and we might need all the tools we can to help.

Along with debuggers we get all sorts of other tools, which also should be used as and when we need them. Address sanitizer can catch memory issues easily, where in a bygone era we would have this 1 in 1000 crash somewhere reading outside of an array bounds, we can enable ASan and catch this every time without the undefined behaviour lottery. Same for undefined behaviour sanitizer, now we can catch UB when it’s benign and not only when a noticeable side effect occurs.

I don’t know if these notepad-only coders are taking all of those tools off the table as well, but when you have something like ASan that can catch an issue for you I just don’t really know why you wouldn’t use it. I have seen a lot of comments that seem to suggest the debugger slows them down, but in this case I certainly think the debugger speeds you up.

So if you’re reading this and you don’t know about these tools I would say take a look and see, they can be useful and might be able to save you a lot of time. There are tons of things you can do and it’s hard to cover it all here. I learned a lot from working with other people and side by side debugging difficult problems. I think there should be more resources to teach these skills instead of it being handed down information.

## Printf debugging is OK

So for the ‘printf’ haters I would also say that whilst using a debugger most of the time, sometimes I revert to ‘printf’ debugging. There are some situations where there is no other choice - in the past I have had to debug release builds where we were unable to reproduce the bug in debug. Even pulling in debug modules for the engine (for on screen debug info) changed the executable such that we couldn’t reproduce the issue. The last thing was to put a few print statements in using the raw ‘printf’ and removing them and adding more as we narrowed down the issue and eventually extracted enough information to fix the problem.

I have also had the need to use ‘printf’ when debugging certain kinds of behaviours in an application. In the case of something like touch event tracking for mobile devices, if you try to debug an issue with breakpoints you interrupt the hardware and it makes it difficult to reproduce issues in the same way they appear naturally. So here printing the state of touch down events, touch up events, and being able to see the logical flow can identify a problem. There are many more scenarios that benefit from this type of debugging. Just throw the prints in and make sure to remove them after so no one knew you were ever there, like a ninja.

## Custom tools

Custom UI based debugging tools can go one step further than printf debugging, providing some similar traits but also allowing more flexibility and controllability, I assume the notepad wielders who don’t use a regular debugger must have some such custom tools and things to help them track down issues. I am a big fan of embedded debugging and profiling tools within an application. You know stuff like performance counters that I can just pop-up in a UI or tweakable values to help to refine behaviours or visual appearance. I find that since the explosion of ImGui the level of integrated ad-hoc debugging tools and info has exponentially increased.

But with these kinds of custom tools, I personally wouldn't try and re-invent the wheel. I would aim to make stuff that complements the existing tools I can pull off the shelf. So for example I like to have a quick, at a glance profiler for all my key performance hotspots that I can check whenever I notice something. But for more in-depth profiling I would use a CPU or GPU profiler to dig deeper.

## Just doing what needs to be done

At the end of the day, finding bugs is just something that we need to get done, whatever helps you find and fix the issue doesn’t bother me as long as we get the job done. On a closing note, I noticed some code in a pull request left in by accident by another person:

```c++
if(some_condition) {
    int x = 0;
}
```

I found this interesting. I do the same thing except I usually name my variable ‘a’. This is to insert some code where a breakpoint can be put on the ‘int x’ line and then it kind of acts like a conditional breakpoint when some_condition is true. You could use a conditional breakpoint within the debugger, but they can be slow and for me historically unreliable, but this little snippet gives you your own conditional breakpoint that works without fail.

Just make sure to remove the code before the PR next time!
