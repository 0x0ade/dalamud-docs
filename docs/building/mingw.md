---
sidebar_position: 1
---

# Building Dalamud with MinGW

:::warning

This guide acts as supplementary information to the main
[Building Dalamud](/building/) page, and thus refers to steps from that page.

Dalamud's native components are maintained via Visual Studio. Those components
don't change often, which means it's much easier to skip the native build steps,
follow the dotnet (C#) build-steps only, and re-use the existing built binaries
with no drawbacks.

**As such, this _will_ break from time to time.** Fixes are welcome though!

:::

Dalamud is built using [Nuke](https://nuke.build), which lacks built-in
first-class support for Linux-supporting build systems. Specifically, as Nuke
only manages kickoff using other tools such as MSBuild and dotnet build, it
relies on that tooling to declare those projects with all their source files and
props.

In the case of Dalamud, the C / C++ projects use
[MSBuild `.vcxproj` files](https://learn.microsoft.com/en-us/cpp/build/walkthrough-using-msbuild-to-create-a-visual-cpp-project?view=msvc-170).
Those are very Visual Studio centric: It's trivial to do basic edits to them in
any text editor, but it's non-trival to find what VS toggle maps to the XML
syntax without using Visual Studio yourself.

## Prerequisites

- The base prerequisites from the main build page, except Windows and VS
- An x64 Linux environment, e.g. a 64-bit Linux OS on x64 hardware
  - This is not a hard requirement, but this page is written with this setup in
    mind.
- CMake
- GCC
- MinGW x64 (`x86_64-w64-mingw32` by default)

Optionally, if you want to run the CI / Test target, set up a Wine prefix with
the following:

- [.NET 10.0 SDK for Windows x64](https://dotnet.microsoft.com/download/dotnet/10.0)
- [Git](https://git-scm.com/downloads)

:::warning

This section is work-in-progress. You can help by expanding it!

:::

## Building

Follow the main build steps, but instead of running the PowerShell script, run:

```bash
./build.sh
```

<sub>(Yes, even if you have PowerShell installed on Linux.)</sub>

:::tip

The Dalamud repository contains some basic VS Code configuration for debugging
various parts of the build process. You should be able to follow the
instructions on installing any missing extensions inside of it, and launch those
using the "Run and Debug" tab.

Feel free to adjust those commands to your needs when testing locally.

:::

## Running

The build process will output the injector to `bin/Debug/Dalamud.Injector.exe`.
For testing purposes, you can follow the main page steps, or use the "Use Local
Dalamud" option in the XIVLauncher.Core settings.

![preview](images/xlcore-local-dalamud.png)

## ... but how?

### Why not move to &lt;insert build system here&gt;?

Originally, building Dalamud with all its components required a full Windows +
Visual Studio setup, as the native components were heavily reliant on MSVC
compilation behavior. Even now, nearly all Dalamud development is happening on
Windows + Visual Studio. As such, a crazy member of the community though:

> I bet I can build this without moving away from .vcxproj and NUKE!

NUKE build recipes have full access to the
[MSBuild parser](https://github.com/dotnet/msbuild), which is written in C#,
open-sourced, MIT-licensed, available on NuGet, and used by NUKE itself. As
such, nothing stops us from parsing said .vcxproj files (or at least the subset
we need), generating CMakeLists out of those, and invoking the necessary CMake
commands to configure and build the project via NUKE.

### Quirks

MinGW does not support `#pragma comment` and `__try / __except` among other
things. Furthermore, header and library names are lowercase, and some parts of
std aren't being imported transitively as with MSVC, or don't support UTF-16
strings in the exact same way as MSVC.

Some are trivial to work around, such as `#pragma comment` and header
mismatches, while others require a bit more insight on how e.g.
`__try / __except` is used for SEH and doesn't map to normal C++ exception
handling by default.

Even if you don't have a Linux build environment, CI will flag most issues.

### Extensions

When the .vcxproj files get parsed, the property `IsMinGW` is set to `true`.
**Don't** check for `Condition="$(IsMinGW)"`, as it defaults to an empty string,
which MSBuild will error out on. **Instead**, check `'$(IsMinGW)'=='true'` where
necessary.

`VCProjToCMakeLists` does not implement MSBuild targets. Any post-build steps
should be defined using the `CMakePostBuild` property with standard CMake
installation syntax instead.
