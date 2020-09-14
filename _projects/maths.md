---
title: 'maths'
subtitle: 'Linear algebra library for c++.'
date: 2020-09-14 00:00:59
featured_image: '/images/pmtech/gifs/maths-functions.gif'
---

A linear algebra library written in c++ with vector swizzling and a comprehensive collection of useful functions for games and graphics development. A good maths library is an essential tool for any games or graphics programmer. I have built this library up and maintained it since I began my journey into game development.

There is a live [WebGL demo](https://www.polymonster.co.uk/pmtech/examples/maths_functions.html) of the intersection functions, this visual representation was also used to generate and verify test cases which run continuously via [Travis CI](https://travis-ci.org/github/polymonster/maths) and [AppVeyor](https://ci.appveyor.com/project/polymonster/maths).

It's most interesting feature is probably the vector swizzling, it uses c++11's template parameter pack to allow shader style swizzles. I wrote a rust program [permute](https://github.com/polymonster/permute) which code generates the c++ template permutations in [swizzle.h](https://github.com/polymonster/maths/blob/master/swizzle.h)

```c++
vec4f swizz = v.wzyx;       // construct from swizzle
swizz = v.xxxx;             // assign from swizzle
swizz.wyxz = v.xxyy;        // assign swizzle to swizzle
vec2f v2 = swizz.yz;        // contstruct truncated
swizz.wx = v.xy;            // assign truncated
swizz.xyz *= swizz2.www;    // arithmetic on swizzles
vec2 v2 = swizz.xy * 2.0f;  // swizzle / scalar arithmentic

// sometimes you may need to cast from swizzle to vec if c++ cant apply implict casts
f32 dp = dot((vec2f)swizz.xz, (vec2f)swizz.yy):
```


For more information or to clone or fork this repository please visit the github page.

<a href="https://github.com/polymonster/maths" class="button button--large">GitHub</a>