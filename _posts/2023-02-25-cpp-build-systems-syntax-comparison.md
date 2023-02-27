## Comparing the syntax of build systems that support C++

Let's see how the same C++ project (or at least very similar ones) would look like in different modern build systems. Courtesy of ChatGPT.

---

I requested that each project contains the following. This covers many real-life uses.
* A static library dependency.
* Custom include path.
* Custom target / output path.
* Custom compiler flags for Windows.

The list is in no particular order.

### Build generators

First, we'll look into the build systems that are actually build generators - they create project files or build scripts for various other IDEs and build executors, but can't build (invoke the compiler and linker) themselves.

#### Meson

```python
project('my_project', 'cpp')

# Define source files
src_files = ['src/file1.cpp', 'src/file2.cpp']
inc_dirs = include_directories('include')

# Define the library target as a static library
my_library = static_library('my_library', 
    sources: src_files,
    include_directories: inc_dirs,
    type: 'static',
    install: true,
)

# Add dependency on another static library
my_dependency = static_library('my_dependency',
    sources: [],
    link_with: 'path/to/my/lib.a',
)

# Set compiler flags for MSVC on Windows
if (is_windows())
    add_project_arguments('/EHsc', '/std:c++latest', language: 'cpp', when: 'cpp')
endif()

# Define the executable target and link against the library
my_executable = executable('my_executable',
    sources: ['src/main.cpp'],
    link_with: my_library,
)
```

#### CMake

```cmake
cmake_minimum_required(VERSION 3.10)

project(my_project LANGUAGES CXX)

# Set the target name and source files for the library
add_library(my_library STATIC 
    src/file1.cpp 
    src/file2.cpp)

# Add a dependency on another static library
add_library(my_dependency STATIC IMPORTED)
set_target_properties(my_dependency PROPERTIES IMPORTED_LOCATION path/to/my/lib.a)
target_link_libraries(my_library PRIVATE my_dependency)

# Set compiler flags for MSVC on Windows
if (MSVC)
    target_compile_options(my_library PRIVATE /EHsc /std:c++latest)
endif()

# Set a custom include path
target_include_directories(my_library PUBLIC include)

# Set a custom output path
set_target_properties(my_library PROPERTIES
    ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
)

# Define an executable target and link against the library
add_executable(my_executable src/main.cpp)
target_link_libraries(my_executable PRIVATE my_library)
```

#### Premake 5

```lua
workspace "my_project"
   configurations { "Debug", "Release" }
   -- Set the target name and source files for the library
   project "my_library"
      kind "StaticLib"
      language "C++"
      targetname "my_library"
      files { "src/file1.cpp", "src/file2.cpp" }

      -- Add a dependency on another static library
      includedirs { "path/to/my/include" }
      links { "my_dependency" }

      -- Set compiler flags for MSVC on Windows
      filter { "system:windows", "action:vs*" }
         flags { "/EHsc", "/std:c++latest" }
      filter {}

      -- Set a custom output path
      targetdir "lib"

   -- Define an executable target and link against the library
   project "my_executable"
      kind "ConsoleApp"
      language "C++"
      files { "src/main.cpp" }
      links { "my_library" }

      -- Set a custom include path
      includedirs { "include" }
```

#### qmake

```make
# Set the target name and source files for the library
TARGET = my_library
TEMPLATE = lib
CONFIG += staticlib
SOURCES += src/file1.cpp \
           src/file2.cpp

# Add a dependency on another static library
INCLUDEPATH += path/to/my/include
LIBS += -Lpath/to/my/lib -lmy_dependency

# Set compiler flags for MSVC on Windows
win32: QMAKE_CXXFLAGS += /EHsc /std:c++latest

# Set a custom output path
DESTDIR = lib

# Define an executable target and link against the library
TARGET = my_executable
TEMPLATE = app
SOURCES += src/main.cpp
INCLUDEPATH += include
LIBS += -Llib -lmy_library
```

#### WAF

```python
import os

def configure(conf):
    # Set the compiler flags for MSVC on Windows
    if conf.env.CC_NAME == 'msvc':
        conf.env.CXXFLAGS += ['/EHsc', '/std:c++latest']

    # Set the custom include path
    conf.env.INCLUDES += [conf.path.abspath('include')]

def build(bld):
    # Set the target name and source files for the library
    my_library = bld.create_static_library(
        name='my_library',
        src=[
            'src/file1.cpp',
            'src/file2.cpp',
        ],
        includes='include',
    )

    # Add a dependency on another static library
    my_dependency = bld.read_shlib(
        'my_dependency',
        paths=['path/to/my/lib.a'],
        features=['cxx', 'cxxstlib'],
        export_includes=['include'],
    )
    my_library.use_libs(my_dependency)

    # Define an executable target and link against the library
    my_executable = bld.program(
        source='src/main.cpp',
        target='my_executable',
        use=my_library,
    )

    # Set a custom output path
    my_library.post()
    my_library.path = os.path.join(bld.bldnode.abspath(), 'lib')
```

