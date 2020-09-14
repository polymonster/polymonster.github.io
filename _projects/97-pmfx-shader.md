---
title: 'pmfx-shader'
subtitle: 'Cross platform shader system.'
date: 2020-09-13 00:00:00
featured_image: '/images/pmfx/pmfx.png'
---

This cross platform shader system originated in pmtech but is now also being used in the engine I am currently developing at [Flavourworks](https://www.flavourworks.co/). It is written in python and provides a command-line interface to configures and build shaders for different platforms. 

It uses a mixture of pre-processor macros and code generation to target HLSL, Metal, GLSL and SPIR-V and comes with shader compilers and validators built in to support byte-code as well as source code shader representations.

In addition to allowing a single unified language to target different platforms, pmfx-shader also adds useful features such as techniques, uber-shader permutations, reflection and c++ code generation to aid fast development, code sharing and data-driven configurability.

<a href="https://github.com/polymonster/pmfx-shader" class="button button--large">GitHub</a>

 