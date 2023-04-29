---
title: 'maths / maths-rs'
subtitle: 'Linear algebra libraries for C++ and Rust'
date: 2020-09-29 00:00:00
featured_image: '/images/pmtech/gifs/maths-functions.gif'
---

Linear algebra libraries written in C++ and Rust with vector swizzling and a comprehensive collection of useful functions for games and graphics development. A good maths library is an essential tool for any games or graphics programmer. I built the C++ library and have maintained it since I began my journey into game development.

There is a live [WebGL demo](https://www.polymonster.co.uk/pmtech/examples/maths_functions.html) of the intersection functions, this visual representation was also used to generate and verify test cases which run continuously via [GitHub Actions](https://github.com/polymonster/maths/actions);

The C++ library's most interesting feature is probably the vector swizzling, it uses C++11's template parameter pack to allow shader style swizzles. I wrote a rust program [permute](https://github.com/polymonster/permute) which code generates the C++ template permutations in [swizzle.h](https://github.com/polymonster/maths/blob/master/swizzle.h)

```c++
vec4f swizz = v.wzyx;       // construct from swizzle
swizz = v.xxxx;             // assign from swizzle
swizz.wyxz = v.xxyy;        // assign swizzle to swizzle
vec2f v2 = swizz.yz;        // construct truncated
swizz.wx = v.xy;            // assign truncated
swizz.xyz *= swizz2.www;    // arithmetic on swizzles
vec2 v2 = swizz.xy * 2.0f;  // swizzle / scalar arithmetic

// sometimes you may need to cast from swizzle to vec if c++ cant apply implicit casts
f32 dp = dot((vec2f)swizz.xz, (vec2f)swizz.yy):
```

I later ported the C++ library functionality to Rust and learned a lot about generics and traits in the process. I tried to make the libs stylistically similar so it is easy to port C++ code to Rust and vice-versa. So you can call generic C++ style maths functions like in shader code.

```rust
// numeric ops
let int_abs = abs(-1);
let float_abs = abs(-1.0);
let int_max = max(5, 6);
let float_max = max(1.0, 2.0);
let vec_max = max(vec3f(1.0, 1.0, 1.0), vec3f(2.0, 2.0, 2.0));
let vec_min = min(vec4f(8.0, 7.0, 6.0, 5.0), vec4f(-8.0, -7.0, -6.0, -5.0));

// float ops
let fsat = saturate(22.0);
let vsat = saturate(vec3f(22.0, 22.0, 22.0));
let f = floor(5.5);
let vc = ceil(vec3f(5.0, 5.0, 5.0));

// vector ops
let dp = dot(vec2, vec2);
let dp = dot(vec3, vec3);
let cp = cross(vec3, Vec3::unit_y());
let n = normalize(vec3);
let qn = normalize(quat);
let m = mag(vec4);
let d = dist(vec31, vec32);

// interpolation
let fi : f32 = lerp(10.0, 2.0, 0.2);
let vi = lerp(vec2, vec2, 0.5);
let qi = lerp(quat1, quat2, 0.75);
let qs = slerp(quat1, quat2, 0.1);
let vn = nlerp(vec2, vec2, 0.3);
let f = smoothstep(5.0, 1.0, f);
```

For more information or to clone or fork this repository please visit the GitHub pages.  

<a href="https://github.com/polymonster/maths" class="button button--large">GitHub (maths)</a>
<a href="https://github.com/polymonster/maths-rs" class="button button--large">GitHub (maths-rs)</a>