#### GN (Generate Ninja)

```python
# Set the target name and source files for the library
my_library = static_library("my_library") {
    sources = [
        "src/file1.cpp",
        "src/file2.cpp",
    ],
    includes = [ "include" ],
}

# Add a dependency on another static library
my_dependency = static_library("my_dependency") {
    deps = [ ":my_lib" ],
    include_dirs = [ "include" ],
}
my_library.deps += [ my_dependency ]

# Set compiler flags for MSVC on Windows
if (target_cpu == "x86_64" && toolchain == "msvc") {
    my_library.cflags_c = [ "/EHsc", "/std:c++latest" ]
}

# Define an executable target and link against the library
executable("my_executable") {
    sources = [ "src/main.cpp" ],
    deps = [ ":my_library" ],
}

# Set a custom output path
output_directory = get_target_gen_dir() + "/lib"
if (is_win) {
    output_directory += "/$configuration"
}
my_library.output_dir = output_directory
```

#### Sharpmake

```cs
using System;
using System.Collections.Generic;
using Sharpmake;

[module: Sharpmake.Include("CustomOptions.cs")]
[module: Sharpmake.Include("CustomOutput.cs")]

[Generate]
class MyProject : CPlusPlusProject
{
    public MyProject()
    {
        Name = "my_project";

        AddTargets(new Target(
            Platform.win64,
            DevEnv.vs2019,
            Optimization.Debug | Optimization.Release));

        SourceRootPath = @"[project.SharpmakeCsPath]\..\src";

        AddLibrary("my_library", new[] {
            @"src\file1.cpp",
            @"src\file2.cpp",
        });

        AddLinkerOptions("my_library", new[] {
            "-Wl,-rpath,$ORIGIN",
        });

        AddIncludeDirs("my_library", new[] {
            @"include",
        });

        AddLibrary("my_dependency", new[] {
            @"path\to\my\lib.lib",
        });

        AddLinkerOptions("my_dependency", new[] {
            "-Lpath/to/my",
        });

        AddDependency("my_library", "my_dependency");

        AddExecutable("my_executable", new[] {
            @"src\main.cpp",
        }, "my_library");
    }

    [Configure()]
    public void ConfigureAll(Configuration conf, Target target)
    {
        conf.IncludePaths.Add(@"[project.SharpmakeCsPath]\..\include");

        conf.Options.Add(SharpmakeOptions.Vc.Compiler.CppLanguageStandard.CPPLatest);

        if (target.Platform == Platform.win64)
        {
            conf.Options.Add(SharpmakeOptions.Vc.Compiler.Exceptions.EnableWithSEH);
            conf.Options.Add(SharpmakeOptions.Vc.Compiler.RuntimeLibrary.MultiThreadedDebugDLL);
            conf.Options.Add(SharpmakeOptions.Vc.Compiler.RuntimeLibrary.MultiThreadedDLL);
            conf.Options.Add(SharpmakeOptions.Vc.Compiler.WarningLevel.Level3);

            conf.Output = "[conf.Name]\\";
            conf.TargetPath = "[conf.OutputDirectory]\\bin\\[target.Platform]\\[conf.Name]\\";
            conf.IntermediatePath = "[conf.OutputDirectory]\\obj\\[target.Platform]\\[conf.Name]\\";
        }
    }
}
```

#### Boost.Build (B2)

```
project my_library : requirements
    <link>static
    <cxxflags>-std=c++latest
    <cxxflags>/EHsc
    ;

lib my_library
    : src/file1.cpp src/file2.cpp
    : <include>path/to/my/include
    : <variant>release:<name>my_library:<install-type>LIB:<location>lib
    : <variant>debug:<name>my_library:<install-type>LIB:<location>lib/debug
    ;

lib my_dependency
    : path/to/my/lib.a
    ;

exe my_executable
    : src/main.cpp
    : my_library
    : <include>include
    : <variant>release:<name>my_executable:<install-type>BIN:<location>bin
    : <variant>debug:<name>my_executable:<install-type>BIN:<location>bin/debug
    ;

install install : my_library my_executable
    ;
```

#### Tundra

