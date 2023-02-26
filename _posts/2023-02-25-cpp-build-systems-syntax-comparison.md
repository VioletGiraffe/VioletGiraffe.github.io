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

First, we'll look into the build systems that are actually build generators - they create project files or build scripts for various IDEs and build executors. 

#### Meson

```python
project('my_project', 'cpp')

# Set the target name and source files for the library
my_library = static_library('my_library', 
    sources : ['src/file1.cpp', 'src/file2.cpp'])

# Add a dependency on another static library
my_dependency = static_library('my_dependency', 
    sources : [], 
    link_args : ['-Lpath/to/my/', '-llib'])

my_library.add_dependency(my_dependency)

# Set compiler flags for MSVC on Windows
if meson.is_cross_build() and meson.get_cross_property('system_name').startswith('windows')
    my_library.add_project_arguments('/EHsc', '/std:c++latest')
endif

# Set a custom include path
my_library.include_directories('include')

# Set a custom output path
my_library.set_target_filename('my_library')
my_library.set_static_linker_args('-Wl,-rpath,$ORIGIN')
my_library.set_default_build_dir('build')
my_library.set_static_library_prefix('lib')

# Define an executable target and link against the library
my_executable = executable('my_executable', 
    sources : ['src/main.cpp'], 
    link_with : my_library)
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

####WAF

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

### Executors / generators

These systems can both generate projects and build their own projects without invoking an additional build system.

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

### Build executors

#### Jam

```ruby
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
