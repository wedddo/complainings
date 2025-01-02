---
title: "The Pain of Initialising OpenGL"
date: 2023-04-30
---

##### *If you're going to make your API forward compatible, don't do it like this.*

## What is OpenGL?
It turns out that drawing geometry to a screen is computationally expensive, particularly if that geometry is of the 3D variety. Luckily, the hardware wizards up in their tower somewhere blessed us with GPUs, which do this dark magic for us stupidly fast. 
OpenGL (short for Open Graphics Library) is an application programming interface (API) used for communicating with the drivers for your GPU. There are a couple of different graphics APIs around, but they all do pretty much the same thing and OpenGL just happens to be the one I somewhat know how to use.

## Getting Started with Getting Started
An internet search tells me that OpenGL has been around since 1992, and I'm not going to question that since I wasn't alive at the time. It's probably been on Windows since Windows NT (basically medieval times). That means you don't need to download or install it from anywhere - if you have a Windows machine from the last 30 years, you can be pretty confident it has OpenGL on it.

In order to use OpenGL, you *could* just use libraries like [GLFW](https://www.glfw.org/) (GL FrameWork) to handle the OS interfacing and [GLEW](https://glew.sourceforge.net/) (GL Extension Wrangler) to get the functions, but I'm not about making life that easy, so we're doing this with just the Win32 and OpenGL APIs.

## Ask Nicely
First, we need to tell Windows what we actually want to OpenGL for and the pixel format we expect to use (since wanting to use a graphics card to draw graphics to a RGB screen must be a niche thing or something). We can tell Windows this by filling out the [`PIXELFORMATDESCRIPTOR`](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/ns-wingdi-pixelformatdescriptor) structure. 

Whoops, did I say we *tell* Windows what we want? Who did you think was in charge here? In reality when we give Windows our format it just chucks it out and gives us a format it made instead. And if we're lucky, it might be similar to the one we asked for.

Well that was weird, but we're pretty much done, yeah?  We use [`wglCreateContext()`](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-wglcreatecontext) to create an OpenGL rendering context that fits with the pixel format and our Windows device context, then we are ready to draw stuff, right? Right?

## The Chicken and Egg Problem
When I mentioned earlier that we can be certain that OpenGL is on our computer, I wasn't *technically* lying. We can, in fact, be sure that we have OpenGL *1.0*. But OpenGL (and GPUs in general) have changed a lot in 30 years, so we probably want to be using something a bit newer than version 1.0. But not to worry, we can use `wglCreateContextAttribsARB()`, which lets us pass in some attributes when we create our rendering context, such as the version we want to target and what kind of compatibility we want to have with other versions. Perfect!

You might have two questions at this point - "why didn't we just do that to start with?" and "why does that function end with `ARB`?". `ARB` means the function is an extension which has passed the Architecture Review Board, and we didn't get the rendering context this way first because *you cannot use an extension before OpenGL is initialised*. That's right, you need a rendering context to get your rendering context.

## A Quick Note on OpenGL extensions
OpenGL development isn't purely driven by Khronos Group, who owns it - it is mainly driven by GPU vendors. If, for example, Nvidia had some new functionality it wanted to expose to users, it would add a new `gl` extension function with the suffix `NV`. If vendors collaborate on an extension function, it would be suffixed with `EXT`.  If the extension passes review, and therefore all vendors should implement it, it is given the `ARB` suffix. Finally, when it is incorporated in a major release, it loses it's suffix and becomes an OpenGL core function.

In the same way, Windows has extensions for it's `wgl` functions, in order to provide functionality that needs to interface with the operating system.

All this makes sense as a progression for new functionality, however it means that 2 GPUs running the same version of OpenGL may have different extensions, and there may be multiple versions of the same function with different suffixes, or no suffix at all. Sometimes these different functions might have the same functionality, sometimes they are subtly different. Its a mess.

## Another Chicken and Egg
So, we need to use `wglCreateContextAttribsARB()` to get a new context, and we can use [`wglGetProcAddress()`](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-wglgetprocaddress) to ask Windows to give us a function pointer for that function. But hold on, what if our version of OpenGL doesn't support that extension? I don't trust what `wglGetProcAddress()` will return in that case, and I don't want to be trying to call invalid memory as if it was a function. Luckily, we can get a list of extensions our GPU driver and pixel format supports, and parse through to see if our function is on there. Not as ideal as just directly asking if an extension is supported, but its something. 

We can get the list of `wgl` extensions by calling `wglGetExtensionStringARB()`. 

*Which is an extension*. 

*Are we meant to check if this function exists by calling it?!*

Guess we are just going to assume that one does exist.

## On the Home Stretch... Towards Getting Started
At this point, we are finally able to call the function to get a rendering context which uses an OpenGL version of our choosing. At this point, we can fetch all of the function pointers we will need via `wglGetProcAddress()`. We can also check any `gl` extensions by seeing if it is in the extension string returned by [`glGetString(GL_EXTENSIONS)`](https://docs.gl/gl4/glGetString), although clearly at some version they switched to having to check via `glGetStringi(GL_EXTENSIONS, index)`. On the bright side, neither of these functions are extensions, so you avoid the silly `wgl` issue.

We also need all of the macro values that OpenGL functions need to be passed, which can be found in various header files on the [Khronos Github](https://github.com/KhronosGroup/OpenGL-Registry/tree/main/api/GL). These headers also have typedefs for the functions types if you can't be bothered doing them yourself.

## Done!
And that's it! We have initialised OpenGL! Well done, you can pat yourself on the back. What's that, we haven't even drawn a pixel on the screen yet? it's 2 lines of code to do this with GLFW and GLEW? Shut up.