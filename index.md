# pmtech

## Introduction

pmtech is a lightweight, cross-platform, multithreaded 3D engine currently supporting Windows, MacOS, Linux and iOS with OpenGL3.1+, OpenGLES3+ and Direct3D11 renderers. Take a look at the [github](https://www.google.com) repository to see the source code.

A common question other coders ask me when I talk about developing my own engine from the ground up is why don't I just use unity or unreal engine. This introduction is to answer that question and to discuss my motivations behind the project.

I have worked now for around 10 years professionally as a programmer on low level code for proprietary engines, graphics, tools and build and content pipelines. So I am somewhat doing in my sparetime for fun what I also do for my day job, but I have found working on my own tech to be more liberating and enjoyable as I can take a more idealistic approach to the code I am developing.
 
Initially the project started to give myself a framework in which I could easily start new projects to prototype and test new ideas, after that it took the path of trying to address some of the frustrating issues I had encountered in my professional work and later it has taken on board some of the more succesful features of engines I have worked on for my job.

I wanted to make the codebase as simple as possible. Having seen code bases that had grown and become extremely complex over time, I was always tasked with last minute optimisations which often led to the analysis that the engine needed to be restructured or redesigned (but we had to make do with some hacky optimisations as opposed to re-writing the engine). This was because a lot of the performance cost was in the way objects communicated with one another, deep call stacks and complex hierarchies cause cache misses and carry a cost to simply call functions and when drilling down into a function body there was often hardly any time spent doing arithmetic or data transformation.

For this reason I decided to take a more c-style data oriented approach and try to avoid using object oriented paradigms as much as possible, I am still getting decent code re-use using static polymorphism through templates, operator overloading and function overloading and probably one of the most notable features of the code base is a
data oriented component entity system.

I wont go into to much more detail about object oriented vs data oriented approaches as there is already a lot written about it already but the data oriented approach naturally lends itself to performance because it is less bloated and more cache friendly and aims toward transforming large amounts of homogenous data instead focusing on individual objects. data oriented code also lends itself to be easily optimised with SIMD which I intend to use where possible to increase throughput even further. 

Multithreading is built into the core systems from the outset. The renderer, audio and physics get processed on dedicated threads and the user thread which contains the main scene and logic is also on a dedicated thread.

With simple and minimal c-style api's I have wrapped up all platform specific code into a very small set of modules, renderer, os (window, input), threads, timer, memory and file system. All platforms share the header for each module which contains a simple procedural api and the platform specific details are hidden in a cpp. The idea behind this is that the lower level os stuff is handled early, files are included or excluded on a platform basis, use of ifdefs is kept to a minimum (because they can be messy and ugly) and all of the more complex code that builds on top of these api's is completely platform agnostic. The amount of code required to port a platform is minimal and I am making use of posix of stdlib where applicable so there are even less platform specific files than there are platforms.

So thats about it, simple minamilistic api's, data oriented and multithreaded code are the focus of this project.

## Getting Started

Starting a new project in pmtech is quick and easy, one of the key features I wanted in a codebase was the ability to create new projects which has all the shared functionality of common apis but could also be stand alone eith their own very bespoke code or contain ad-hoc stuff that could be thrown away later.  







