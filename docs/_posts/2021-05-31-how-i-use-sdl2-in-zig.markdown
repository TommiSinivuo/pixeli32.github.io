---
layout: post
title:  "How I Use SDL2 in Zig - a Brief Overview"
date:   2021-05-31 18:30:00 +0300
categories: zig sdl2
---
I'm currently working on a small puzzle game, which is inspired by the
mechanics of the classic Boulder Dash and other games like it. I'm writing it
in [the Zig programming language](https://ziglang.org) and I'm using
[SDL2](http://libsdl.org) to make things happen. SDL is currently responsible
for creating a window, querying input state, sound output, file I/O and
rendering. This post is about me explaining how I interface with the SDL2 API
from my Zig code. The method is something that works for me, at this point in
time, and it's pretty simplistic.

## Building with SDL2

I'm using the Zig build system to build the game. That means that I have a
`build.zig` file defining what and how to build. This is what it looks like:

```
const Builder = @import("std").build.Builder;

pub fn build(b: *Builder) void {
    const target = b.standardTargetOptions(.{});
    const mode = b.standardReleaseOptions();

    const game_exe = b.addExecutable("game01", "src/main.zig");
    game_exe.setTarget(target);
    game_exe.setBuildMode(mode);
    game_exe.linkLibC();
    game_exe.addIncludeDir("C:/SDL2/include");
    game_exe.addLibPath("C:/SDL2/lib/x64");
    game_exe.linkSystemLibrary("SDL2");
    game_exe.install();

    const game_run_cmd = game_exe.run();
    game_run_cmd.step.dependOn(b.getInstallStep());
    if (b.args) |args| {
        game_run_cmd.addArgs(args);
    }

    const game_run_step = b.step("run", "Run the game");
    game_run_step.dependOn(&game_run_cmd.step);
}
```

First of all, I'm linking libc, because SDL2 depends on that. Then, I add SDL2
to the mix by introducing it's include and lib directories followed by linking
it as a system library.

As you can see, I'm currently working on Windows. I have downloaded the MSVC
development libraries (currently `SDL2-devel-2.0.14-VC.zip`) found at the
download page and simply extracted the contents to `C:\SDL2`, which is decent
enough for a one-person team.

This can be run using the following simple command:
```
zig build run
```

## Wrapping SDL2

In Zig, you don't _really_ need "Zig bindings" for a C library, since the
library can simply be imported like this:

```
const c = @cImport({
    @cInclude("SDL.h");
});
```

After that you can call any C function like this:

```
c.SDL_NumJoysticks();
```

This is very convenient and it shows off how well Zig embraces existing C code
and libraries. However, this has some downsides on scale, since you have to
handle errors, cast Zig types to C types to be able to call functions etc. In
other words, it can become a little messy. After a little bit of thought, I 
came up with exactly two requirements for my own interface to SDL:

- functions should return Zig errors when they fail instead of values
- functions should internally handle casting to C types when needed

That's it. As you can see, these requirements rule out using the C functions
directly, so I need some kind of abstraction to take care of my requirements.

My approach to this problem is rather simplistic. I have a file
called `sdl.zig`, where I have defined very simple wrappers for the SDL2
functions, types, constants and enumerations that I need in my game.

This is the simplest form of a function wrapper:

```
pub fn pollEvent(event: *Event) i32 {
    return c.SDL_PollEvent(event);
}
```

As you can see, it's basically a 1:1 mapping to the actual SDL2 function. If
you are wondering what `Event` is, it´s defined like this in `sdl.zig`:

```
pub const Event = c.SDL_Event;
```

In fact, I've exposed all the other SDL structs that I use the same way, too.

However, many SDL2 functions can fail under certain circumstances. This is
where it gets a little bit more complicated, because the functions don't
(and can't) have a perfectly unified way of indicating when an error occurs.
You have to read the documentation of a function to find out whether an error
state is indicated by a return value being less than 0, exactly 0, NULL or
something else.

This is a pretty common example:

```
pub fn renderClear(renderer: *Renderer) !void {
    if (c.SDL_RenderClear(renderer) < 0) {
        return error.RenderError;
    }
}
```

The `SDL_RenderClear` function returns a value less than zero when an error
occurs. If that happens, I just return an error. Currently I have defined all
my errors in the `SdlError` error union, which looks like this:

```
pub const SdlError = error{
    InitError,
    CreateWindowError,
    CreateRendererError,
    CreateTextureError,
    CursorError,
    GameControllerError,
    GetWindowIdError,
    JoystickError,
    LoadBmpError,
    LoadWavError,
    OpenAudioDeviceError,
    QueueAudioError,
    QueryTextureError,
    RenderSetLogicalSizeError,
    RenderError,
    RwError,
    RwCloseError,
    RwWriteError,
    SetColorKeyError,
};
```

Here's another type of error case:

```
pub fn loadBmp(file: []const u8) !*Surface {
    if (c.SDL_LoadBMP(@ptrCast([*c]const u8, file))) |surface| {
        return surface;
    } else {
        return error.LoadBmpError;
    }
}
```

`SDL_LoadBMP` returns a pointer to an `SDL_Surface` if it succeeds or `NULL`
if it fails. This can intuitively be handled as an optional pointer in Zig,
so we can use a simple `if else` clause to either return the pointer or an
error if the pointer is `null`. Also note that I convert a `[]const u8` to a
`[*c]const u8`, because that's what the C function is expecting. This way I can
call this function without having to worry about casting the string to a C
type. I try to do these kinds of casts inside the wrapper functions to keep the
client code clean.

Working with enumerations is a little bit awkward at points. I have exposed
some of the SDL enumerations like this:

```
pub const AudioFormat = c.SDL_AudioFormat;
pub const GameControllerButton = c.SDL_GameControllerButton;
```

This is required when I use SDL functions that take an enum type as parameter,
for example:

```
pub fn gameControllerGetButton(
    game_controller: *GameController,
    button: GameControllerButton,
) u8 {
    return c.SDL_GameControllerGetButton(game_controller, button);
}
```
Here `GameControllerButton` is the enum type.

However, there are many enumerations that I've just defined as public
constants because they are way simpler to use that way, like the scancodes for
example:

```
...

pub const SCANCODE_DOWN = c.SDL_SCANCODE_DOWN;
pub const SCANCODE_LEFT = c.SDL_SCANCODE_LEFT;
pub const SCANCODE_RIGHT = c.SDL_SCANCODE_RIGHT;
pub const SCANCODE_UP = c.SDL_SCANCODE_UP;

...
```

Here's an example of when things get a little messy:
```
pub fn isGameController(joystick_index: i32) bool {
    return @enumToInt(c.SDL_IsGameController(joystick_index)) == c.SDL_TRUE;
}
```

`SDL_IsGameController` returns a value of the `SDL_bool` enum type, either
`SDL_TRUE` (1) or `SDL_FALSE` (0), but if I try to compare that value directly
to `SDL_TRUE`, then I will get a compilation error. That's because the return
value is of type `SDL_bool` and the enum value `SDL_TRUE` is an integer. I have
to cast the returned value to an integer first to make that comparison work.
I'm glad I can hide it inside the wrapper function and just return a `bool`.

## Conclusion

That's basically  it! So, why don't I just use existing SDL2 bindings instead of wrapping
it myself? Well, I'm using a pretty limited subset of SDL, so I would say that
it simply makes sense for me to wrap only the stuff that I need, exactly the
way I want to. It's working well for me and if I suddenly think something's
awkward or weird, I can just change it right away.

Thanks for reading!
