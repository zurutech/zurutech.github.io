---
author: "pgaleone"
layout: post
title:  "Code Coverage of Unreal Engine projects"
slug:  "Code Coverage of Unreal Engine projects"
date:   2022-02-17 8:00:00
categories: coding
image: images/unreal-coverage/cov-head.png
tags: coding
description: "Dive into the Unreal Build Tool (UBT) - in particular in the Linux Tool Chain - for understand what has to be modified to add the support for the code coverage. We’ll show how to correctly use the lcov tool for generating the code coverage report."
---

Code coverage is a widely used metric that measures the percentage of lines of code covered by automated tests. Unreal Engine doesn't come with out-of-the-box support for computing this metric, although it provides a quite good testing suite.

In this article, we dive into the Unreal Build Tool (UBT) - particularly in the Linux Tool Chain - to understand what has to be modified to add the support, UBT-side, for the code coverage. Moreover, we'll show how to correctly use the `lcov` tool for generating the code coverage report.

The article will also focus on the compiler-side support for the code coverage; this focus will help us understand why on our first attempt the code coverage hasn't been generated and, thus, finding a subtle bug in the Automation System "quit" command.

## The Unreal Build Tool

The [Unreal Build Tool (UBT)](https://docs.unrealengine.com/4.27/en-US/ProductionPipelines/BuildTools/UnrealBuildTool/) is - more or less - the Unreal equivalent of [CMake](https://cmake.org/). It allows us to define modules, compilation targets, setting attributes to these targets, define module dependencies, compile and link them. Unreal Engine itself uses the UBT to build its modules.

Every well-written unreal project is composed of several modules. A module is identified by a `.Build.cs` file that controls how it's built, its dependencies, and what to expose publicly (when used as a dependency) and privately. These `Build.cs` files must be under the `Source` folder. Any unreal project has at least one target.

A target identifies the type of project we are building. Targets are declared through C# source files with a `.Target.cs` extension, and are stored under the project's `Source` directory. UBT supports building several target types. Among that, there's the "Editor" target that's the target we are interested in. We need to build the editor target because the Automation Test suite executes the tests using editor features, hence, inside the editor.

Independently by the chosen target type (Editor, Program, Client, Server, Game) we can use several [read/write properties](https://docs.unrealengine.com/4.27/en-US/ProductionPipelines/BuildTools/UnrealBuildTool/TargetFiles/#read/writeproperties) we can use to customize the building process. These properties are the **rules** to follow during the compilation of the target and, thus, of the dependent modules.

As an example, we have high-level flags like `bUseChaos` that allow us to enable/disable engine features like Chaos (the Unreal physics plugin). Still, we also have low-level flags like `bAllowLTCG` to allow the use of link-time code generation (LTCG).

The Unreal Build Tool uses the Target configuration to generate the compilation command to execute when building every source file of every module used by the target. Hence, we can think about the Target as a way for customizing the building process and the compilation command.

Understanding this is extremely important since the code-coverage support is both a compile-time and run-time support completely provided by the **compiler**.

## Code coverage on the Unreal Build Tool

The code coverage workflow consists of four steps:

1. **Compiling** with coverage enabled. Every compiler provides some dedicated flags. Moreover, optimizations must be disabled because the coverage report should correctly track the number of times every LOC (Line Of Code) has been executed.
2. **Linking** the executable/library with the compiler-provided profile library (on LLVM is the [compiler-rt](https://compiler-rt.llvm.org/) runtime library, in particular, the "profile" module).
3. **Running** the executable (or load the compiled library and use it)
4. **Creating coverage reports**

Using Unreal Engine, the first 2 points require some modifications to the UBT. UBT is a cross-platform build tool that has its platform-specific implementations and toolchains. This article only focuses on the Linux toolchain, but the reasoning here presented works for all the other toolchains and operating systems as well.

As presented in the [previous section](#the-unreal-build-tool) every target has a set of read/write properties called **rules**. These rules touch both the compilation and the linking phase of the target (and thus, of the modules compiled in the target). Some of these rules can be set not only from the `.Target.cs` file, but also via CLI arguments.

Our goal is to create a custom target (and module) rule that can also be set via CLI. This rule will allow us to compile a project with optimization disabled, with the correct compilation flags, and link it with the correct runtime libraries.

### Customizing the UBT: -CodeCoverage flag

Here's how we want to invoke the Unreal Build Tool:

```bash
mono Engine/Binaries/DotNET/UnrealBuildTool.exe \ # the UBT
         TargetNameEditor \ # Our compilation target, in Editor
         Linux \ # Our target platfrom
         Development \
         -project=Project.uproject \ # The unreal project we are building
         -CodeCoverage # The custom flag we are going to add
```

The invocation is standard where the only difference is the `-CodeCoverage` flag. That's what we are going to add.

The UBT source code is organized in the following way:

- `Configuration`: this folder contains the source code for the configuration of the building process. Therein we find the configuration of the targets (Target Rules), modules (Module Rules), and the dependent configuration of the Compilation Environment and Linking Environment. These environments are nothing but classes with all the compilation and linking configuration set that the per-platform toolchains use to create the compilation/linking commands.
- `Platform`: this folder contains the platform-specific toolchains. In this context, a toolchain is an alias for toolchain-manager. Usually, a toolchain is the set of tools we need for compiling/linking a program. Unreal ships with the engine itself an (old :() version of clang and the LLVM tools: this is the real toolchain. In the context of UBT, the toolchain is the C# source code that creates the real toolchain invocation, depending on the rules. (e.g. it creates the `clang filename.cpp -option1 -option2` invocation, where `option1` and `option2` are the results of the processing of the target and module rules.
- `System`: this folder contains the platform-agnostic configuration, e.g., only the compilation/linking environments definitions but not their usage.

The execution flow is presented in the following diagram

```text
       +--------------+  +--------------------+  +--------------------+  +---------------------+
       |              |  |                    |  |                    |  |                     |
(CLI)->|Target Config |->| Compile Env Config |->| Platform Toolchain |->| Compilation Command |
       |              |  |                    |  |                    |  |                     |
       +--------------+  +--------------------+  +--------------------+  +---------------------+
```

The first modification is the addition of the `-CodeCoverage` flag in the `TargetRules.cs` file.


```diff
diff --git a/Engine/Source/Programs/UnrealBuildTool/Configuration/TargetRules.cs b/Engine/Source/Programs/UnrealBuildTool/Configuration/TargetRules.cs
index b3dac4efa6c..e0b6e130e9c 100644
--- a/Engine/Source/Programs/UnrealBuildTool/Configuration/TargetRules.cs
+++ b/Engine/Source/Programs/UnrealBuildTool/Configuration/TargetRules.cs
@@ -1083,6 +1083,13 @@ namespace UnrealBuildTool
                [XmlConfigFile(Category = "BuildConfiguration")]
                public bool bPGOOptimize = false;

+               /// <summary>
+               /// Whether the target requires code coverage compilation and linking.
+               /// </summary>
+               [CommandLine("-CodeCoverage", Value = "true")]
+               [XmlConfigFile(Category = "BuildConfiguration")]
+               public bool bCodeCoverage;
+
                /// <summary>
                /// Whether to support edit and continue.  Only works on Microsoft compilers.
                /// </summary>
@@ -2493,6 +2500,11 @@ namespace UnrealBuildTool
                        get { return Inner.bPGOOptimize; }
                }

+               public bool bCodeCoverage
+               {
+                       get {return Inner.bCodeCoverage; }
+               }
+
                public bool bSupportEditAndContinue
                {
                        get { return Inner.bSupportEditAndContinue; }
```

Now, we need to extend the compilation and linking environments to be aware of the code coverage flag. As anticipated, these environments are in the `System` folder, and the changes are just the addition of the attribute `bCodeCoverage` and the handling in the copy constructor.

```diff
diff --git a/Engine/Source/Programs/UnrealBuildTool/System/CppCompileEnvironment.cs b/Engine/Source/Programs/UnrealBuildTool/System/CppCompileEnvironment.cs
index 189954552a3..f56830c64d2 100644
--- a/Engine/Source/Programs/UnrealBuildTool/System/CppCompileEnvironment.cs
+++ b/Engine/Source/Programs/UnrealBuildTool/System/CppCompileEnvironment.cs
@@ -221,6 +221,11 @@ namespace UnrealBuildTool
                /// </summary>
                public bool bOptimizeCode = false;

+               /// <summary>
+               /// True if the compilation should produce tracing output for code coverage.
+               /// </summary>
+               public bool bCodeCoverage = false;
+
                /// <summary>
                /// Whether to optimize for minimal code size
                /// </summary>
@@ -428,6 +433,7 @@ namespace UnrealBuildTool
                        bUndefinedIdentifierWarningsAsErrors = Other.bUndefinedIdentifierWarningsAsErrors;
                        bEnableUndefinedIdentifierWarnings = Other.bEnableUndefinedIdentifierWarnings;
                        bOptimizeCode = Other.bOptimizeCode;
+                       bCodeCoverage = Other.bCodeCoverage;
                        bOptimizeForSize = Other.bOptimizeForSize;
                        bCreateDebugInfo = Other.bCreateDebugInfo;
                        bIsBuildingLibrary = Other.bIsBuildingLibrary;
diff --git a/Engine/Source/Programs/UnrealBuildTool/System/LinkEnvironment.cs b/Engine/Source/Programs/UnrealBuildTool/System/LinkEnvironment.cs
index 610e4b3db4d..9a94a6b4388 100644
--- a/Engine/Source/Programs/UnrealBuildTool/System/LinkEnvironment.cs
+++ b/Engine/Source/Programs/UnrealBuildTool/System/LinkEnvironment.cs
@@ -196,6 +196,11 @@ namespace UnrealBuildTool
                /// </summary>
                public bool bOptimizeForSize = false;

+               /// <summary>
+               /// Wether to link code coverage / tracing libs
+               /// </summary>
+               public bool bCodeCoverage = false;
+
                /// <summary>
                /// Whether to omit frame pointers or not. Disabling is useful for e.g. memory profiling on the PC
                /// </summary>
@@ -349,6 +354,7 @@ namespace UnrealBuildTool
                        DefaultStackSize = Other.DefaultStackSize;
                        DefaultStackSizeCommit = Other.DefaultStackSizeCommit;
                        bOptimizeForSize = Other.bOptimizeForSize;
+                       bCodeCoverage = Other.bCodeCoverage;
                        bOmitFramePointers = Other.bOmitFramePointers;
                        bSupportEditAndContinue = Other.bSupportEditAndContinue;
                        bUseIncrementalLinking = Other.bUseIncrementalLinking;
```

So far so good. We have the target configuration and the environments set up. We can now start to change the build behavior of the modules when the code coverage flag is passed. Several other parts require the "propagation" of the code coverage flag but are not reported in this article for brevity. The complete patch is, anyway, available as a [Github Gist](https://gist.github.com/galeone/f8bdf0fb4fafc517a4f65537b2ae2634).

The most important part of the [patch](https://gist.github.com/galeone/f8bdf0fb4fafc517a4f65537b2ae2634) is the modification of the Linux Toolchain, which effectively generates the compilation command to execute when the `-CodeCoverage` flag is passed.

```diff
diff --git a/Engine/Source/Programs/UnrealBuildTool/Platform/Linux/LinuxToolChain.cs b/Engine/Source/Programs/UnrealBuildTool/Platform/Linux/LinuxToolChain.cs
index fb38ffe34fe..ba6b28f48d0 100644
--- a/Engine/Source/Programs/UnrealBuildTool/Platform/Linux/LinuxToolChain.cs
+++ b/Engine/Source/Programs/UnrealBuildTool/Platform/Linux/LinuxToolChain.cs
@@ -170,6 +170,7 @@ namespace UnrealBuildTool
 				bIsCrossCompiling = true;
 
 				bHasValidCompiler = DetermineCompilerVersion();
+				CompilerRTPath = Path.Combine(Path.Combine(BaseLinuxPath, String.Format("lib/clang/{0}/lib/linux/", CompilerVersionString)));
 			}
 
 			if (!bHasValidCompiler)
@@ -767,8 +768,13 @@ namespace UnrealBuildTool
 				}
 			}
 
-			// optimization level
-			if (!CompileEnvironment.bOptimizeCode)
+			if (CompileEnvironment.bCodeCoverage)
+			{
+				Result += " -O0";
+				Result += " -fprofile-arcs -ftest-coverage"; // gcov
+				//Result += " -fprofile-instr-generate -fcoverage-mapping"; // llvm-cov
+			}
+			else if (!CompileEnvironment.bOptimizeCode) // optimization level
 			{
 				Result += " -O0";
 			}
@@ -1019,6 +1025,15 @@ namespace UnrealBuildTool
 				Result += " -Wl,--gdb-index";
 			}
 
+			if (LinkEnvironment.bCodeCoverage)
+			{
+				// Unreal Separates the linking phase and the compilation phase.
+				// We pass to clang the flag `--coverage` during the compile time
+				// And we link the correct compiler-rt library (shipped by UE, and part of the LLVM toolchain)
+				// to every binary produced.
+				Result += string.Format(" -L{0} -l{1}", CompilerRTPath, "clang_rt.profile-x86_64"); // gcov
+				// Result += " -fprofile-instr-generate"; // llvm-cov
+			}
 			// RPATH for third party libs
 			Result += " -Wl,-rpath=${ORIGIN}";
 			Result += " -Wl,-rpath-link=${ORIGIN}";
@@ -1142,6 +1157,7 @@ namespace UnrealBuildTool
 		protected string BaseLinuxPath;
 		protected string ClangPath;
 		protected string GCCPath;
+		protected string CompilerRTPath;
 		protected string ArPath;
 		protected string LlvmArPath;
 		protected string RanlibPath;
@@ -1270,6 +1286,11 @@ namespace UnrealBuildTool
 				Log.TraceInformation("  Prefix for PGO data files='{0}'", CompileEnvironment.PGOFilenamePrefix);
 			}
 
+			if (CompileEnvironment.bCodeCoverage)
+			{
+				Log.TraceInformation("Using --coverage build flag");
+			}
+
 			if (CompileEnvironment.bPGOProfile)
 			{
 				Log.TraceInformation("Using PGI (profile guided instrumentation).");
```

There are two important parts in this patch. The first part is relative to the compilation flag

```c#
Result += " -O0";
Result += " -fprofile-arcs -ftest-coverage"; // gcov
//Result += " -fprofile-instr-generate -fcoverage-mapping"; // llvm-cov
```

where we are constructing the build command passing the flags:

- `-O0`: for disabling the optimizations
- `-fprofile-arcs`: *add code so that program flow arcs are instrumented. During execution the program records how many times each branch and call is executed and how many times it is taken or returns. On targets that support constructors with priority support, profiling properly handles constructors, destructors and C++ constructors (and destructors) of classes which are used as a type of a global variable.
When the compiled program exits it saves this data to a file called auxname.gcda for each source file. The data may be used for profile-directed optimizations (`-fbranch-probabilities`), or for test coverage analysis (`-ftest-coverage`). Each object file’s auxname is generated from the name of the output file, if explicitly specified and it is not the final executable, otherwise it is the basename of the source file. In both cases any suffix is removed (e.g. foo.gcda for input file dir/foo.c, or dir/foo.gcda for output file specified as -o dir/foo.o).* [^1]
- `-ftest-coverage`: for generating a notes file that the `gcov` code-coverage utility can use to show program coverage. The auxname.gcno files.

Depending on the code-coverage utility chosen for the report, a different set of compile-time flag can be passed. The chosen tool is `gcov` (and its graphical frontend `lcov`), but if the desired tool is `llvm-cov profile` then the commented flags have to be passed.

[llvm-cov](https://llvm.org/docs/CommandGuide/llvm-cov.html) is a tool that supports different commands. Depending on the command passed as first argument its behavior changes. In this article, we don't use `llvm-cov` to show/export/report the coverage information, but only in its `llvm-cov gcov` configuration. When used in this way, `llvm-cov gcov` is a tool for reading coverage data files and display coverage information compatible with `gcov`.

The second important part is the linker flag. Unreal separates the compilation phase from the linking phase (usually, clang/gcc create and execute the linker commands for us, while unreal completely separates these steps), and for this reason, we need to pass the correct flag to the linker.

```patch
if (LinkEnvironment.bCodeCoverage)
{
	// Unreal Separates the linking phase and the compilation phase.
	// We pass to clang the flag `--coverage` during the compile time
	// And we link the correct compiler-rt library (shipped by UE, and part of the LLVM toolchain)
	// to every binary produced.
	Result += string.Format(" -L{0} -l{1}", CompilerRTPath, "clang_rt.profile-x86_64"); // gcov
	// Result += " -fprofile-instr-generate"; // llvm-cov
}
```

Using `gcov` the linker flag could have been `--coverage` (the same flag could have been used instead of the two separate flags shown before), but in practice, the flag `--coverage` become `-lgcov` during linking. That linker flag is nothing but the link of the profile module of the LLVM compiler-rt runtime library (because we are using clang as compiler), so I made it explicit (for my future self).

I decided to use `gcov` and `lcov` because I find them easier to use respect to `llvm-cov` and, moreover, it's a widely used format that it's compatible with online services like [coveralls](https://coveralls.io/) and [codecov](https://about.codecov.io/). Anyway, we'll use `llvm-cov` as a gcov-compatible tool (using the `gcov-tool` flag of `lcov`), and because unreal provides an old version of the LLVM toolchain and we need to be compatible with it.

Once again, the complete patch code is available as a [Github Gist](https://gist.github.com/galeone/f8bdf0fb4fafc517a4f65537b2ae2634).

Here we go! Applying [this patch](https://gist.github.com/galeone/f8bdf0fb4fafc517a4f65537b2ae2634) to the engine source code, and re-compiling the UBT we have the `-CodeCoverage` flag that will instrument our built program to generate the `.gcdo` files when compiled and the `.gcda` files when *correctly executed*.

## Measuring the coverage

Now that our UBT has the `-CodeCoverage` flag, we can generate our instrumented program (library) and run the tests. As presented in my previous article [GitLab CI/CD for cross-platform Unreal Engine 4 projects](/coding/2020/09/29/gitlab-ci-cd-for-cross-platform-unreal-engine-4-projects#tests), it's straightforward executing tests via CLI, and also integrating them in a CI pipeline.

Hence, for measuring the code coverage we need to:

1. Compile our unreal project, using the UBT and passing the `-CodeCoverage`. Our project will be compiled in several shared objects (libraries) the Unreal Editor will load.
2. Run the Unreal Editor (making it load our `.so` files), and execute the tests.
3. Verify that for every `.cgno` file, there's a corresponding `.gcda` file containing the coverage information.
4. Use a graphical frontend to gcov (we'll use `lcov`) to generate the coverage report.

The first point it's precisely the invocation of the build command presented in the previous section:

```bash
mono Engine/Binaries/DotNET/UnrealBuildTool.exe \ # the UBT
         TargetNameEditor \ # Our compilation target, in Editor
         Linux \ # Our target platfrom
         Development \
         -project=Project.uproject \ # The unreal project we are building
         -CodeCoverage # The custom flag we are going to add
```

Supposing that our project contains some tests written using the Unreal Automation Testing suite, we can run the tests (without GUI) as presented in the [previous article](/coding/2020/09/29/gitlab-ci-cd-for-cross-platform-unreal-engine-4-projects#tests):

```bash
Engine/Binaries/Linux/UE4Editor Project.uproject \
        -ExecCmds="automation RunTests Now MODULE+TO+TEST+PLUS+SEPARATED; quit" \
        -buildmachine -forcelogflush -unattended -nopause -nosplash -log -nullrhi -stdout -FullStdOutLogOutput
```

The important part is the `-ExecCmds="automation RunTests Now MODULE+TO+TEST+PLUS+SEPARATED; quit"` flag, that:

- instructs the Editor to invoke the `automation` suite for
- running the tests (`RunTests`)
- as soon as a worker is available (`Now`)
- of all the tests matching one of the plus-separated strings (`MODULE+TO+TEST+PLUS+SEPARATED`)
- Once all of them have been executed (`;`), invoke the automation "quit" command. **IMPORTANT**.

The quit command is extremely important because the automation module uses it to shut down the editor itself and, as we'll see soon, how we close the application interacts with the generations of the `.gcda` files and, thus, with the ability to measure the coverage.

Unfortunately, when running this command, the tests get executed correctly, but in the `Intermediates` folder, we'll find **only** the `.gcno` files generated during the compilation. For example:

```
Intermediate/Build/Linux/B4D820EA/UE4Editor/Development/FSM/StateMachine.cpp.gcno
Intermediate/Build/Linux/B4D820EA/UE4Editor/Development/FSM/FSMModule.cpp.gcno
```

This is strange since we correctly built our instrumented executable and also correctly linked it to the correct module of the compiler-rt. So why does this happen?

### Forced Shutdown

The key to the problem resides in the "quit" command. In fact, the generation of the `.gcda` files happens at the end of the execution - of **successful** execution. By successful execution, we intend a normal program execution that allows the C++ runtime to call all the destructors, handle all the termination routines, and correctly terminate.

By digging into the Automation Commands code (`Engine/Source/Developer/AutomationController/Private/AutomationCommandline.cpp`), we can see what happens when the "quit" command is executed:

```cpp
UE_LOG(LogAutomationCommandLine, Display, TEXT("**** TEST COMPLETE. EXIT CODE: %d ****"), GIsCriticalError ? -1 : 0);
FPlatformMisc::RequestExitWithStatus(true, GIsCriticalError ? -1 : 0);
```

The first parameter of `LogAutomationCommandLine` is `bForce`, that in this case is `true`. When `bForce` is `true`, the `FGenericPlatformMisc::RequestExit(bool Force)` method is called, whose body is presented below.

```cpp
void FGenericPlatformMisc::RequestExit( bool Force )
{
    UE_LOG(LogGenericPlatformMisc, Log,  TEXT("FPlatformMisc::RequestExit(%i)"), Force );
    if( Force )
    {
        // Force immediate exit.
        // Dangerous because config code isn't flushed, global destructors aren't called, etc.
        // Suppress abort message and MS reports.
        abort();
    }
    else
    {
        // Tell the platform specific code we want to exit cleanly from the main loop.
        RequestEngineExit(TEXT("GenericPlatform RequestExit"));
    }
}
```

It's clear that calling `abort()` prevents a successful shut down and, thus, it prevents the correct generation of the `.gcda` files. Therefore, there's another small patch to apply to the engine source code, that changes the behavior of the "quit" command.

```patch
diff --git a/Engine/Source/Developer/AutomationController/Private/AutomationCommandline.cpp b/Engine/Source/Developer/AutomationController/Private/AutomationCommandline.cpp
index 62aeed07439..024c7aa3558 100644
--- a/Engine/Source/Developer/AutomationController/Private/AutomationCommandline.cpp
+++ b/Engine/Source/Developer/AutomationController/Private/AutomationCommandline.cpp
@@ -488,7 +488,7 @@ if (bMeetsMatch)
                                        }
                                        UE_LOG(LogAutomationCommandLine, Log, TEXT("Shutting down. GIsCriticalError=%d"), GIsCriticalError);
                                        UE_LOG(LogAutomationCommandLine, Display, TEXT("**** TEST COMPLETE. EXIT CODE: %d ****"), GIsCriticalError ? -1 : 0);
-                                       FPlatformMisc::RequestExitWithStatus(true, GIsCriticalError ? -1 : 0);
+                                       FPlatformMisc::RequestExitWithStatus(false, GIsCriticalError ? -1 : 0);
                                        AutomationTestState = EAutomationTestState::Complete;
                                }
                                break;
```

In this way, we don't force an abnormal exit and the `.gcda` files can be correctly generated.

After applying this patch - recompiled the engine `AutomationController` module, re-built (with `-CodeCoverage`), and re-ran the tests, we end up with the `.gcda` files next to the `.gcno` files, e.g.:

```
Intermediate/Build/Linux/B4D820EA/UE4Editor/Development/FSM/StateMachine.cpp.gcno
Intermediate/Build/Linux/B4D820EA/UE4Editor/Development/FSM/StateMachine.cpp.gcda

Intermediate/Build/Linux/B4D820EA/UE4Editor/Development/FSM/FSMModule.cpp.gcno
Intermediate/Build/Linux/B4D820EA/UE4Editor/Development/FSM/FSMModule.cpp.gcda
```

Bingo!

## Code Coverage Report

On Linux, unreal compiles the source code with the clang+llvm toolchain that itself provides. Unfortunately, this is an old version (in Unreal 4.27 the clang version is 11.0.1 while, at the time of writing, the latest stable release is the 13.0). Using an old version for generating the `.gcda` and `.gcno` files means that we cannot use a newer version of `lcov` or `llvm-cov gcov` for creating a summary, because there's no compatibility among different versions.

Moreover, the toolchain provided by unreal does not come with a compiled version of `llvm-cov`, thus we need to compile it by ourselves. We could also download a pre-built version, but depending on the glibc version used in the OS it might be compatible or not. Therefore, compiling the `clang-tools-extra` (the module of the LLVM project that contains `llvm-cov`) is the best option we have for being sure that the tool will work on our OS.

```bash
wget https://github.com/llvm/llvm-project/archive/refs/tags/llvmorg-11.0.1.tar.gz
tar xf llvmorg-11.0.1.tar.gz
cd llvm-project-*
mkdir build
cd build
cmake -DLLVM_ENABLE_PROJECTS=clang-tools-extra -DCMAKE_BUILD_TYPE=Release -G "Unix Makefiles" ../llvm
make -j$(nproc)
```

Neat! Now, using `lcov` we can specify  the `gcov-tool` to use for parsing the coverage files and generating the reports. The `llvm-cov` tool is in the `build/bin/` dir (or wherever you decide to install it).

First, we need to wrap the `llvmm-cov gcov` execution in a (executable) bash script for being compatible with the expected `lcov` format:

```bash
# Create llvm-gcov.sh with the following content
#!/bin/bash
exec llvm-cov gcov "$@"

# Make it executable
chmod +x llvm-gcov.sh
```

We can now use `lcov` together with `gcov-tool` for creating a `coverage.info` file that contains only the coverage of our source code removing, thus, all the references to the engine code or plugin code.


```bash
# $engine contains the path of the engine

lcov -q --gcov-tool llvm-gcov.sh --directory . --capture --no-external --base-directory $engine/Engine/Source/ -o cov.info
lcov --remove cov.info '/usr/*' \
     --remove cov.info "$engine"'/Engine/Source/*' \
     --remove cov.info $(pwd)'/Plugins/*' \
     --remove cov.info $(pwd)'/Intermediate/*' \
     --remove cov.info $(pwd)'/Source/*/ThirdParty/*' \
     --output-file cov.info

# Now cov.info has all the information aboout the prevous execution, without any reference to:
# - engine files
# - system files
# - Plugins
# - generated files
# - ThirdParty libraries included in some of our modules
```

The `lcov` package comes with a set of Perl scripts that parse the `.info` file and generate reports and statistics. Since we are interested in the HTML report, we can use `genhtml`:

```
genhtml cov.info -o coverage
```

The `coverage` folder contains the complete coverage report, and thus we can point our preferred browser to `coverage/index.html` for getting the coverage report.

<div class="blog-image-container" markdown="1">
![coverage report](/images/unreal-coverage/cov-long.png){:class="blog-image"}
</div>

## Conclusions

The Unreal Build Tool is a (not easily) extendible cross-platform build tool. This tool doesn't come with code-coverage support out of the box, and it requires modification of every toolchain for making it work. This article focused on the modification of the Linux Toolchain, because the Linux ecosystem gives us an easy to use set of tools for measuring the coverage and, moreover, using Linux on the CI/CD pipeline it's natural to focus on the same platform we use for running the tests.

The `Quit` command of the automation suite forces the shut down (`abort()`) preventing a correct generation of the coverage files, and for this reason, the engine code - not only the Linux toolchain - has to be patched.

This article showed how it is possible to integrate the code coverage when running unit tests without a GUI: the `Quit` command correctly closes the editor only in this scenario. When running GUI-based tests, the Quit command should force the exit (with an abort!), otherwise, the automation command isn't able to close the game and the Editor.

---

[^1]: [https://gcc.gnu.org/onlinedocs/gcc-10.1.0/gcc/Instrumentation-Options.html](3.12 Program Instrumentation Options)
