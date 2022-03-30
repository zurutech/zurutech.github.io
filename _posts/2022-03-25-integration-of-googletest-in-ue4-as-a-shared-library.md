---
author: stefano-strati
layout: post
title: "Integration of GoogleTest in UE4 as a Shared Library"
slug:  "Integration of GoogleTest in UE4 as a Shared Library"
date:   2022-03-30 8:00:00
categories: coding
image: images/unreal-googletest-integration/googletest-logo.png
tags: coding
description: "This article will show how to integrate GoogleTest as a Shared Library in UE4"
---

It often happens that when writing unit tests the SUT (System Under Test) cannot be tested in isolation because it is dependent on other components. In such cases there are generally three possibilities:
- If the DOCs (Depended On Component) are not contained in the same module of the SUT you can opt for writing an integration test (or at least a higher level test than a unit test with respect to the Test Pyramid).
- If the DOCs are contained in the same module of the SUT you can opt for writing a sociable unit test.
- If the DOCs are contained in the same module of the SUT you can opt for writing an isolated unit test as long as the possibility of creating Test Doubles exists.

Test Doubles is a generic term used to refer to all those objects that are used to replace production components in the testing phase: you can learn more about this topic by reading the [article written by Martin Fowler](https://martinfowler.com/bliki/TestDouble.html).
Although Test Doubles are also useful for writing sociable and integration unit tests, it is in writing isolated unit tests that they are most used since in the latter you try to test the smallest portion of code that can be logically isolated from the rest of the system. There are 5 types of Test Doubles: Fakes, Spies, Dummies, Stubs, and Mocks. In this article, we will focus on how to make it possible to create Dummies, Stubs, and Mocks. Although it is not necessary to use an external library to create this type of entity, there are several [COTS](https://en.wikipedia.org/wiki/Commercial_off-the-shelf) (Commercial Off the Shelf) libraries available to take advantage of, and we have chosen the GoogleTest library because it was the one with the most support and documentation as well as an excellent amount of features useful to us.

## Using GoogleMock and the Automation System of UE4

Although GoogleTest also offers an [xUnit test framework](https://en.wikipedia.org/wiki/XUnit), it was decided to rely on the Automation System provided by UE4 and to use only GoogleMock. As explained in the [article by Von Ihno Lübbers](https://www.maibornwolff.de/en/blog/how-we-create-real-unittests-unreal-engine-googlemock) by using an external test framework it is possible to test only those modules that depend only on the UE4 core modules. Although a possibility is to make the business logic independent from the graphical engine of reference in a similar way to what is explained in the [article by Eric Lemes](https://ericlemes.com/2018/12/17/unit-tests-in-unreal-pt-3/) this solution was precluded because of the size of our codebase: such an operation would have been too expensive in terms of costs and time.

## Compilation and integration of GoogleTest as a Shared Library for UE4

In the [UE4 documentation](https://docs.unrealengine.com/4.27/en-US/ProductionPipelines/BuildTools/UnrealBuildTool/ThirdPartyLibraries/) it is described how, once in possession of the binary files, to integrate third-party libraries, be they static or dynamic. In the next sections, we will explain why it was decided to compile GoogleTest as a dynamic library and illustrate the steps necessary to create it.

### Comparison of static and dynamic libraries

One of the disadvantages of static libraries is that each module would maintain its copy of the library, not only increasing the overall size of the executable but also making it necessary for each module to initialize its copy (as we will see later, this is a mandatory step). However, we must point out that static libraries remain a valid choice because they can be more performing and not vulnerable to the DLL Boundaries problem that forced us to make changes to the library itself.  

### The problem related to DLL Boundaries

On Windows, there is a problem with DLL Boundaries. This problem occurs when an object that is allocated in one DLL is passed by pointer to a second DLL that deallocates it. This can lead to memory errors if multiple copies of the CRT libraries are used. This implies that the `new` and `delete` operators used (as opposed to what happens in Linux dynamic libraries) are local to the DLL and in UE4 this is problematic: as you can read in the file `ModuleBoilerplate.h` each module redefines the `new` and `delete` operators in such a way that the functions `FMemory::Malloc` and `FMemory::Free` are respectively used. Since in the macros `EXPECT_CALL` and `ON_CALL` provided by GoogleMock, objects are built using the allocator of our module and destroyed using the library's deallocator (which is the default one provided by the language), errors inevitably arise in memory management. To solve this, it was necessary to modify the library in such a way that it was possible to redefine the operators `new` and `delete` and to use the functions `FMemory::Malloc` and `FMemory::Free`. There are 4 changes to be made to the library:

1. Adding the following `include` directive in the file `googletest/include/gtest/gtest.h`:

   ```cpp
   #include "gtest/gtest-memory.h"
   ```

2. Adding the following `include` directive in the file `googletest/src/gtest-all.cc`:

   ```cpp
   #include "src/gtest-memory.cc"
   ```

3. Adding the file `gtest-memory.h` to the path `googletest/include/gtest`:

   ```cpp
   #ifndef GOOGLETEST_INCLUDE_GTEST_GTEST_MEMORY_H_
   #define GOOGLETEST_INCLUDE_GTEST_GTEST_MEMORY_H_

   #if GTEST_OS_WINDOWS

   #include <functional>
   #include "gtest/internal/gtest-port.h"

   namespace testing {

   typedef std::function<void*(size_t)> NewOperatorFunction;

   typedef std::function<void(void*)> DeleteOperatorFunction;

   // Sets the function to be used in case of allocation
   // It's the caller's responsibility to ensure that the function in question does not raise any exceptions
   GTEST_API_ void SetNewOperatorFunction(NewOperatorFunction function);

   // Resets the function to be used in case of allocation
   GTEST_API_ void ResetNewOperatorFunction();

   // Set the function to be used in case of deallocation
   // It's the caller's responsibility to ensure that the function in question does not raise any exceptions
   GTEST_API_ void SetDeleteOperatorFunction(DeleteOperatorFunction function);

   // Resets the function to be used in case of deallocation
   GTEST_API_ void ResetDeleteOperatorFunction();

   }

   #endif // GTEST_OS_WINDOWS

   #endif // GOOGLETEST_INCLUDE_GTEST_GTEST_MEMORY_H_
   ```

4. Adding the file `gtest-memory.cc` to the path `googletest/src`:

   ```cpp
   #include "gtest/gtest-memory.h"

   #if GTEST_OS_WINDOWS

   namespace testing {

   NewOperatorFunction newOperatorFunction;

   DeleteOperatorFunction deleteOperatorFunction;

   GTEST_API_ void SetNewOperatorFunction(NewOperatorFunction function) {
       newOperatorFunction = function;
   }

   GTEST_API_ void ResetNewOperatorFunction() {
       newOperatorFunction = nullptr;
   }

   GTEST_API_ void SetDeleteOperatorFunction(DeleteOperatorFunction function) {
       deleteOperatorFunction = function;
   }

   GTEST_API_ void ResetDeleteOperatorFunction() {
       deleteOperatorFunction = nullptr;
   }

   }

   void* newOperatorTemplateFunction(size_t size) {
       size = size ? size : 1;

       if (testing::newOperatorFunction) {
           return testing::newOperatorFunction(size);
       }

       void* ptr;

       while ((ptr = std::malloc(size)) == 0) {
           std::new_handler newHandler = std::get_new_handler();

           if (newHandler) {
               newHandler();
           } else {
               throw std::bad_alloc();
           }
       }

       return ptr;
   }

   void* newOperatorTemplateFunctionNoThrow(size_t size) {
       void* ptr = nullptr;

       try {
           ptr = operator new(size);
       } catch (...) { }

       return ptr;
   }

   void deleteOperatorTemplateFunction(void* ptr) {
       if (testing::deleteOperatorFunction) {
           testing::deleteOperatorFunction(ptr);
       } else {
           free(ptr);
       }  
   }

   void* operator new(size_t size) { return newOperatorTemplateFunction(size); }

   void* operator new[](size_t size) { return newOperatorTemplateFunction(size); }

   void* operator new(size_t size, const std::nothrow_t&) throw() { return newOperatorTemplateFunctionNoThrow(size); }

   void* operator new[](size_t size, const std::nothrow_t&) throw() { return newOperatorTemplateFunctionNoThrow(size); }

   void operator delete(void* ptr) { deleteOperatorTemplateFunction(ptr); }

   void operator delete[](void* ptr) { deleteOperatorTemplateFunction(ptr); }

   void operator delete(void* ptr, const std::nothrow_t&) throw() { deleteOperatorTemplateFunction(ptr); }

   void operator delete[](void* ptr, const std::nothrow_t&) throw() { deleteOperatorTemplateFunction(ptr); }

   void operator delete(void* ptr, size_t) { deleteOperatorTemplateFunction(ptr); }

   void operator delete[](void* ptr, size_t) { deleteOperatorTemplateFunction(ptr); }

   void operator delete(void* ptr, size_t, const std::nothrow_t&) throw() { deleteOperatorTemplateFunction(ptr); }

   void operator delete[](void* ptr, size_t, const std::nothrow_t&) throw() { deleteOperatorTemplateFunction(ptr); }

   #endif // GTEST_OS_WINDOWS
   ```

At [this link](https://gist.github.com/stefano-strati/54865fec970257e578911fbab0fdd97a) you can download the git patch of the changes listed above.

### Compilation of GoogleTest as a Shared Library for UE4

Here are the steps to compile GoogleTest as a dynamic library for UE4 for Windows and Linux systems. To do this we will rely on CMake, as indicated in the [guidelines](https://github.com/google/googletest/blob/main/googletest/README.md) provided by the library itself.

#### Linux

1. Download the source of GoogleTest (the version we use is the [1.11.0](https://github.com/google/googletest/tree/release-1.11.0))

2. Modify the library as indicated in the paragraph concerning DLL Boundaries or by using [this git patch](https://gist.github.com/stefano-strati/54865fec970257e578911fbab0fdd97a).

3. Place the downloaded source in a folder together with the following file `CMakeList.txt`
    
   ```cmake
   cmake_minimum_required(VERSION 3.13)

   if (UNIX)
       # Settings
       set(LLVM_DIR "")
       set(UE4_DIR "")

       set(LLVM_BIN_DIR "${LLVM_DIR}/build/bin")
       set(CMAKE_C_COMPILER "${LLVM_BIN_DIR}/clang")
       set(CMAKE_CXX_COMPILER "${LLVM_BIN_DIR}/clang++")
   endif (UNIX)

   set(CMAKE_C_STANDARD 11)
   set(CMAKE_CXX_STANDARD 14)
         
   set(CMAKE_POSITION_INDEPENDENT_CODE TRUE)

   project("GoogleTestSolution")

   if (UNIX)
       set(UE4_LIBCXX_DIR "${UE4_DIR}/Engine/Source/ThirdParty/Linux/LibCxx")
       set(UE4_LIBCXX_INCLUDE_DIR "${UE4_LIBCXX_DIR}/include")
       set(UE4_LIBCXX_LIB_DIR "${UE4_LIBCXX_DIR}/lib/Linux/x86_64-unknown-linux-gnu")
       add_compile_options(-nostdinc++ -I${UE4_LIBCXX_INCLUDE_DIR}/ -I${UE4_LIBCXX_INCLUDE_DIR}/c++/v1)
       add_link_options(-nodefaultlibs -L${UE4_LIBCXX_LIB_DIR}/ ${UE4_LIBCXX_LIB_DIR}/libc++.a ${UE4_LIBCXX_LIB_DIR}/libc++abi.a -lm -lc -lpthread -lgcc_s -lgcc)
   endif (UNIX)

   set(BUILD_GMOCK TRUE CACHE BOOL "BUILD_GMOCK" FORCE)
   set(BUILD_SHARED_LIBS TRUE CACHE BOOL "BUILD_SHARED_LIBS" FORCE)
   set(INSTALL_GTEST FALSE CACHE BOOL "INSTALL_GTEST" FORCE)

   set(gmock_build_tests FALSE CACHE BOOL "gmock_build_tests" FORCE)
   set(gtest_build_samples FALSE CACHE BOOL "gtest_build_samples" FORCE)
   set(gtest_build_tests FALSE CACHE BOOL "gtest_build_tests" FORCE)
   set(gtest_disable_pthreads FALSE CACHE BOOL "gtest_disable_pthreads" FORCE)
   set(gtest_force_shared_crt TRUE CACHE BOOL "gtest_force_shared_crt" FORCE)
   set(gtest_hide_internal_symbols TRUE CACHE BOOL "gtest_hide_internal_symbols" FORCE)

   add_subdirectory("${CMAKE_SOURCE_DIR}/src")
   ```

4. Rename the folder containing the downloaded GoogleTest source code to `src`

5. Since UE4 4.27 uses Clang 11.0.1 it is necessary to specify in the variable `LLVM_DIR` in the file `CMakeList.txt` defined above the path to the compiler

    1. You can download the LLVM source code at [this link](https://github.com/llvm/llvm-project/releases/tag/llvmorg-11.0.1)

    2. To compile the downloaded source code run the following commands:

        1. `cd llvm-project/`        
        2. `mkdir build`        
        3. `cd build/`      
        4. `cmake ../llvm -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX=~/llvm-project/build -DBUILD_SHARED_LIBS=on -DLLVM_ENABLE_PROJECTS=clang`      
        5. `make -j$(nproc)`

6. Since UE4 uses its specific version of libc++ you need to specify in the `UE4_DIR` variable in the `CMakeList.txt` file defined above the path to the source code of your local version of UE4

7. As a last step, go to the folder containing the `src` folder and the file `CMakeList.txt` we created and run the following commands:

    1. `mkdir linux`
    2. `cd linux/`
    3. `cmake ..`
    4. `make`

#### Windows

1. Steps 1, 2, 3, and 4 are the same as for creating binaries on Linux

2. Open CMake GUI (the version used in this article is 3.22.1) and specify:
    - In the field `Where is the source code:` the path to the folder containing the `src` folder and the file `CMakeList.txt` we created
    - In the field `Where to build the binaries:` a path of your choice where CMake will create the solution from which to generate the binaries

3. Press the `Configure` button specifying Visual Studio 2019 as the project generator

4. Press the `Generate` button

5. Open the generated solution at the path specified in step 2 using Visual Studio 2019

6. To generate the DLLs using the Release configuration select the `Release` configuration and do the build of the `gmock` project

7. To generate the DLLs using the Debug configuration select the `Debug` configuration and:
    - Change the Runtime Libraries used by the `gmock` project from `Multi-threaded Debug DLL (/MDd)` to `Multi-threaded DLL (/MD)`.
    - Change Runtime Library used by project `gtest` from `Multi-threaded Debug DLL (/MDd)` to `Multi-threaded DLL (/MD)`.
    - Make the project `gmock` build.

### Integration of GoogleTest as a Shared Library for UE4

Once you have compiled GoogleTest as a dynamic library you need to integrate it with UE4 and to do that you need to create two modules: the `GoogleTestModule` and the `AutomationTestCore` module (you can choose the names you prefer but in this article, we will use these as a reference).
The `GoogleTestModule` is structured as follows:

- `GoogleTestmodule`
    - `include`
        - `gmock`
            - ...
        - `gtest`
            - ...
    - `lib`
        - `linux`
            - `x64`
                - `Release`
                    - `libgmock.so`
                    - `libgmock.so.1.11.0`
                    - `libgtest.so`
                    - `libgtest.so.1.11.0`
        - `win`
            - `x64`
                - `Debug`
                    - `gmockd.dll`
                    - `gmockd.lib`
                    - `gmockd.pdb`
                - `Release`
                    - `gmock.dll`
                    - `gmock.lib`
    - `GoogleTestModule.Build.cs`

At the path `GoogleTestmodule/include/gmock` copy the files of the GoogleTest library that you downloaded at point 1 of the previous paragraph from the path `src/googlemock/include/gmock`.
At the path `GoogleTestmodule/include/gtest` copy the files of the GoogleTest library that you downloaded at point 1 of the previous paragraph from the path `src/googletest/include/gtest` (make sure you have first made the changes specified in step 2 of the previous paragraph).
At the path `GoogleTestmodule/lib/linux/x64/Release` copy the binaries generated for Linux.
At the path `GoogleTestmodule/lib/win/x64/Debug` copy the binaries generated for Windows with the Debug configuration.
At the path `GoogleTestmodule/lib/win/x64/Release` copy the binaries generated for Windows with the Release configuration.

This is the content of the file `GoogleTestModule.Build.cs`:

```csharp
using System.IO;
using UnrealBuildTool;

public class GoogleTestModule : ModuleRules {
    public GoogleTestModule(ReadOnlyTargetRules Target) : base(Target) {
        Type = ModuleType.External;

        PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;

        PublicIncludePaths.Add(Path.Combine(ModuleDirectory, "include"));

        if (Target.Platform == UnrealTargetPlatform.Win64) {
            HandleWin64Platform();
        } else if (Target.Platform == UnrealTargetPlatform.Linux) {
            HandleLinuxPlatform();
        } else if (Target.Platform == UnrealTargetPlatform.Mac) {
            HandleMacPlatform();
        } else {
            HandleUnsupportedPlatform();
        }

        AddCrossPlatformPublicDefinitions();
    }

    private void HandleWin64Platform() {
        PublicDefinitions.Add("GTEST_CAN_STREAM_RESULTS_=0");
        PublicDefinitions.Add("GTEST_OS_WINDOWS=1");
        PublicDefinitions.Add("GTEST_OS_LINUX=0");
        PublicDefinitions.Add("GTEST_USES_POSIX_RE=0");

        if (Target.Configuration == UnrealTargetConfiguration.Debug || Target.Configuration == UnrealTargetConfiguration.DebugGame) {
            string GoogleTestLibsPath = Path.Combine(ModuleDirectory, "lib", "win", "x64", "Debug");
            PublicAdditionalLibraries.Add(Path.Combine(GoogleTestLibsPath, "gmockd.lib"));
            RuntimeDependencies.Add("$(BinaryOutputDir)/gmockd.dll", Path.Combine(GoogleTestLibsPath, "gmockd.dll"));
            RuntimeDependencies.Add("$(BinaryOutputDir)/gmockd.pdb", Path.Combine(GoogleTestLibsPath, "gmockd.pdb"));
        } else {
            string GoogleTestLibsPath = Path.Combine(ModuleDirectory, "lib", "win", "x64", "Release");
            PublicAdditionalLibraries.Add(Path.Combine(GoogleTestLibsPath, "gmock.lib"));
            RuntimeDependencies.Add("$(BinaryOutputDir)/gmock.dll", Path.Combine(GoogleTestLibsPath, "gmock.dll"));
        }
    }

    private void HandleLinuxPlatform() {
        PublicDefinitions.Add("GTEST_OS_WINDOWS=0");
        PublicDefinitions.Add("GTEST_OS_LINUX=1");

        string GoogleTestLibsPath = Path.Combine(ModuleDirectory, "lib", "linux", "x64", "Release");
        PublicRuntimeLibraryPaths.Add(GoogleTestLibsPath);
        PublicAdditionalLibraries.Add(Path.Combine(GoogleTestLibsPath, "libgtest.so"));
        PublicAdditionalLibraries.Add(Path.Combine(GoogleTestLibsPath, "libgmock.so"));
    }

    private void HandleMacPlatform() {
        PublicDefinitions.Add("GTEST_OS_MAC=1");

        string GoogleTestLibsPath = Path.Combine(ModuleDirectory, "lib", "mac");
        PublicRuntimeLibraryPaths.Add(GoogleTestLibsPath);
        PublicAdditionalLibraries.Add(Path.Combine(GoogleTestLibsPath, "libgtest.dylib"));
        PublicAdditionalLibraries.Add(Path.Combine(GoogleTestLibsPath, "libgmock.dylib"));
    }

    private void HandleUnsupportedPlatform() {
        string Error = "Target.Platform: " + Target.Platform + " not supported";
        System.Console.WriteLine(Error);
        throw new BuildException(Error);
    }

    private void AddCrossPlatformPublicDefinitions() {
        PublicDefinitions.Add("GTEST_CREATE_SHARED_LIBRARY=0");

        PublicDefinitions.Add("GTEST_DONT_DEFINE_ASSERT_EQ=0");
        PublicDefinitions.Add("GTEST_DONT_DEFINE_ASSERT_FALSE=0");
        PublicDefinitions.Add("GTEST_DONT_DEFINE_ASSERT_GE=0");
        PublicDefinitions.Add("GTEST_DONT_DEFINE_ASSERT_GT=0");
        PublicDefinitions.Add("GTEST_DONT_DEFINE_ASSERT_LE=0");
        PublicDefinitions.Add("GTEST_DONT_DEFINE_ASSERT_LT=0");
        PublicDefinitions.Add("GTEST_DONT_DEFINE_ASSERT_NE=0");
        PublicDefinitions.Add("GTEST_DONT_DEFINE_ASSERT_TRUE=0");
        PublicDefinitions.Add("GTEST_DONT_DEFINE_EXPECT_FALSE=0");
        PublicDefinitions.Add("GTEST_DONT_DEFINE_EXPECT_TRUE=0");
        PublicDefinitions.Add("GTEST_DONT_DEFINE_FAIL=0");
        PublicDefinitions.Add("GTEST_DONT_DEFINE_SUCCEED=0");
        PublicDefinitions.Add("GTEST_DONT_DEFINE_TEST=0");

        PublicDefinitions.Add("GTEST_FOR_GOOGLE_=0");

        PublicDefinitions.Add("GTEST_HAS_ABSL=0");
        PublicDefinitions.Add("GTEST_HAS_DOWNCAST_=0");
        PublicDefinitions.Add("GTEST_HAS_EXCEPTIONS=0");
        PublicDefinitions.Add("GTEST_HAS_MUTEX_AND_THREAD_LOCAL_=0");
        PublicDefinitions.Add("GTEST_HAS_NOTIFICATION_=0");
        PublicDefinitions.Add("GTEST_HAS_RTTI=0");

        PublicDefinitions.Add("GTEST_INTERNAL_HAS_ANY=0");
        PublicDefinitions.Add("GTEST_INTERNAL_HAS_OPTIONAL=0");
        PublicDefinitions.Add("GTEST_INTERNAL_HAS_STRING_VIEW=0");
        PublicDefinitions.Add("GTEST_INTERNAL_HAS_VARIANT=0");

        PublicDefinitions.Add("GTEST_LINKED_AS_SHARED_LIBRARY=1");

        PublicDefinitions.Add("GTEST_OS_AIX=0");
        PublicDefinitions.Add("GTEST_OS_CYGWIN=0");
        PublicDefinitions.Add("GTEST_OS_DRAGONFLY=0");
        PublicDefinitions.Add("GTEST_OS_ESP32=0");
        PublicDefinitions.Add("GTEST_OS_ESP8266=0");
        PublicDefinitions.Add("GTEST_OS_FREEBSD=0");
        PublicDefinitions.Add("GTEST_OS_FUCHSIA=0");
        PublicDefinitions.Add("GTEST_OS_GNU_KFREEBSD=0");
        PublicDefinitions.Add("GTEST_OS_HAIKU=0");
        PublicDefinitions.Add("GTEST_OS_HPUX=0");
        PublicDefinitions.Add("GTEST_OS_IOS=0");
        PublicDefinitions.Add("GTEST_OS_LINUX_ANDROID=0");
        PublicDefinitions.Add("GTEST_OS_NACL=0");
        PublicDefinitions.Add("GTEST_OS_NETBSD=0");
        PublicDefinitions.Add("GTEST_OS_OPENBSD=0");
        PublicDefinitions.Add("GTEST_OS_QNX=0");
        PublicDefinitions.Add("GTEST_OS_SOLARIS=0");
        PublicDefinitions.Add("GTEST_OS_WINDOWS_MINGW=0");
        PublicDefinitions.Add("GTEST_OS_WINDOWS_MOBILE=0");
        PublicDefinitions.Add("GTEST_OS_WINDOWS_PHONE=0");
        PublicDefinitions.Add("GTEST_OS_WINDOWS_RT=0");
        PublicDefinitions.Add("GTEST_OS_WINDOWS_TV_TITLE=0");
        PublicDefinitions.Add("GTEST_OS_XTENSA=0");
        PublicDefinitions.Add("GTEST_OS_ZOS=0");

        PublicDefinitions.Add("GTEST_USES_PCRE=0");
    }
}
```

The `AutomationTestCore` module is structured as follows:

- `AutomationTestCore`
    - `Private`
        - `ATCGTestFailureReporter.cpp`
        - `ATCGTestFailureReporter.h`
        - `AutomationTestCoreModule.cpp`
        - `AutomationTestCoreModule.h`
    - `Public`
        - `ATCGoogleTest.h`
    - `AutomationTestCore.Build.cs`

This is the content of the file `AutomationTestCore.Build.cs`:

```csharp
using System.IO;
using UnrealBuildTool;

public class AutomationTestCore : ModuleRules {
    public AutomationTestCore(ReadOnlyTargetRules Target) : base(Target) {
        PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;
 
        PrivateIncludePaths.AddRange(
            new[] { Path.Combine(ModuleDirectory, "Private") });
 
        PublicDependencyModuleNames.AddRange(new[] {
            "Core",
            "CoreUObject",
            "Engine",
            "GoogleTestModule",
        });
    }
}
```

This is the content of the files `ATCGTestFailureReporter.h` and `ATCGTestFailureReporter.cpp` (for their realization has been followed the idea described in the [article by Von Ihno Lübbers](https://www.maibornwolff.de/en/blog/how-we-create-real-unittests-unreal-engine-googlemock)):

```cpp
#pragma once

#include "ATCGoogleTest.h"

class FATCGTestFailureReporter final : public testing::EmptyTestEventListener {
    void OnTestPartResult(const testing::TestPartResult& result) override;
};
```

```cpp
#include "ATCGTestFailureReporter.h"

#include "Containers/UnrealString.h"
#include "CoreGlobals.h"

void FATCGTestFailureReporter::OnTestPartResult(const testing::TestPartResult& result) {
    const auto logType = result.type();
    const auto logMessage = FString(result.message());
 
    if (logType == testing::TestPartResult::kFatalFailure || logType == testing::TestPartResult::kNonFatalFailure) {
        UE_LOG(LogTemp, Error, TEXT("%s"), *logMessage);
    } else {
        UE_LOG(LogTemp, Display, TEXT("%s"), *logMessage);
    }
}
```

This is the content of the files `AutomationTestCoreModule.h` and `AutomationTestCoreModule.cpp`:

```cpp
#pragma once

#include "CoreMinimal.h"
#include "Modules/ModuleInterface.h"

class FATCGTestFailureReporter;

class FAutomationTestCoreModule final : public IModuleInterface {
public:
    void StartupModule() override;

    void ShutdownModule() override;

private:
    FATCGTestFailureReporter* _testFailureReporter = nullptr;
};
```

```cpp
#include "AutomationTestCoreModule.h"

#include "ATCGTestFailureReporter.h"
#include "ATCGoogleTest.h"

#include "HAL/Platform.h"
#include "Modules/ModuleManager.h"

using namespace testing;

IMPLEMENT_MODULE(FAutomationTestCoreModule, AutomationTestCore)

void FAutomationTestCoreModule::StartupModule() {
#if PLATFORM_WINDOWS
    SetNewOperatorFunction([](size_t size) { return FMemory::Malloc(size ? size : 1); });
    SetDeleteOperatorFunction([](void* ptr) { FMemory::Free(ptr); });
#endif

    _testFailureReporter = new FATCGTestFailureReporter();
    TestEventListeners& googleTestListeners = UnitTest::GetInstance()->listeners();
    googleTestListeners.Release(googleTestListeners.default_result_printer());
    googleTestListeners.Append(_testFailureReporter);
}

void FAutomationTestCoreModule::ShutdownModule() {
    UnitTest::GetInstance()->listeners().Release(_testFailureReporter);
    delete _testFailureReporter;

#if PLATFORM_WINDOWS
    ResetNewOperatorFunction();
    ResetDeleteOperatorFunction();
#endif
}
```

This is the content of the file `ATCGoogleTest.h`:

```cpp
#pragma once

#include "HAL/Platform.h"

#if PLATFORM_WINDOWS
#include "Windows/AllowWindowsPlatformTypes.h"
#include "Windows/WindowsHWrapper.h"
#endif

THIRD_PARTY_INCLUDES_START
#include "gmock/gmock.h"
#include "gtest/gtest.h"
THIRD_PARTY_INCLUDES_END

#if PLATFORM_WINDOWS
#include "Windows/HideWindowsPlatformTypes.h"
#endif
```

This is the file you have to include every time you want to use the functionality of the GoogleTest library.

## Shipping builds

One unresolved problem is that unit tests, having to test parts of code not exposed to other modules, must be contained in the very module they are testing. Currently, there is no automated mechanism to remove such files from shipping builds other types of tests (such as integration tests) can simply be placed in a separate module of type `Editor` (which can be defined in the UE4 project's `uproject` file). A clean solution would be to modify the UE4 toolchain to exclude certain paths (which will be those containing the mock and test classes) from the shipping builds.

# References

- [https://github.com/google/googletest](https://github.com/google/googletest)
- [https://martinfowler.com/bliki/TestPyramid.html](https://martinfowler.com/bliki/TestPyramid.html)
- [https://martinfowler.com/bliki/UnitTest.html](https://martinfowler.com/bliki/UnitTest.html)
- [https://martinfowler.com/bliki/TestDouble.html](https://martinfowler.com/bliki/TestDouble.html)
- [https://www.ibm.com/docs/en/aix/7.2?topic=techniques-when-use-dynamic-linking-static-linking](https://www.ibm.com/docs/en/aix/7.2?topic=techniques-when-use-dynamic-linking-static-linking)
- [https://docs.microsoft.com/en-us/cpp/c-runtime-library/potential-errors-passing-crt-objects-across-dll-boundaries?view=msvc-170](https://docs.microsoft.com/en-us/cpp/c-runtime-library/potential-errors-passing-crt-objects-across-dll-boundaries?view=msvc-170)
- [https://www.maibornwolff.de/en/blog/how-we-create-real-unittests-unreal-engine-googlemock](https://www.maibornwolff.de/en/blog/how-we-create-real-unittests-unreal-engine-googlemock)
- [https://ericlemes.com/2018/12/17/unit-tests-in-unreal-pt-3/](https://ericlemes.com/2018/12/17/unit-tests-in-unreal-pt-3/)
