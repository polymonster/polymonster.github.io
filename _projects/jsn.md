---
title: 'jsn'
subtitle: 'A relaxed, user-friendly json-like data format.'
date: 2020-09-26 00:00:00
featured_image: '/images/jsn.png'
---

jsn is a user-friendly data format that can be easily read and reliably edited by humans. It adds powerful features such as inheritance, variables, includes and syntax improvements to make jsn files more compact and re-usable than a json counterpart, it is an ideal solution for multi-platform build configuration, packaging and content building pipelines.

Example extract of a jsn config to setup a multi-platform code and data build pipeline:

```c++
base:
{
    clean: {
        directories: [
            "${data_dir}"
            "${bin_dir}"
            "${build_dir}"
            "temp"
        ]
    }
    
    copy: {
        files: [
            ["assets/source/fonts", "${data_dir}/fonts"]
            ["assets/source/lua", "${data_dir}/lua"]
            ["assets/source/timelines/*.json", "${data_dir}/timelines"]
            ["assets/imgui_default.ini", "${data_dir}/imgui_default.ini"]
            // 2 meta data files for syncing data
            ["assets/fwbuild_init.jsn", "${bin_dir}/fwbuild_init.jsn"]
            ["assets/config.jsn", "${bin_dir}/config.jsn"]
        ]
    }
}

ios-library(base): 
{
    jsn_vars: {
        data_dir: "bin/ios_library/data"
        bin_dir: "bin/ios"
        build_dir: "build/ios"
    }
    
    premake: {
        args: [
            "xcode4"
            "--library"
            "--wwise"
            "--platform=ios"
            "--file=tv_editor.lua"
            "--fwtechdir=${fwtechdir}/"
            "--scripts=${fwtechdir}/third_party/premake-android-studio"
            "--curl_fetch"
        ]
    }
            
    pmfx: {
        args: [
            "-platform osx"
            "-shader_platform metal"
            "-i assets/source/shaders ${fwtechdir}/assets/shaders"
            "-o assets/built/ios/data/shaders"
            "-h shader_structs"
            "-t assets/temp/ios"
            "-metal_sdk iphoneos"
            "-metal_min_os 10.9"
        ]
    }
    
    premake: ["xcode4 --library --wwise --platform=ios --file=tv_library.lua"]
}
```

Check out the GitHub page for more information and how to use.

<a href="https://github.com/polymonster/jsn" class="button button--large">GitHub</a>