```lua
Settings {
    Env = {
        CXXOPTS = {
            Release = { "/EHsc", "/std:c++latest" },
            Debug   = { "/EHsc", "/std:c++latest", "-g" },
        },
        VARIANT_DIR = ".build/$(VARIANT)",
        OUTPUT_DIR  = "$(VARIANT_DIR)/bin",
        OBJECT_DIR  = "$(VARIANT_DIR)/obj",
        LIB_DIR     = "$(VARIANT_DIR)/lib",
    },
    Platforms = {
        win64 = {
            CXX = "cl",
            CXXOPTS = {
                Debug   = { "/MDd" },
                Release = { "/MD" },
            },
            LD = "link",
            LDOPTS = {
                Debug   = { "/DEBUG" },
                Release = {},
            },
            LIBRARY_PREFIX = "",
            LIBRARY_SUFFIX = ".lib",
            EXECUTABLE_SUFFIX = ".exe",
        },
    },
}

Program {
    Name = "my_executable",
    Sources = { "src/main.cpp" },
    Libs = { "my_library" },
    Includes = { "include" },
    Frameworks = {},
    Defs = {},
    PreBuildCommands = {},
    PostBuildCommands = {},
    Variant = "Release",
    Config = "win64",
}

Library {
    Name = "my_library",
    Sources = { "src/file1.cpp", "src/file2.cpp" },
    Includes = { "include" },
    Frameworks = {},
    Defs = {},
    Variant = "Release",
    Config = "win64",
}

StaticLibrary {
    Name = "my_dependency",
    Sources = {},
    Libs = { "path/to/my/lib.a" },
    Includes = {},
    Frameworks = {},
    Defs = {},
    Variant = "Release",
    Config = "win64",
}
```

### Executors / generators

These systems can both generate projects for other build systems / IDEs, and build their own projects without invoking an additional lower-level build system - your choice how to use them.

#### qbs

```js
import qbs

// Set up the project
Project {
    name: "my_project"

    // Set up the library target
    StaticLibrary {
        name: "my_library"
        files: [
            "src/file1.cpp",
            "src/file2.cpp"
        ]

        // Add a dependency on another static library
        cpp.includePaths: ["path/to/my/include"]
        cpp.systemIncludePaths: ["path/to/my/system/include"]
        cpp.libraryPaths: ["path/to/my/lib"]
        cpp.staticLibraries: ["my_dependency"]

        // Set compiler flags for MSVC on Windows
        cpp.compilerFlags: ["-EHsc", "-std:c++latest"]
        qbs.buildVariant: "release"
        qbs.architecture: "x86_64"

        // Set a custom output path
        destinationDirectory: "lib"
    }

    // Set up the executable target
    cpp.executableTemplate: "console"
    cpp.applicationName: "my_executable"
    cpp.sourceFiles: ["src/main.cpp"]
    cpp.includePaths: ["include"]
    cpp.staticLibraries: ["my_library"]

    // Set a custom output path
    files: {
        "my_executable": "bin/my_executable"
    }
}
```

#### Bazel

```perl
# Define the Bazel workspace
workspace(name = "my_project")

# Define the static library target
cc_library(
    name = "my_library",
    srcs = ["src/file1.cpp", "src/file2.cpp"],
    includes = ["include"],
    copts = select({
        "//conditions:default": ["-std=c++17"],
        "//conditions:windows": ["/EHsc", "/std:c++latest"],
    }),
    linkopts = ["-Lpath/to/my", "-lmy_dependency"],
)

# Define the executable target
cc_binary(
    name = "my_executable",
    srcs = ["src/main.cpp"],
    deps = [":my_library"],
)

# Set a custom output directory for the library target
filegroup(
    name = "my_library_files",
    srcs = [":my_library"],
    visibility = ["//visibility:private"],
)
exports_files(["BUILD"])
cc_library(
    name = "my_library",
    visibility = ["//visibility:public"],
    data = [":my_library_files"],
    strip_include_prefix = "path/to",
    strip_prefix = "path/to",
    linkstatic = True,
    linkshared = False,
)
filegroup(
    name = "lib_files",
    srcs = [":my_library"],
    visibility = ["//visibility:public"],
)
filegroup(
    name = "include_files",
    srcs = glob(["include/**/*.h"]),
    visibility = ["//visibility:public"],
)
exports_files(["include"])
exports_files(["lib"])
exports_files(["BUILD"])
filegroup(
    name = "include",
    srcs = glob(["include/**/*.h"]),
    visibility = ["//visibility:public"],
)
```

#### Buck (Facebook)

