---
title: "Hot Reloading is Hot"
date: 2025-07-13
---

##### *Lose the idea that making changes requires a restart.*


## Programming productivity is all about the iteration cycle
I mean, what more needs to be said? Programming is all about iteration - we make changes, test those changes, realise we are stupid and the changes don't work, make more changes, maybe this time they kinda work, etc. You can never expect us programmers to get it right the first time, every time (that would be ridiculous). But what you can expect is for programmers to maximise their iteration speed by making that iteration loop as tight as possible. 

Sadly, I don't see many programmers taking this seriously, and many projects have aspects that make this cycle slow - whether it is slow compile times, slow startup times, convoluted building or debugging setups, or any one of many other things that can slow you down. This article will be going over (in my opinion) the best way to tighten that iteration loop: *hot reloading*.

## But wait, what *is* hot reloading?
The idea behind hot reloading is simple. As opposed to *cold reloading*, which is where you must wait for some external stimulus to stop the code from running before making changes (ie. my fingers hitting *alt-F4*), *hot-reloading* makes those code changes possible *while the program is still running*. 

"But wait!" I hear you say. "How can we change out the code, while it is still running?" Unfortunately, I'd have to agree with you - that's not really possible, at least not without some complex system to manage it. Fortunately however, I can let you in on a little secret: code spends most of it's time **not running**.

## Identifying the main loop
While this doesn't apply to **all** software, **most** software is built around some kind of loop where the code does some work, then waits for some criteria to be met before doing more work, over and over again. For command line interfaces, it is waiting for user input. For servers, it is waiting for network traffic. For GUI applications, it is waiting for the monitor to refresh the pixels on the screen, so that the next frame can be processed. Unless you are dealing with an extremely short-lived program, it is likely your program's control flow has some kind of main loop, somewhere.

To use a simplified example of a C++ Windows GUI application using OpenGL, the loop may look something like this:
```
int WinMain(HINSTANCE instanceHandle, HINSTANCE prevInstanceHandle, char* commandLine, int windowShowType) {
    
    AppState state = {};

    state.window = createOutputWindow();
    state.openGLRC = initOpenGL(state.window.handle);
    
    doInitialStuff(&state);

    while(true) {
        getInput(&state);
        if(state.shouldClose) {
            break;
        }

        glClearColor(0.0f, 0.0f, 0.0f, 1.0f);
        glClear(GL_COLOR_BUFFER_BIT);

        processInput(&state);

        swapBuffers(state);
    }

    return 0;
}
```

Given this example, it is fairly clear that there are two places where important per-frame logic is happening: `getInput()` and `processInput()`. However unless we have performance problems, it is likely that a large chunk of time is *not* spent in these important functions. In fact most of the time is probably being spent in `swapBuffers()`, waiting for the monitor to refresh. So we have plenty of time to check if we should swap out the code of those two functions for something newer.

## Using a dynamic library
In order to achieve hot reloading, we'll split our code into two complilations: an executable, and a DLL. With our main loop in `main.cpp` and our input handling code in `input.cpp`, compiling from the command line may look something like this:

```
cl /LD %CompileFlags% input.cpp /link %LinkFlags% /noimplib /noexp

cl %CompileFlags% main.cpp /link %LinkFlags% %LinkLibraries%
```

The idea is, if our executable is still running, the second complilation will fail, but the first will still succeed.

We'll want to **export** our input handling functions - this associates the name of the function with it's address, that way we can actually find our function addresses at runtime. We'll also want to avoid C++ name mangling by using `extern "C"` to tell the compiler to treat this function as if it is written in C:
```
extern "C" __declspec(dllexport) getInput(AppState* state) {
    // do stuff
}

extern "C" __declspec(dllexport) processInput(AppState* state) {
    // do stuff
}
```

We can then use the Win32 API calls `LoadLibraryA()`, `FreeLibrary()` and `GetProcAddress()` to reload our DLL and get the function pointers at the start of our loop:

```
...
typedef void HotReloadFunction(AppState*);
...
    HMODULE dynamicModule = LoadLibraryA("path\\to\\input.dll")
    while(true) {
        FreeLibrary(dynamicModule);
        dynamicModule = LoadLibraryA("path\\to\\input.dll");
        HotReloadFunction* getInput = getProcAddress(dynamicModule, "getInput");
        HotReloadFunction* processInput = getProcAddress(dynamicModule, "processInput");

        getInput(&state);
...
```

Now, at this point it works - every loop our DLL gets reloaded, but there are a couple of big problems.

The first is that since calling `LoadLibraryA()` takes a handle to `input.dll`, we can no longer overwrite it with a new DLL to load. We can fix this by copying the DLL, then loading the copy instead:

```
CopyFileA("path\\to\\input.dll", "path\\to\\input_temp.dll", false);
HMODULE dynamicModule = LoadLibrary("path\\to\\input_temp.dll")
```

The second problem is that we are going to take a pretty big performance hit if we reload the DLL every single frame. The solution to this is to do a much quicker check to know if we *should* reload the DLL. We can do this by keeping track of the last write time of the file - if it has changed, it is probably because there is a newer version of the DLL, so therefore we want to reload it.
```
FILETIME lastWriteTime = {};
bool shouldReload = false;
...
HANDLE dllFileHandle = CreateFileA("path\\to\\input_temp.dll", GENERIC_READ, FILE_SHARE_READ, 0, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, 0);
FILETIME newWriteTime;
bool gotFileTime = GetFileTime(dllFileHandle, 0, 0, &newWriteTime);
if(gotFileTime && lastWriteTime.DateTime != newWriteTime.DateTime) {
    lastWriteTime = newWriteTime;
    shouldReload = true;
}
```

Finally, we'll want to do good error checking and cleanup (the kind that these articles never show) and make sure we are setting the function pointers for `getInput()` and `processInput()` to stubs (empty versions of the functions) whenever we unload the DLL - that way even if something goes horribly wrong with loading the DLL and we still call the DLL functions, they won't cause crashes because they point to functions which don't exist.

Once code is pulled out into functions and structures, it might look a little like this:
```
...
    InputDLL inputDll = {};
    inputDll.path = "path\\to\\input.dll";

    loadInputDll(&inputDll);

    while(true) {
        if(inputDllHasChanged()) {
            unloadInputDll(&inputDll);
            loadInputDll(&inputDll);
        }

        inputDll.getInput(&state);
        if(state.shouldClose) {
            break;
        }

        glClearColor(0.0f, 0.0f, 0.0f, 1.0f);
        glClear(GL_COLOR_BUFFER_BIT);

        inputDll.processInput(&state);

        swapBuffers(state);
    }
...
```

## Some caveats
There is one catch to hot reloading via DLLs in this way: it doesn't scale well with lots of shared code between the two compilations. In this example, any functions in `main.cpp` that you would want to be called from `input.cpp` would require a function pointer passed from `main.cpp` to `input.cpp` (in this case, through the `AppState` struct). Likewise any global variables would have to be carefully passed between the two. It can also get tedious if there are many dynamic functions being called from `main.cpp`. Personally I don't find these problems affect me, since it encourages good code separation and API layers, but it may not be a good for some codebases. Also, it goes without saying that changes to `main.cpp` will still require a program restart to be made.

## Give it a go
It is hard to describe how much hot reloading has positively impacted my workflow, and how powerful it is to have code changes immediately appear in a running program. Given it doesn't take much work, I would strongly recommend giving it a go if you are writing native code, or to find out if it is possible in your language of choice.
