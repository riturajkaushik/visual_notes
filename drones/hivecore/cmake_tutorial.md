# CMake Tutorial — Everything You Need to Know

> **Generated from the real CMakeLists.txt files in this repository.**
> Every concept, command, and pattern shown here is used in production code in this project.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Step 1 — Minimum Project Setup](#step-1--minimum-project-setup)
3. [Step 2 — Setting the C++ Standard](#step-2--setting-the-c-standard)
4. [Step 3 — Compiler Detection & Warning Flags](#step-3--compiler-detection--warning-flags)
5. [Step 4 — Build Options with `option()`](#step-4--build-options-with-option)
6. [Step 5 — Finding Dependencies with `find_package()`](#step-5--finding-dependencies-with-find_package)
7. [Step 6 — Finding Dependencies with `pkg-config`](#step-6--finding-dependencies-with-pkg-config)
8. [Step 7 — Creating Library Targets (SHARED, STATIC, INTERFACE)](#step-7--creating-library-targets-shared-static-interface)
9. [Step 8 — Include Directories & Generator Expressions](#step-8--include-directories--generator-expressions)
10. [Step 9 — Linking Libraries (PUBLIC / PRIVATE / INTERFACE)](#step-9--linking-libraries-public--private--interface)
11. [Step 10 — Creating Executables](#step-10--creating-executables)
12. [Step 11 — Compile Definitions (Preprocessor Macros)](#step-11--compile-definitions-preprocessor-macros)
13. [Step 12 — Installing Targets & Files](#step-12--installing-targets--files)
14. [Step 13 — Exporting Libraries for `find_package()`](#step-13--exporting-libraries-for-find_package)
15. [Step 14 — Testing with GTest & CTest](#step-14--testing-with-gtest--ctest)
16. [Step 15 — ROS 2 / Ament CMake Basics](#step-15--ros-2--ament-cmake-basics)
17. [Step 16 — ROS 2 Message & Service Generation](#step-16--ros-2-message--service-generation)
18. [Step 17 — ROS 2 Component Nodes](#step-17--ros-2-component-nodes)
19. [Step 18 — Advanced: pybind11, CUDA, Conditional Dependencies](#step-18--advanced-pybind11-cuda-conditional-dependencies)
20. [Full Example — Core Library CMakeLists.txt](#full-example--core-library-cmakeliststxt)
21. [Full Example — ROS 2 Wrapper CMakeLists.txt](#full-example--ros-2-wrapper-cmakeliststxt)

---

## 1. Introduction

**CMake** is a build system generator. You write a `CMakeLists.txt` file describing *what* to build, and CMake generates the actual build scripts (Makefiles, Ninja files, etc.) for your platform.

**Build flow:**
```
CMakeLists.txt  →  cmake (configure)  →  Makefile/build.ninja  →  make/ninja (build)  →  binary
```

**Typical commands:**
```bash
mkdir build && cd build
cmake ..                  # Configure
cmake --build .           # Build
cmake --install .         # Install
ctest                     # Run tests
```

---

## Step 1 — Minimum Project Setup

Every CMakeLists.txt file starts with two mandatory commands:

```cmake
# Require at least CMake 3.8 (needed for generator expressions, C++17 support, etc.)
cmake_minimum_required(VERSION 3.8)

# Declare the project name. This sets the ${PROJECT_NAME} variable.
project(my_library)
```

You can also specify languages explicitly:

```cmake
# Only enable C++ and C compilers (skip Fortran, etc.)
project(my_library LANGUAGES CXX C)

# C++ only
project(my_library CXX)
```

**Why this matters:**
- `cmake_minimum_required` ensures users have a compatible CMake version.
- `project()` sets `${PROJECT_NAME}`, which is used everywhere to avoid hardcoding names.
- Specifying `LANGUAGES` avoids unnecessary compiler checks.

**From the repo** (`object_recognition/CMakeLists.txt`):
```cmake
cmake_minimum_required(VERSION 3.8)
project(object_recognition LANGUAGES CXX C)
```

---

## Step 2 — Setting the C++ Standard

```cmake
# Use C++17 features
set(CMAKE_CXX_STANDARD 17)

# Make C++17 a hard requirement — fail if the compiler doesn't support it
set(CMAKE_CXX_STANDARD_REQUIRED ON)
```

**What `set()` does:** Assigns a value to a variable. `CMAKE_CXX_STANDARD` and `CMAKE_CXX_STANDARD_REQUIRED` are built-in CMake variables that control the compiler's C++ standard flag.

| Variable | Effect |
|---|---|
| `CMAKE_CXX_STANDARD 17` | Adds `-std=c++17` (or equivalent) |
| `CMAKE_CXX_STANDARD_REQUIRED ON` | Error if C++17 is unavailable (instead of silently falling back) |

**From the repo** (used in virtually every CMakeLists.txt):
```cmake
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
```

---

## Step 3 — Compiler Detection & Warning Flags

```cmake
# Enable strict warnings only on GCC and Clang (skip MSVC, etc.)
if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()
```

**Breakdown:**
- `CMAKE_COMPILER_IS_GNUCXX` — `TRUE` when using GCC's C++ compiler.
- `CMAKE_CXX_COMPILER_ID MATCHES "Clang"` — Pattern match for Clang (also matches AppleClang).
- `add_compile_options(...)` — Adds flags to ALL targets in this directory.
- `-Wall` — Enable most warnings. `-Wextra` — Extra warnings. `-Wpedantic` — Strict ISO compliance.

You can also add custom warning suppressions:

```cmake
# From odp_common — suppress a specific warning
add_compile_options(-Wall -Wextra -Wpedantic -Wno-class-memaccess)
```

---

## Step 4 — Build Options with `option()`

`option()` defines a boolean ON/OFF switch that users can toggle at configure time.

```cmake
# Define a build option (default OFF)
option(MEASURE_PERF "Enable performance measurements" OFF)

# Use it conditionally
if(MEASURE_PERF)
    add_compile_definitions(MEASURE_PERF)
    message(STATUS "Performance measurement enabled (MEASURE_PERF)")
endif()
```

**Usage from the command line:**
```bash
cmake -DMEASURE_PERF=ON ..
```

**More examples from the repo:**

```cmake
# GPU support toggle (default ON)
option(ORTM_WITH_GPU "Build with GPU/object_recognition support" ON)

# Testing toggle (standard CMake convention)
option(BUILD_TESTING "Build tests" OFF)
```

**Using options for conditional dependencies:**
```cmake
option(ORTM_WITH_GPU "Build with GPU support" ON)

if(ORTM_WITH_GPU)
  find_package(object_recognition_ros REQUIRED)   # Fail if missing
else()
  find_package(object_recognition_ros QUIET)       # Silently skip if missing
endif()
```

**Commands introduced:**
| Command | Purpose |
|---|---|
| `option(NAME "desc" DEFAULT)` | Define a boolean option |
| `message(STATUS "text")` | Print an informational message during configure |
| `add_compile_definitions(DEF)` | Add a preprocessor `#define` to all targets |

---

## Step 5 — Finding Dependencies with `find_package()`

`find_package()` searches for pre-installed libraries and makes their headers and link flags available.

```cmake
# Basic usage — fail if not found
find_package(OpenCV REQUIRED)

# With specific components
find_package(OpenCV REQUIRED COMPONENTS core highgui imgproc videoio)

# With components (different library)
find_package(hsmcpp REQUIRED COMPONENTS std)

# Optional — don't fail if missing
find_package(pybind11 QUIET)
```

**What happens behind the scenes:**
- CMake searches for a file like `OpenCVConfig.cmake` or `FindOpenCV.cmake`.
- If found, it sets variables like `${OpenCV_LIBRARIES}`, `${OpenCV_INCLUDE_DIRS}`.
- Modern packages also create **imported targets** like `fmt::fmt`, `Eigen3::Eigen`, `GTest::gtest_main`.

**Common dependency patterns from the repo:**

```cmake
# Simple libraries
find_package(fmt REQUIRED)          # → fmt::fmt
find_package(yaml-cpp REQUIRED)     # → yaml-cpp
find_package(Eigen3 REQUIRED)       # → Eigen3::Eigen, ${EIGEN3_INCLUDE_DIRS}
find_package(Boost REQUIRED)        # → Boost::boost
find_package(GTest REQUIRED)        # → GTest::gtest_main, GTest::gmock_main
find_package(vpi REQUIRED)          # → vpi

# GPU
find_package(CUDA REQUIRED)         # → ${CUDA_INCLUDE_DIRS}, ${CUDA_LIBRARIES}

# Geographiclib needs a custom module path
set(CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH};/usr/share/cmake/geographiclib")
find_package(GeographicLib REQUIRED)
```

**`CMAKE_MODULE_PATH`** — If a package's `Find*.cmake` file is installed in a non-standard location, you must tell CMake where to look:

```cmake
# Append a custom search path (don't overwrite existing paths!)
set(CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH};/usr/share/cmake/geographiclib")
find_package(GeographicLib REQUIRED)
```

**`REQUIRED` vs `QUIET`:**
| Keyword | Behavior |
|---|---|
| `REQUIRED` | Error and stop if not found |
| `QUIET` | Silently continue; check with `if(package_FOUND)` |
| *(neither)* | Warning but continue |

---

## Step 6 — Finding Dependencies with `pkg-config`

Some libraries (especially Linux system libraries like GStreamer) don't provide CMake config files but do provide `.pc` files for `pkg-config`.

```cmake
# Step 1: Find the PkgConfig CMake module
find_package(PkgConfig REQUIRED)

# Step 2: Use pkg_check_modules to find libraries via pkg-config
pkg_check_modules(GST REQUIRED gstreamer-1.0 gstreamer-app-1.0)
```

This sets variables based on the first argument (`GST`):
| Variable | Content |
|---|---|
| `${GST_INCLUDE_DIRS}` | Header search paths |
| `${GST_LIBRARIES}` | Libraries to link |
| `${GST_CFLAGS_OTHER}` | Additional compiler flags |

**Using the results:**
```cmake
target_include_directories(${PROJECT_NAME}
  PUBLIC ${GST_INCLUDE_DIRS}
)

target_link_libraries(${PROJECT_NAME}
  PUBLIC ${GST_LIBRARIES}
)

# If extra compiler flags are needed
target_compile_options(${PROJECT_NAME} PRIVATE ${GST_CFLAGS_OTHER})
```

**More examples from the repo:**
```cmake
# GStreamer with video support
pkg_check_modules(GST REQUIRED gstreamer-1.0 gstreamer-app-1.0 gstreamer-video-1.0)

# Separate GStreamer modules
pkg_check_modules(GSTREAMER REQUIRED gstreamer-1.0)
pkg_check_modules(GSTREAMER_APP REQUIRED gstreamer-app-1.0)

# hsmcpp via pkg-config
pkg_check_modules(HSMCPP_STD hsmcpp_std REQUIRED)

# TensorRT inference library
pkg_check_modules(INFERENCE_TENSORRT_CPP inference_tensorrt_cpp REQUIRED)
```

---

## Step 7 — Creating Library Targets (SHARED, STATIC, INTERFACE)

CMake supports three types of libraries:

### Shared Library (`.so` / `.dll`)
Most common in this repo — dynamically linked at runtime.

```cmake
add_library(${PROJECT_NAME} SHARED
  src/my_source.cpp
  src/another_source.cpp
)
```

### Static Library (`.a` / `.lib`)
Compiled into the consuming binary at link time.

```cmake
add_library(${PROJECT_NAME} STATIC
  src/shared_functions.cpp
  src/state_manager/base_state_manager.cpp
)

# Static libraries linked into shared libraries need -fPIC
set_target_properties(${PROJECT_NAME} PROPERTIES POSITION_INDEPENDENT_CODE ON)
```

### Interface (Header-Only) Library
No compiled code — just headers. Uses `INTERFACE` keyword everywhere.

```cmake
add_library(${PROJECT_NAME} INTERFACE)
```

**Choosing between them (pattern from `odp_common`):**

```cmake
set(SRC_FILES
    src/shared_functions.cpp
    src/state_manager/base_state_manager.cpp
)

if(SRC_FILES)
    # Has implementation files → STATIC library
    add_library(${PROJECT_NAME} STATIC ${SRC_FILES})
    set_target_properties(${PROJECT_NAME} PROPERTIES POSITION_INDEPENDENT_CODE ON)
    set(INCLUDE_SCOPE PUBLIC)
else()
    # Header-only → INTERFACE library
    add_library(${PROJECT_NAME} INTERFACE)
    set(INCLUDE_SCOPE INTERFACE)
endif()
```

**Using variables for source lists:**
```cmake
set(SRC_FILES
    src/multi_object_tracking.cpp
    src/sort/kalman_box_tracker.cpp
    src/sort/kuhn_munkres.cpp
    src/sort/sort.cpp
)

add_library(${PROJECT_NAME} SHARED ${SRC_FILES})
```

---

## Step 8 — Include Directories & Generator Expressions

### Basic Include Directories

```cmake
target_include_directories(${PROJECT_NAME}
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)
```

### Generator Expressions Explained

Generator expressions (`$<...>`) produce different values depending on the context:

| Expression | When it expands | Typical value |
|---|---|---|
| `$<BUILD_INTERFACE:path>` | When building the project | `/full/path/to/project/include` |
| `$<INSTALL_INTERFACE:path>` | After installation | `include` (relative to install prefix) |

**Why you need both:**
- When building locally, other targets need the full path to your source `include/` directory.
- After installation, the headers are at `<prefix>/include`, and consumers need that relative path.

```cmake
target_include_directories(${PROJECT_NAME}
  PUBLIC
    # During build: use source tree headers
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    # After install: use installed headers
    $<INSTALL_INTERFACE:include>
    # External dependency headers (always needed)
    ${EIGEN3_INCLUDE_DIRS}
    ${GeographicLib_INCLUDE_DIRS}
)
```

### Scope keywords (PUBLIC / PRIVATE / INTERFACE)

| Scope | Who sees it? |
|---|---|
| `PUBLIC` | This target AND anything that links to it |
| `PRIVATE` | Only this target |
| `INTERFACE` | Only targets that link to this one (not this target itself) |

```cmake
target_include_directories(${PROJECT_NAME}
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>  # Everyone needs our headers
  PRIVATE
    ${HSMCPP_STD_INCLUDE_DIRS}  # Only we use hsmcpp internally
)
```

### Legacy `include_directories()` (avoid in new code)

```cmake
# Global — affects ALL targets in this directory. Use target_include_directories instead.
include_directories(${GST_INCLUDE_DIRS})
```

---

## Step 9 — Linking Libraries (PUBLIC / PRIVATE / INTERFACE)

```cmake
target_link_libraries(${PROJECT_NAME}
  PUBLIC
    odp_common::odp_common    # Our consumers also need odp_common
    ${OpenCV_LIBRARIES}       # OpenCV types in our public API
    ${GST_LIBRARIES}          # GStreamer in our public API
  PRIVATE
    tracing::tracing          # Only used internally
    fmt::fmt                  # Only used in our .cpp files
)
```

### Scope Rules (same as include directories)

| If your public headers `#include` a dependency's headers → | `PUBLIC` |
|---|---|
| If only your `.cpp` files use the dependency → | `PRIVATE` |
| If you're an INTERFACE library forwarding a dependency → | `INTERFACE` |

### Namespaced Targets

Modern CMake packages export **namespaced imported targets**. These are the preferred way to link:

```cmake
# Namespaced targets (preferred — CMake will error if the package isn't found)
target_link_libraries(${PROJECT_NAME}
  PUBLIC
    odp_common::odp_common     # namespace::target
    dmm::video_handler
    hlpm::global_path_planning
    ortm::object_recognition
    smm::system_state_manager
    fmt::fmt
    Eigen3::Eigen
    GTest::gtest_main
    GTest::gmock_main
)
```

**Advantage:** If you typo `odp_common::odp_commo`, CMake immediately errors. With plain `odp_common`, it silently becomes a linker flag `-lodp_common` and you get a cryptic linker error later.

### Mixing styles

```cmake
# OK to mix namespaced targets with variable-based libraries
target_link_libraries(${PROJECT_NAME}
  PUBLIC
    ortm::object_recognition           # Namespaced imported target
    ${OpenCV_LIBS}                      # Variable from find_package
    ${INFERENCE_TENSORRT_CPP_LIBRARIES} # Variable from pkg_check_modules
    ${CUDA_LIBRARIES}                   # Variable from find_package(CUDA)
    ${CMAKE_THREAD_LIBS_INIT}           # Threading library
)
```

---

## Step 10 — Creating Executables

Executables are typically test binaries in this repo:

```cmake
add_executable(tests
  ${TEST_SOURCES}
)

# Or with explicit source files
add_executable(${PROJECT_NAME} 
  ${CMAKE_CURRENT_SOURCE_DIR}/test_object_recognition.cpp
)
```

Executables support the same `target_include_directories()` and `target_link_libraries()` as libraries.

---

## Step 11 — Compile Definitions (Preprocessor Macros)

### Per-target definitions

```cmake
# Add a #define visible to the target's source files
target_compile_definitions(tests PRIVATE
  "CONFIG_DIR=\"${CMAKE_CURRENT_SOURCE_DIR}/config\""
)

# PUBLIC — also visible to consumers
target_compile_definitions(${PROJECT_NAME}_test PUBLIC
  "PARAMS_DIR=\"${CMAKE_CURRENT_SOURCE_DIR}/config\""
)
```

In your C++ code:
```cpp
// CONFIG_DIR is replaced by the actual path at compile time
std::string config_path = CONFIG_DIR;
```

### Global definitions

```cmake
# Add a #define to ALL targets in this directory
add_compile_definitions(MEASURE_PERF)
```

---

## Step 12 — Installing Targets & Files

Installation copies your build artifacts to a destination (default `/usr/local`).

### Installing Targets

```cmake
install(
  TARGETS ${PROJECT_NAME}
  EXPORT ${PROJECT_NAME}_targets    # Save target info for find_package (see Step 13)
  ARCHIVE DESTINATION lib           # Static libraries (.a)
  LIBRARY DESTINATION lib           # Shared libraries (.so)
  RUNTIME DESTINATION bin           # Executables
)
```

| Keyword | What gets installed |
|---|---|
| `ARCHIVE` | Static libraries (`.a`), import libraries (`.lib` on Windows) |
| `LIBRARY` | Shared libraries (`.so`) |
| `RUNTIME` | Executables and DLLs |
| `EXPORT` | Record this target in an export set (for `find_package()` support) |

### Installing Directories

```cmake
# Install all headers
install(DIRECTORY include/ DESTINATION include)

# Install only .hpp files
install(DIRECTORY include/ DESTINATION include
  FILES_MATCHING
  PATTERN "*.hpp"
)

# Install config files
install(DIRECTORY config DESTINATION share/${PROJECT_NAME})

# Install launch files (ROS 2)
install(DIRECTORY launch DESTINATION share/${PROJECT_NAME})

# Install service definitions
install(DIRECTORY srv DESTINATION share/${PROJECT_NAME})
```

**Note the trailing `/`:**
- `install(DIRECTORY include/ ...)` — Installs the *contents* of `include/` (the headers).
- `install(DIRECTORY include ...)` — Installs the `include` directory *itself* (creating `include/include/`).

---

## Step 13 — Exporting Libraries for `find_package()`

This is what makes `find_package(my_library REQUIRED)` work for downstream consumers.

### Step-by-step

```cmake
# 1. Install the library target and register it in an export set
install(
  TARGETS ${PROJECT_NAME}
  EXPORT ${PROJECT_NAME}_targets
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
)

# 2. Install headers
install(DIRECTORY include/ DESTINATION include)

# 3. Generate and install a Config.cmake file from the export set
install(
  EXPORT ${PROJECT_NAME}_targets
  FILE ${PROJECT_NAME}Config.cmake    # The file find_package() will look for
  NAMESPACE dmm::                      # Prefix for imported targets
  DESTINATION share/${PROJECT_NAME}/cmake
)
```

**After installation, a consumer can do:**
```cmake
find_package(dm_state_manager REQUIRED)
target_link_libraries(my_app PRIVATE dmm::dm_state_manager)
```

### Namespace conventions from the repo

| Module | Namespace | Example target |
|---|---|---|
| DMM (Drone Mission Module) | `dmm::` | `dmm::dm_state_manager`, `dmm::video_handler` |
| HLPM (High Level Path Module) | `hlpm::` | `hlpm::global_path_planning`, `hlpm::path_translator` |
| ORTM (Object Recognition & Tracking) | `ortm::` | `ortm::object_recognition`, `ortm::multi_object_tracking` |
| SMM (System Health Module) | `smm::` | `smm::system_state_manager` |
| ODP Common | `odp_common::` | `odp_common::odp_common` |
| IPM (Image Processing Module) | `ipm::` | `ipm::image_streaming` |

---

## Step 14 — Testing with GTest & CTest

### Pattern 1: Standalone CMake project (core libraries)

```cmake
if(BUILD_TESTING)
  # Enable CMake's testing subsystem
  enable_testing()

  # Find GoogleTest
  find_package(GTest REQUIRED)

  # Glob all test source files
  file(GLOB TEST_SOURCES "test/*.cpp")

  # Create a test executable
  add_executable(tests ${TEST_SOURCES})

  # Pass config path as a compile-time constant
  target_compile_definitions(tests PRIVATE
    "CONFIG_DIR=\"${CMAKE_CURRENT_SOURCE_DIR}/config\""
  )

  # Link the library under test + GTest
  target_link_libraries(tests
    PRIVATE
    ${PROJECT_NAME}
    GTest::gtest_main
  )

  # Auto-discover and register all TEST_F / TEST cases
  gtest_discover_tests(tests)
endif()
```

### Pattern 2: Subdirectory tests

```cmake
# In the parent CMakeLists.txt
if(BUILD_TESTING)
  enable_testing()
  add_subdirectory(test)
endif()

# In test/CMakeLists.txt
project(test_multi_object_tracking LANGUAGES CXX C)

find_package(GTest REQUIRED)
find_package(OpenCV REQUIRED)

add_executable(${PROJECT_NAME} test_multi_object_tracking.cpp)

target_link_libraries(${PROJECT_NAME}
  PUBLIC
    GTest::gtest_main
    GTest::gmock_main
    multi_object_tracking
    ${OpenCV_LIBS}
)

# Register tests by name (alternative to gtest_discover_tests)
gtest_add_tests(
    TARGET ${PROJECT_NAME}
    TEST_LIST MultiObjectTrackingTest
)
```

### `gtest_discover_tests` vs `gtest_add_tests`

| Command | How it works | When to use |
|---|---|---|
| `gtest_discover_tests(target)` | Runs the binary at *test time* to discover tests | Preferred — always up-to-date |
| `gtest_add_tests(TARGET t TEST_LIST name)` | Parses source code at *configure time* | When you need a named test list |

### `file(GLOB ...)` for test sources

```cmake
# Collect all .cpp files in test/ into a list
file(GLOB TEST_SOURCES "test/*.cpp")
```

**Note:** `file(GLOB)` is evaluated at configure time. If you add new test files, you must re-run `cmake ..` to pick them up. For production source files, prefer listing them explicitly.

### Using `include(CTest)`

```cmake
# Alternative to enable_testing() — also enables CDash integration
include(CTest)
```

---

## Step 15 — ROS 2 / Ament CMake Basics

ROS 2 packages use `ament_cmake` instead of plain CMake. It wraps standard CMake with ROS-specific macros.

### Minimal ROS 2 package (launch/config only)

```cmake
cmake_minimum_required(VERSION 3.8)
project(hlpm_bringup_ros)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# The ament_cmake build system
find_package(ament_cmake REQUIRED)

# Install launch and config directories
install(DIRECTORY launch DESTINATION share/${PROJECT_NAME})
install(DIRECTORY config DESTINATION share/${PROJECT_NAME})

# MUST be the last command — finalizes the package
ament_package()
```

### `ament_target_dependencies()` — Link ROS 2 Packages

This is the ROS 2 way of linking dependencies. It handles include paths, libraries, and transitive dependencies.

```cmake
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(rclcpp_components REQUIRED)
find_package(sensor_msgs REQUIRED)

add_library(${PROJECT_NAME} SHARED src/my_node.cpp)

# ROS 2 dependencies — use ament_target_dependencies instead of
# target_link_libraries for ROS packages
ament_target_dependencies(${PROJECT_NAME}
  PUBLIC
  rclcpp
  rclcpp_components
  sensor_msgs
)

# Non-ROS dependencies — use standard target_link_libraries
target_link_libraries(${PROJECT_NAME}
  PUBLIC
    dmm::video_handler
    odp_common::odp_common
  PRIVATE
    tracing::tracing
)
```

### Ament Export Functions

These make your package visible to other ROS 2 packages:

```cmake
# Export include directories
ament_export_include_directories(include)

# Export library names
ament_export_libraries(${PROJECT_NAME})

# Export CMake targets (modern approach — preferred)
ament_export_targets(${PROJECT_NAME}Targets HAS_LIBRARY_TARGET)

# Export transitive dependencies
ament_export_dependencies(
  rclcpp
  sensor_msgs
  OpenCV
)

# MUST be last
ament_package()
```

| Function | Purpose |
|---|---|
| `ament_export_include_directories(include)` | Make headers findable |
| `ament_export_libraries(lib)` | Export library name |
| `ament_export_targets(T HAS_LIBRARY_TARGET)` | Export CMake targets (modern) |
| `ament_export_dependencies(...)` | Propagate find_package requirements |
| `ament_package()` | Finalize — **must be last** |

### Ament Testing

```cmake
if(BUILD_TESTING)
  find_package(ament_cmake_gtest REQUIRED)

  ament_add_gtest(${PROJECT_NAME}_test test/test_my_node.cpp)

  target_compile_definitions(${PROJECT_NAME}_test PUBLIC
    "PARAMS_DIR=\"${CMAKE_CURRENT_SOURCE_DIR}/config\""
  )

  target_include_directories(${PROJECT_NAME}_test PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
  )

  ament_target_dependencies(${PROJECT_NAME}_test
    yaml-cpp
  )

  target_link_libraries(${PROJECT_NAME}_test
    ${PROJECT_NAME}
    fmt::fmt
    yaml-cpp
  )
endif()
```

### Ament Lint (automated code quality)

```cmake
if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  ament_lint_auto_find_test_dependencies()
endif()
```

This auto-discovers and runs all linters configured in your `package.xml`.

---

## Step 16 — ROS 2 Message & Service Generation

### Pure message package (no C++ code)

```cmake
cmake_minimum_required(VERSION 3.8)
project(drone_mission_logger_msgs)

find_package(ament_cmake REQUIRED)
find_package(builtin_interfaces REQUIRED)
find_package(rosidl_default_generators REQUIRED)
find_package(std_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)

# Generate C++/Python bindings from .msg and .srv files
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/FlightSession.msg"
  "msg/EnrichedImage.msg"
  "srv/LoadMission.srv"
  "srv/Configure.srv"
  "srv/StartSession.srv"
  DEPENDENCIES builtin_interfaces std_msgs geometry_msgs
)

ament_package()
```

### Using a set variable for message lists

```cmake
set(msg_files
  msg/ObjectTrack2D.msg
  msg/ObjectTrack2DArray.msg
  msg/ObjectTrack3D.msg
  msg/ObjectTrack3DArray.msg
)

rosidl_generate_interfaces(${PROJECT_NAME}
  ${msg_files}
  DEPENDENCIES std_msgs geometry_msgs geographic_msgs vision_msgs builtin_interfaces
)
```

### Generating interfaces AND having a library in the same package

When a package both generates messages AND has a C++ library, you need to link the type support:

```cmake
# 1. Generate the interfaces
rosidl_generate_interfaces(${PROJECT_NAME}
  "srv/StartRecording.srv"
  "srv/StopRecording.srv"
)

# 2. Create your library with a DIFFERENT target name
add_library(${PROJECT_NAME}_component SHARED src/my_node.cpp)

# 3. Get the generated type support target
rosidl_get_typesupport_target(cpp_typesupport_target ${PROJECT_NAME} "rosidl_typesupport_cpp")

# 4. Link it to your library
target_link_libraries(${PROJECT_NAME}_component PUBLIC "${cpp_typesupport_target}")
```

### Interface library for type support (header-only wrapper)

```cmake
rosidl_get_typesupport_target(cpp_typesupport_target "${PROJECT_NAME}" "rosidl_typesupport_cpp")

if(cpp_typesupport_target)
  # Create a header-only library that bundles the type support
  add_library(${PROJECT_NAME}_library INTERFACE)
  
  target_include_directories(${PROJECT_NAME}_library INTERFACE
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
    "$<INSTALL_INTERFACE:include>")
  
  target_link_libraries(${PROJECT_NAME}_library INTERFACE
    "${cpp_typesupport_target}")

  install(TARGETS ${PROJECT_NAME}_library EXPORT export_${PROJECT_NAME})
endif()

ament_export_targets(export_${PROJECT_NAME} HAS_LIBRARY_TARGET)
ament_export_dependencies(rosidl_default_runtime)
```

---

## Step 17 — ROS 2 Component Nodes

ROS 2 supports **component nodes** — shared libraries that can be loaded into a container process at runtime.

### Registering multiple components (list of class names)

```cmake
rclcpp_components_register_nodes(${PROJECT_NAME}
  "dmm::DmStateManagerROS"
)
```

This registers the node class so it can be loaded with:
```bash
ros2 component load /ComponentManager my_package dmm::DmStateManagerROS
```

### Registering with an auto-generated executable

```cmake
rclcpp_components_register_node(${PROJECT_NAME}
  PLUGIN "ipm::ImageStreamingROS"
  EXECUTABLE image_streaming_ros_exe
)
```

This does two things:
1. Registers the component for dynamic loading.
2. Creates a standalone executable `image_streaming_ros_exe` that you can run directly.

---

## Step 18 — Advanced: pybind11, CUDA, Conditional Dependencies

### pybind11 Python Bindings

```cmake
if(BUILD_TESTING)
  # Only build Python bindings if pybind11 is available
  find_package(pybind11 QUIET)
  if(pybind11_FOUND)
    # Create a Python module from C++ code
    pybind11_add_module(terminal_guidance_py
      test/visualize/bind_terminal_guidance.cpp
    )

    target_link_libraries(terminal_guidance_py
      PRIVATE
      ${PROJECT_NAME}
    )

    install(
      TARGETS terminal_guidance_py
      LIBRARY DESTINATION lib
    )
  endif()
endif()
```

### CUDA Support

```cmake
# Set the CUDA toolkit location
set(CUDA_TOOLKIT_ROOT_DIR /usr/local/cuda)

# Find CUDA
find_package(CUDA REQUIRED)

# Use CUDA variables
target_include_directories(${PROJECT_NAME}
  PUBLIC ${CUDA_INCLUDE_DIRS}
)

target_link_libraries(${PROJECT_NAME}
  PUBLIC ${CUDA_LIBRARIES}
)
```

### Conditional Dependency Pattern

```cmake
option(ORTM_WITH_GPU "Build with GPU support" ON)

if(ORTM_WITH_GPU)
  find_package(object_recognition_ros REQUIRED)   # Must be present
else()
  find_package(object_recognition_ros QUIET)       # Optional
endif()
```

---

## Full Example — Core Library CMakeLists.txt

This example combines all core CMake concepts from the repo into a single, fully commented CMakeLists.txt for a hypothetical shared library:

```cmake
# ============================================================================
# FULL EXAMPLE: Core Library CMakeLists.txt
# Combines every CMake concept used in the repository's core libraries.
# ============================================================================

# ---------------------------------------------------------------------------
# 1. Minimum CMake version
#    VERSION 3.8 needed for: generator expressions, C++17, GTest integration
# ---------------------------------------------------------------------------
cmake_minimum_required(VERSION 3.8)

# ---------------------------------------------------------------------------
# 2. Project declaration
#    LANGUAGES restricts compiler checks to C++ and C only.
#    Sets ${PROJECT_NAME} = "my_core_library"
# ---------------------------------------------------------------------------
project(my_core_library LANGUAGES CXX C)

# ---------------------------------------------------------------------------
# 3. C++ standard
# ---------------------------------------------------------------------------
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ---------------------------------------------------------------------------
# 4. Build options — toggled with -DMEASURE_PERF=ON, -DBUILD_TESTING=ON
# ---------------------------------------------------------------------------
option(MEASURE_PERF "Enable performance measurements" OFF)
option(BUILD_TESTING "Build tests" OFF)

if(MEASURE_PERF)
    add_compile_definitions(MEASURE_PERF)
    message(STATUS "Performance measurement enabled (MEASURE_PERF)")
endif()

# ---------------------------------------------------------------------------
# 5. Compiler-specific warning flags
#    Only apply to GCC and Clang — skip MSVC, Intel, etc.
# ---------------------------------------------------------------------------
if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# ---------------------------------------------------------------------------
# 6. Find dependencies
# ---------------------------------------------------------------------------

# 6a. Standard CMake packages
find_package(OpenCV REQUIRED COMPONENTS core highgui imgproc videoio)
find_package(Eigen3 REQUIRED)
find_package(fmt REQUIRED)
find_package(yaml-cpp REQUIRED)

# 6b. Packages with non-standard CMake module paths
set(CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH};/usr/share/cmake/geographiclib")
find_package(GeographicLib REQUIRED)

# 6c. CUDA (GPU support)
set(CUDA_TOOLKIT_ROOT_DIR /usr/local/cuda)
find_package(CUDA REQUIRED)

# 6d. pkg-config based packages (no CMake config files)
find_package(PkgConfig REQUIRED)
pkg_check_modules(GST REQUIRED gstreamer-1.0 gstreamer-app-1.0)
pkg_check_modules(INFERENCE_TENSORRT_CPP inference_tensorrt_cpp REQUIRED)

# 6e. Internal project dependency (installed with install(EXPORT ...))
find_package(odp_common REQUIRED)

# 6f. Hierarchical state machine library (with components)
find_package(hsmcpp REQUIRED COMPONENTS std)

# ---------------------------------------------------------------------------
# 7. Define sources
# ---------------------------------------------------------------------------
set(SRC_FILES
    src/my_core_library.cpp
    src/module_a/feature_a.cpp
    src/module_b/feature_b.cpp
)

# ---------------------------------------------------------------------------
# 8. Create the library target
#    SHARED → produces a .so file on Linux
# ---------------------------------------------------------------------------
add_library(${PROJECT_NAME} SHARED ${SRC_FILES})

# ---------------------------------------------------------------------------
# 9. Include directories with generator expressions
#    PUBLIC  = this target + consumers
#    PRIVATE = this target only
# ---------------------------------------------------------------------------
target_include_directories(${PROJECT_NAME}
  PUBLIC
    # During build: point to source tree headers
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    # After install: point to installed headers
    $<INSTALL_INTERFACE:include>
    # External dependency headers needed by our public API
    ${OpenCV_INCLUDE_DIRS}
    ${EIGEN3_INCLUDE_DIRS}
    ${GeographicLib_INCLUDE_DIRS}
  PRIVATE
    # Internal-only headers
    ${CUDA_INCLUDE_DIRS}
    ${GST_INCLUDE_DIRS}
    ${INFERENCE_TENSORRT_CPP_INCLUDE_DIRS}
)

# ---------------------------------------------------------------------------
# 10. Link libraries
#     PUBLIC  = dependency is part of our public API
#     PRIVATE = dependency is an implementation detail
# ---------------------------------------------------------------------------
target_link_libraries(${PROJECT_NAME}
  PUBLIC
    # Namespaced imported targets (preferred)
    odp_common::odp_common
    Eigen3::Eigen
    fmt::fmt
    yaml-cpp
    # Variable-based libraries
    ${OpenCV_LIBRARIES}
    ${GeographicLib_LIBRARIES}
  PRIVATE
    ${CUDA_LIBRARIES}
    ${GST_LIBRARIES}
    ${INFERENCE_TENSORRT_CPP_LIBRARIES}
    ${CMAKE_THREAD_LIBS_INIT}
)

# ---------------------------------------------------------------------------
# 11. Install the library
# ---------------------------------------------------------------------------

# 11a. Install the binary (and register it in an export set)
install(
  TARGETS ${PROJECT_NAME}
  EXPORT ${PROJECT_NAME}_targets
  ARCHIVE DESTINATION lib     # .a files
  LIBRARY DESTINATION lib     # .so files
  RUNTIME DESTINATION bin     # executables / .dll
)

# 11b. Install public headers
install(
  DIRECTORY include/
  DESTINATION include
)

# 11c. Install config/data files
install(
  DIRECTORY config
  DESTINATION share/${PROJECT_NAME}
)

# ---------------------------------------------------------------------------
# 12. Export for find_package()
#     After this, other projects can do:
#       find_package(my_core_library REQUIRED)
#       target_link_libraries(app PRIVATE myns::my_core_library)
# ---------------------------------------------------------------------------
install(
  EXPORT ${PROJECT_NAME}_targets
  FILE ${PROJECT_NAME}Config.cmake
  NAMESPACE myns::
  DESTINATION share/${PROJECT_NAME}/cmake
)

# ---------------------------------------------------------------------------
# 13. Testing
# ---------------------------------------------------------------------------
if(BUILD_TESTING)
  enable_testing()
  find_package(GTest REQUIRED)

  # Glob test sources (re-run cmake when adding new test files)
  file(GLOB TEST_SOURCES "test/*.cpp")

  add_executable(tests ${TEST_SOURCES})

  # Pass compile-time config path
  target_compile_definitions(tests PRIVATE
    "CONFIG_DIR=\"${CMAKE_CURRENT_SOURCE_DIR}/config\""
  )

  target_link_libraries(tests
    PRIVATE
    ${PROJECT_NAME}
    GTest::gtest_main
  )

  # Auto-discover TEST_F and TEST macros in the binary
  gtest_discover_tests(tests)

  # --- Optional: Python bindings for visualization ---
  find_package(pybind11 QUIET)
  if(pybind11_FOUND)
    pybind11_add_module(my_library_py
      test/visualize/bindings.cpp
    )
    target_link_libraries(my_library_py PRIVATE ${PROJECT_NAME})
    install(TARGETS my_library_py LIBRARY DESTINATION lib)
  endif()
endif()
```

---

## Full Example — ROS 2 Wrapper CMakeLists.txt

This example combines all ROS 2 / ament_cmake concepts into a single CMakeLists.txt for a ROS 2 component node package that also generates custom service definitions:

```cmake
# ============================================================================
# FULL EXAMPLE: ROS 2 Component Node with Service Generation
# Combines every ament_cmake concept used in the repository.
# ============================================================================

# ---------------------------------------------------------------------------
# 1. Project setup
# ---------------------------------------------------------------------------
cmake_minimum_required(VERSION 3.8)
project(my_node_ros)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# ---------------------------------------------------------------------------
# 2. Build options
# ---------------------------------------------------------------------------
option(WITH_GPU "Build with GPU support" ON)

# ---------------------------------------------------------------------------
# 3. Find dependencies
# ---------------------------------------------------------------------------

# 3a. ROS 2 / ament
find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)
find_package(rclcpp REQUIRED)
find_package(rclcpp_components REQUIRED)

# 3b. ROS 2 message packages
find_package(std_msgs REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(vision_msgs REQUIRED)
find_package(message_filters REQUIRED)
find_package(cv_bridge REQUIRED)
find_package(mavros_msgs REQUIRED)
find_package(odp_common_ros REQUIRED)

# 3c. External libraries
find_package(OpenCV REQUIRED COMPONENTS core highgui imgproc videoio)
find_package(Boost REQUIRED)
find_package(fmt REQUIRED)
find_package(yaml-cpp REQUIRED)
find_package(tracing REQUIRED)

# 3d. Internal core library
find_package(my_core_library REQUIRED)

# 3e. Conditional GPU dependency
if(WITH_GPU)
  find_package(object_recognition_ros REQUIRED)
else()
  find_package(object_recognition_ros QUIET)
endif()

# ---------------------------------------------------------------------------
# 4. Generate ROS 2 service interfaces
#    Creates C++ and Python bindings from .srv files
# ---------------------------------------------------------------------------
rosidl_generate_interfaces(${PROJECT_NAME}
  "srv/StartRecording.srv"
  "srv/StopRecording.srv"
)

# ---------------------------------------------------------------------------
# 5. Build the component library
#    NOTE: Use a different target name (${PROJECT_NAME}_component) because
#    ${PROJECT_NAME} is taken by rosidl_generate_interfaces above.
# ---------------------------------------------------------------------------
add_library(${PROJECT_NAME}_component SHARED
  src/my_node_ros.cpp
)

target_include_directories(${PROJECT_NAME}_component
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)

# ---------------------------------------------------------------------------
# 6. Link ROS 2 dependencies with ament_target_dependencies
# ---------------------------------------------------------------------------
ament_target_dependencies(${PROJECT_NAME}_component
  PUBLIC
  rclcpp
  rclcpp_components
  message_filters
  std_msgs
  sensor_msgs
  vision_msgs
  cv_bridge
  mavros_msgs
  OpenCV
  Boost
  odp_common_ros
)

# ---------------------------------------------------------------------------
# 7. Link non-ROS libraries with standard target_link_libraries
# ---------------------------------------------------------------------------
target_link_libraries(${PROJECT_NAME}_component
  PUBLIC
    myns::my_core_library
    odp_common::odp_common
  PRIVATE
    tracing::tracing
)

# ---------------------------------------------------------------------------
# 8. Link the generated service type support
# ---------------------------------------------------------------------------
rosidl_get_typesupport_target(cpp_typesupport_target
  ${PROJECT_NAME} "rosidl_typesupport_cpp"
)
target_link_libraries(${PROJECT_NAME}_component
  PUBLIC "${cpp_typesupport_target}"
)

# ---------------------------------------------------------------------------
# 9. Register as a ROS 2 component node
#    Option A: register_nodes — for manual loading into a component container
#    Option B: register_node — also creates a standalone executable
# ---------------------------------------------------------------------------

# Option A: Component-only registration
rclcpp_components_register_nodes(${PROJECT_NAME}_component
  "myns::MyNodeROS"
)

# Option B (alternative): Component + standalone executable
# rclcpp_components_register_node(${PROJECT_NAME}_component
#   PLUGIN "myns::MyNodeROS"
#   EXECUTABLE my_node_ros_exe
# )

# ---------------------------------------------------------------------------
# 10. Install
# ---------------------------------------------------------------------------

# 10a. Library binary
install(TARGETS ${PROJECT_NAME}_component
  EXPORT ${PROJECT_NAME}Targets
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
)

# 10b. Headers (only .hpp files)
install(DIRECTORY include/ DESTINATION include
  FILES_MATCHING
  PATTERN "*.hpp"
)

# 10c. ROS-specific directories
install(DIRECTORY launch DESTINATION share/${PROJECT_NAME})
install(DIRECTORY config DESTINATION share/${PROJECT_NAME})
install(DIRECTORY srv DESTINATION share/${PROJECT_NAME})

# ---------------------------------------------------------------------------
# 11. Testing
# ---------------------------------------------------------------------------
if(BUILD_TESTING)
  find_package(ament_cmake_gtest REQUIRED)

  ament_add_gtest(${PROJECT_NAME}_test test/test_my_node_ros.cpp)

  target_compile_definitions(${PROJECT_NAME}_test PUBLIC
    "PARAMS_DIR=\"${CMAKE_CURRENT_SOURCE_DIR}/test\""
  )

  target_include_directories(${PROJECT_NAME}_test PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
  )

  ament_target_dependencies(${PROJECT_NAME}_test
    std_msgs
    sensor_msgs
    OpenCV
    cv_bridge
  )

  target_link_libraries(${PROJECT_NAME}_test
    ${PROJECT_NAME}_component
    fmt::fmt
    yaml-cpp
  )

  # Alternatively, use ament_lint_auto for automated linting:
  # find_package(ament_lint_auto REQUIRED)
  # ament_lint_auto_find_test_dependencies()
endif()

# ---------------------------------------------------------------------------
# 12. Ament exports — make this package findable by other ROS 2 packages
# ---------------------------------------------------------------------------

# Export CMake targets (modern approach)
ament_export_targets(${PROJECT_NAME}Targets HAS_LIBRARY_TARGET)

# Export transitive dependencies so consumers don't need to find them manually
ament_export_dependencies(
  rosidl_default_runtime
  rclcpp
  rclcpp_components
  message_filters
  sensor_msgs
  vision_msgs
  OpenCV
  cv_bridge
  Boost
)

# ---------------------------------------------------------------------------
# 13. Finalize the ament package — MUST be the very last command
# ---------------------------------------------------------------------------
ament_package()
```

---

## Quick Reference Card

| Task | Command |
|---|---|
| Set CMake minimum version | `cmake_minimum_required(VERSION 3.8)` |
| Declare project | `project(name LANGUAGES CXX C)` |
| Set C++ standard | `set(CMAKE_CXX_STANDARD 17)` |
| Add compiler warnings | `add_compile_options(-Wall -Wextra -Wpedantic)` |
| Define a build option | `option(NAME "desc" OFF)` |
| Find a package | `find_package(Foo REQUIRED)` |
| Find via pkg-config | `pkg_check_modules(PREFIX REQUIRED pkg-name)` |
| Create shared library | `add_library(name SHARED src.cpp)` |
| Create static library | `add_library(name STATIC src.cpp)` |
| Create header-only library | `add_library(name INTERFACE)` |
| Create executable | `add_executable(name src.cpp)` |
| Set include dirs | `target_include_directories(name PUBLIC dir)` |
| Link libraries | `target_link_libraries(name PUBLIC lib)` |
| Add preprocessor define | `target_compile_definitions(name PRIVATE "KEY=val")` |
| Install targets | `install(TARGETS name LIBRARY DESTINATION lib)` |
| Install headers | `install(DIRECTORY include/ DESTINATION include)` |
| Export for find_package | `install(EXPORT set FILE Config.cmake NAMESPACE ns::)` |
| Enable tests | `enable_testing()` / `include(CTest)` |
| Discover GTests | `gtest_discover_tests(target)` |
| ROS 2: link deps | `ament_target_dependencies(target dep1 dep2)` |
| ROS 2: generate msgs | `rosidl_generate_interfaces(pkg "msg/Foo.msg")` |
| ROS 2: register component | `rclcpp_components_register_nodes(lib "ns::Class")` |
| ROS 2: finalize package | `ament_package()` |

---

*This tutorial was generated by analyzing all CMakeLists.txt files in the `nestos.edge.uxv.uxv-main` repository.*