```python
cxx_library(
    name = 'my_library',
    srcs = [        'src/file1.cpp',        'src/file2.cpp',    ],
    deps = [        ':my_dependency',    ],
    includes = [        'include',    ],
    copts = select({
        'windows': [
            '/EHsc',
            '/std:c++latest',
        ],
        '//conditions:default': [],
    }),
    linkerflags = [        '-Wl,-rpath,$ORIGIN',    ],
    visibility = [        'PUBLIC',    ],
    link_style = 'static',
    link_whole = True,
)

cxx_library(
    name = 'my_dependency',
    linkstyle = 'static',
    link_whole = True,
    link_libs = [        'path/to/my/lib',    ],
    visibility = [        'PUBLIC',    ],
)

cxx_binary(
    name = 'my_executable',
    srcs = [        'src/main.cpp',    ],
    deps = [        ':my_library',    ],
    visibility = [        'PUBLIC',    ],
)

# Set a custom output path
genrule(
    name = 'copy_library',
    srcs = [        ':my_library',    ],
    outs = [        'build/lib/libmy_library.a',    ],
    cmd = 'mkdir -p $$(dirname $OUT) && cp $< $OUT',
)
```

#### Fastbuild

```python
Settings(
    CompilerPath = 'C:/Program Files (x86)/Microsoft Visual Studio/2019/Community/VC/Tools/MSVC/14.28.29910/bin/Hostx86/x86',
    LinkerPath = 'C:/Program Files (x86)/Microsoft Visual Studio/2019/Community/VC/Tools/MSVC/14.28.29910/bin/Hostx86/x86',
    LibraryPaths = [ 'path/to/my' ],
    IncludePaths = [ 'include' ]
)

Library('my_dependency',
    Sources = [],
    Libs = [ 'llib' ],
    Type = 'Static'
)

Library('my_library',
    Sources = [ 'src/file1.cpp', 'src/file2.cpp' ],
    Libs = [ ':my_dependency' ],
    Type = 'Static'
)

if (TargetEnvironment == 'windows')
{
    SetTargetProperties(
        'my_library',
        OutputDirectory = 'build',
        OutputFilename = 'libmy_library.lib',
        CompilerOptions = '/EHsc /std:c++latest',
        LinkerOptions = '-Wl,-rpath,$ORIGIN'
    )
}
else
{
    SetTargetProperties(
        'my_library',
        OutputDirectory = 'build',
        OutputFilename = 'libmy_library.a'
    )
}

Executable('my_executable',
    Sources = [ 'src/main.cpp' ],
    Libs = [ ':my_library' ]
)

if (TargetEnvironment == 'windows')
{
    SetTargetProperties(
        'my_executable',
        OutputDirectory = 'build',
        OutputFilename = 'my_executable.exe',
        CompilerOptions = '/EHsc /std:c++latest',
        LinkerOptions = '-Wl,-rpath,$ORIGIN'
    )
}
else
{
    SetTargetProperties(
        'my_executable',
        OutputDirectory = 'build',
        OutputFilename = 'my_executable'
    )
)

```

#### xmake

```lua
-- Define the project and its language
add_rules('mode.debug', 'mode.release')
target('my_project')
    set_languages('cxx17')

-- Set the source files for the library
add_files('src/file1.cpp', 'src/file2.cpp')

-- Add a dependency on another static library
add_deps('my_dependency')
add_linkdirs('path/to/my')
add_links('lib')

-- Set compiler flags for MSVC on Windows
if is_plat('windows') then
    add_cxxflags('/EHsc', '/std:c++latest')
end

-- Set a custom include path
add_includedirs('include')

-- Set a custom output path
set_targetdir('build')
set_objectdir('build/obj')
set_targetname('my_library')
set_targetprefix('lib')
add_ldflags('-Wl,-rpath,$ORIGIN')

-- Define an executable target and link against the library
target('my_executable')
    set_kind('binary')
    add_files('src/main.cpp')
    add_deps('my_library')
```

### Build executors

These build systems actually build your projects, but they cannot generate a project for your IDE or for another build system.

#### Jam

```
# Set the project name and version
project my_project : version 1.0 ;

# Set the C++ toolset
toolset gcc ;

# Set the source files for the library
lib my_library : src/file1.cpp src/file2.cpp ;

# Add a dependency on another static library
lib my_dependency : : <file>path/to/my/lib.a ;
use-project /my_dependency ;

# Set compiler flags for GCC on Linux
if $(os) = LINUX
{
    CFLAGS += -std=c++17 ;
}

# Set compiler flags for MSVC on Windows
if $(os) = NT
{
    CFLAGS += /EHsc /std:c++latest ;
}

# Set a custom include path
INCLUDES += include ;

# Set a custom output path
lib my_library
{
    LOCATE lib ;
    BINDIR $(CWD)/build/lib ;
}

# Define an executable target and link against the library
exe my_executable : src/main.cpp : my_library ;
```

#### Makefile

```make
CC = g++
AR = ar

CFLAGS = -Wall -Wextra -Wpedantic
ifeq ($(OS),Windows_NT)
    CFLAGS += -std=c++17 -m64
else
    CFLAGS += -std=c++17
endif

INCLUDES = -Iinclude

LDFLAGS =

LIBRARY_OUTPUT_PATH = $(CURDIR)/lib

SRCS = src/file1.cpp src/file2.cpp
OBJS = $(SRCS:.cpp=.o)

all: my_executable

libmy_library.a: $(OBJS)
	$(AR) rcs $(LIBRARY_OUTPUT_PATH)/$@ $(OBJS)

%.o: %.cpp
	$(CC) $(CFLAGS) $(INCLUDES) -c $< -o $@

my_executable: src/main.cpp libmy_library.a
	$(CC) $(CFLAGS) $(INCLUDES) $(LDFLAGS) src/main.cpp -o my_executable -L$(LIBRARY_OUTPUT_PATH) -lmy_library

clean:
	rm -f $(LIBRARY_OUTPUT_PATH)/libmy_library.a $(OBJS) my_executable
```

#### Ninja

```make
cxx = c++
cxx_flags = -Wall -Wextra -Wpedantic

# Targets

build my_library.a: cxx_library | src/file1.cpp src/file2.cpp
  ldflags = 
  include_directories = include
  lib = 
  linkflags = 
  out = my_library.a
  pic = 
  position_dependent_code = False
  soname = 
  sources = src/file1.cpp src/file2.cpp

build my_dependency.a: phony
  ldflags = 
  include_directories = 
  lib = path/to/my/lib.a
  linkflags = 
  out = my_dependency.a
  pic = 
  position_dependent_code = False
  soname = 

build my_executable: cxx_link | src/main.cpp my_library.a
  ldflags = 
  include_directories = 
  lib = my_library.a
  linkflags = 
  out = my_executable
  pic = 
  position_dependent_code = False
  soname = 
  sources = src/main.cpp
```

#### Gradle

```js
plugins {
    id 'cpp'
}

model {
    components {
        my_library(NativeLibrarySpec) {
            sources {
                cpp {
                    source {
                        srcDirs "src"
                        include "**/*.cpp"
                    }
                }
            }

            binaries {
                if (toolChain in VisualCpp) {
                    staticLibrary {
                        libFile = "my_library.lib"
                        cppCompiler.args "/EHsc", "/std:c++latest"
                    }
                } else {
                    staticLibrary {
                        libFile = "libmy_library.a"
                        cppCompiler.args "-std=c++17"
                    }
                }
            }

            buildTypes {
                release {
                    cppCompiler.args "-O2"
                }
            }

            targetPlatform "x86_64"
            targetPlatformVersion "10.0"
            targetMachines "machine1"
            targetDirectory "build"
        }

        my_executable(NativeExecutableSpec) {
            sources {
                cpp {
                    source {
                        srcDirs "src"
                        include "**/*.cpp"
                    }
                }
            }

            binaries.all {
                if (toolChain in VisualCpp) {
                    linker.args "/SUBSYSTEM:CONSOLE", "/ENTRY:mainCRTStartup"
                }
            }

            linkLibraries "my_library"
        }
    }
}
```

#### Scons 4.2.0

```python
env = Environment()

# Set compiler flags for MSVC on Windows
if env['PLATFORM'] == 'win32':
    env.Append(CCFLAGS=['/EHsc', '/std:c++latest'])

# Set a custom include path
env.Append(CPPPATH=['include'])

# Set a custom output path
env.Append(LIBPATH=['build'])
env.Append(LINKFLAGS=['-Wl,-rpath,$ORIGIN'])
env.Append(LIBPREFIX='lib')

# Set the target name and source files for the library
my_library = env.StaticLibrary(target='my_library', 
    source=['src/file1.cpp', 'src/file2.cpp'])

# Add a dependency on another static library
my_dependency = env.StaticLibrary(target='my_dependency', 
    source=[], 
    LIBPATH=['path/to/my'], 
    LIBS=['lib'])

my_library.Depends(my_library, my_dependency)

# Define an executable target and link against the library
my_executable = env.Program(target='my_executable', 
    source='src/main.cpp', 
    LIBS=[my_library])
```
