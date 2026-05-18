# Dependency Management Guideline for a Large Multi-Team ROS 2 Project

## 1. Purpose

This document defines how dependencies should be managed in a large ROS 2 development project where:

- The system is divided into multiple modules.
- Each module is maintained by a separate team.
- Each module lives in its own source repository.
- A main integration repository pulls all module repositories together for building, testing, and container image creation.
- ROS 2 packages inside modules may depend on:
  - Common ROS 2 or system libraries installable through `rosdep` / `apt`.
  - Third-party C++ libraries that are not available through `rosdep` or `apt`.
  - Custom internal libraries that are part of the ROS 2 project itself.

The goal is to make dependency ownership clear, keep modules self-describing, support reproducible container builds, and avoid hidden dependencies in team-owned packages.

---

## 2. Project Model

The project uses a multi-repository architecture.

Example structure:

```text
main_repo/
  Dockerfile
  project.repos
  rosdep/
    company.yaml
  third_party/
    install_dependencies.sh
    install_my_cpp_lib.sh
    install_other_cpp_lib.sh
  scripts/
    build_workspace.sh

src/
  vision_module/
    vision_detector_pkg/
    vision_tracking_pkg/

  planning_module/
    route_planner_pkg/
    motion_planner_pkg/

  control_module/
    controller_pkg/

  common_module/
    geometry_utils_pkg/
    math_utils_pkg/
```

The source layout above may exist inside the Docker build context or CI workspace after the main repository imports all module repositories.

The main repository is responsible for:

- Selecting which module repositories are included in the build.
- Pinning module versions, branches, or commits.
- Building the final development or runtime Docker image.
- Running `rosdep` and `colcon` over the integrated workspace.
- Providing installation scripts for third-party libraries that are not available through `rosdep` or `apt`.

Each module repository is responsible for:

- Maintaining its own ROS 2 packages.
- Declaring direct dependencies in each package's `package.xml`.
- Ensuring its packages build correctly when dependencies are available.
- Avoiding hidden assumptions about libraries being installed globally unless those dependencies are documented and declared.

---

## 3. Core Principle

Every ROS 2 package should declare what it directly depends on.

The package should answer:

```text
What do I need to build and run?
```

The main repository and Docker image should answer:

```text
How are those dependencies provided in this product build?
```

This separation is important.

A package should not silently assume that a library exists in the Docker image. Even if the Dockerfile installs a library manually, the package should still declare the dependency in `package.xml` so that developers, CI, and other teams can understand the dependency graph.

---

## 4. Dependency Categories

There are three main dependency categories.

| Category | Example | How to handle |
|---|---|---|
| Common ROS 2 or system libraries available through `rosdep` / `apt` | `rclcpp`, `sensor_msgs`, `OpenCV`, `Eigen`, `PCL`, `Boost` | Declare in `package.xml`; install using `rosdep` |
| Third-party C++ libraries not available through `rosdep` / `apt` | Vendor SDK, custom fork of an open-source library, special optimization library | Install in Docker using scripts under `third_party/`; declare in `package.xml`; skip key in `rosdep` if needed |
| Custom internal libraries that are part of the ROS 2 project | `geometry_utils`, `vehicle_model`, `common_math` | Make them normal ROS 2 packages and depend on them through `package.xml` |

---

# 5. Case 1: Common Libraries Installable Through `rosdep` / `apt`

## 5.1 Description

This is the standard and preferred case.

If a dependency can be resolved by `rosdep`, each package should declare it in `package.xml`. The Docker build or CI pipeline then runs `rosdep install` across the full workspace.

Examples:

- ROS 2 packages: `rclcpp`, `sensor_msgs`, `std_msgs`, `tf2_ros`, `nav_msgs`.
- Common system libraries: OpenCV, Eigen, Boost, PCL, yaml-cpp, fmt.

## 5.2 Package Example

Suppose a vision package uses OpenCV and ROS image transport.

Example package:

```text
vision_module/
  vision_detector_pkg/
    package.xml
    CMakeLists.txt
    src/
      vision_detector_node.cpp
```

`package.xml`:

```xml
<?xml version="1.0"?>
<package format="3">
  <name>vision_detector_pkg</name>
  <version>0.1.0</version>
  <description>Vision detector package</description>

  <maintainer email="vision-team@example.com">Vision Team</maintainer>
  <license>Proprietary</license>

  <buildtool_depend>ament_cmake</buildtool_depend>

  <depend>rclcpp</depend>
  <depend>sensor_msgs</depend>
  <depend>cv_bridge</depend>
  <depend>image_transport</depend>
  <depend>opencv</depend>

  <test_depend>ament_lint_auto</test_depend>
  <test_depend>ament_lint_common</test_depend>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

`CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.8)
project(vision_detector_pkg)

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(cv_bridge REQUIRED)
find_package(image_transport REQUIRED)
find_package(OpenCV REQUIRED)

add_executable(vision_detector_node
  src/vision_detector_node.cpp
)

ament_target_dependencies(vision_detector_node
  rclcpp
  sensor_msgs
  cv_bridge
  image_transport
)

target_link_libraries(vision_detector_node
  ${OpenCV_LIBS}
)

install(TARGETS vision_detector_node
  DESTINATION lib/${PROJECT_NAME}
)

ament_package()
```

## 5.3 Docker Usage

The Docker image should install all available dependencies using `rosdep`:

```dockerfile
RUN rosdep update && \
    rosdep install \
      --from-paths /workspace/src \
      --ignore-src \
      --rosdistro humble \
      -r -y
```

This command scans the ROS 2 packages under `/workspace/src`, reads their `package.xml` files, and installs missing system dependencies.

## 5.4 Rule for Teams

For dependencies supported by `rosdep`:

```text
Each package must declare the dependency in package.xml.
The Dockerfile should not manually install package-specific dependencies unless there is a special reason.
```

Good:

```xml
<depend>opencv</depend>
```

Avoid:

```dockerfile
RUN apt-get install -y libopencv-dev
```

unless the Dockerfile is intentionally installing a base image dependency shared by the whole platform.

---

# 6. Case 2: Third-Party C++ Libraries Not Installable Through `rosdep` / `apt`

## 6.1 Description

Some dependencies are not available as Ubuntu packages or public ROS dependencies. Examples include:

- A vendor-provided C++ SDK.
- A special C++ library that must be built from source.
- A forked version of an open-source library.
- A library that requires custom CMake options.
- A library that is not stable enough to publish as a private `.deb` package yet.

For this project, the preferred approach for this category is:

```text
Build and install the third-party library inside the Docker image before building the ROS 2 workspace.
```

To keep this maintainable, the install logic should be placed under the main repository's `third_party/` directory.

---

## 6.2 Recommended Directory Structure

```text
main_repo/
  Dockerfile
  project.repos
  third_party/
    install_dependencies.sh
    install_my_cpp_lib.sh
    install_vendor_sdk.sh
  rosdep/
    company.yaml
```

The `third_party/` directory contains scripts that install external non-ROS dependencies.

The Dockerfile calls these scripts before running `rosdep install` and `colcon build`.

---

## 6.3 Purpose of `third_party/install_my_cpp_lib.sh`

The installer script is responsible for installing a third-party C++ library into the Docker image.

It usually performs the following steps:

1. Install build tools needed by the library.
2. Clone the library source repository.
3. Checkout a pinned tag, branch, or commit.
4. Configure the library with CMake.
5. Build the library.
6. Install the library into the image.
7. Clean up temporary build files.
8. Run `ldconfig` if shared libraries were installed.

This keeps the Dockerfile clean and makes third-party dependency installation explicit and reviewable.

---

## 6.4 Example Installer Script

Example file:

```text
main_repo/third_party/install_my_cpp_lib.sh
```

Script:

```bash
#!/usr/bin/env bash
set -euo pipefail

LIB_NAME="my_cpp_lib"
LIB_REPO="https://github.com/example/my_cpp_lib.git"
LIB_VERSION="v1.2.3"

BUILD_DIR="/tmp/${LIB_NAME}"
INSTALL_PREFIX="/usr/local"

echo "Installing ${LIB_NAME} ${LIB_VERSION}"

apt-get update
apt-get install -y --no-install-recommends \
  git \
  cmake \
  build-essential \
  ca-certificates

rm -rf "${BUILD_DIR}"

git clone --depth 1 --branch "${LIB_VERSION}" "${LIB_REPO}" "${BUILD_DIR}"

cmake -S "${BUILD_DIR}" -B "${BUILD_DIR}/build" \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX="${INSTALL_PREFIX}" \
  -DBUILD_TESTING=OFF

cmake --build "${BUILD_DIR}/build" --parallel "$(nproc)"

cmake --install "${BUILD_DIR}/build"

rm -rf "${BUILD_DIR}"

ldconfig

echo "${LIB_NAME} installed successfully"
```

Important details:

- `set -euo pipefail` makes the script fail early on errors.
- `LIB_VERSION` should be pinned to a tag or commit, not a floating branch, for reproducibility.
- `INSTALL_PREFIX=/usr/local` is simple and works for many libraries.
- `ldconfig` refreshes the dynamic linker cache after installing shared libraries.

---

## 6.5 Installing Into `/usr/local` vs `/opt/vendor`

### Option A: Install into `/usr/local`

This is the simplest option:

```bash
INSTALL_PREFIX="/usr/local"
```

Typical installed files:

```text
/usr/local/include/my_cpp_lib/...
/usr/local/lib/libmy_cpp_lib.so
/usr/local/lib/cmake/my_cpp_lib/my_cpp_libConfig.cmake
```

This usually works without extra environment variables.

### Option B: Install into `/opt/vendor/<library>`

This is cleaner when you want better isolation:

```bash
INSTALL_PREFIX="/opt/vendor/my_cpp_lib"
```

Then the Dockerfile may need:

```dockerfile
ENV CMAKE_PREFIX_PATH="/opt/vendor/my_cpp_lib:${CMAKE_PREFIX_PATH}"
ENV LD_LIBRARY_PATH="/opt/vendor/my_cpp_lib/lib:${LD_LIBRARY_PATH}"
```

For most teams, `/usr/local` is acceptable initially. Use `/opt/vendor/<library>` when you need stronger version isolation or multiple versions.

---

## 6.6 Aggregating Multiple Third-Party Installers

If the project has multiple manually installed third-party libraries, use one top-level script.

Example:

```text
third_party/
  install_dependencies.sh
  install_my_cpp_lib.sh
  install_vendor_sdk.sh
  install_optimization_lib.sh
```

`install_dependencies.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

"${SCRIPT_DIR}/install_my_cpp_lib.sh"
"${SCRIPT_DIR}/install_vendor_sdk.sh"
"${SCRIPT_DIR}/install_optimization_lib.sh"
```

Dockerfile:

```dockerfile
COPY third_party/ /tmp/third_party/

RUN chmod +x /tmp/third_party/*.sh && \
    /tmp/third_party/install_dependencies.sh && \
    rm -rf /tmp/third_party
```

This makes the Dockerfile stable even when individual dependencies are added or changed.

---

## 6.7 Dockerfile Example

Example Dockerfile for the main repository:

```dockerfile
FROM ros:humble-ros-base-jammy

SHELL ["/bin/bash", "-c"]

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    python3-rosdep \
    python3-vcstool \
    python3-colcon-common-extensions \
    git \
    cmake \
    build-essential \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Install third-party libraries that are not available through rosdep/apt.
COPY third_party/ /tmp/third_party/

RUN chmod +x /tmp/third_party/*.sh && \
    /tmp/third_party/install_dependencies.sh && \
    rm -rf /tmp/third_party

WORKDIR /workspace

# Copy or import ROS 2 module repositories.
# In CI, this may already be done using vcs import before docker build.
COPY src/ /workspace/src/

# Install standard ROS/system dependencies declared in package.xml.
# Skip keys that are manually installed by third_party scripts.
RUN rosdep update && \
    rosdep install \
      --from-paths /workspace/src \
      --ignore-src \
      --rosdistro humble \
      --skip-keys "my_cpp_lib vendor_sdk optimization_lib" \
      -r -y

# Build the integrated workspace.
RUN source /opt/ros/humble/setup.bash && \
    colcon build --symlink-install
```

---

## 6.8 Declaring Manually Installed Third-Party Dependencies

Even though the Dockerfile installs the library manually, the ROS package should still declare the dependency.

Example `package.xml`:

```xml
<depend>my_cpp_lib</depend>
```

This is important because the package really does depend on `my_cpp_lib`.

However, because `my_cpp_lib` is not known to public `rosdep`, the Dockerfile uses:

```bash
--skip-keys "my_cpp_lib"
```

This tells `rosdep`:

```text
Do not try to resolve this key. It has already been provided by the image.
```

This gives a good balance:

- The dependency remains visible in `package.xml`.
- The Docker image controls how the dependency is installed.
- `rosdep` still installs all normal dependencies.

---

## 6.9 Consumer Package Example

Suppose `vision_detector_pkg` uses `my_cpp_lib`.

`package.xml`:

```xml
<?xml version="1.0"?>
<package format="3">
  <name>vision_detector_pkg</name>
  <version>0.1.0</version>
  <description>Vision detector using my_cpp_lib</description>

  <maintainer email="vision-team@example.com">Vision Team</maintainer>
  <license>Proprietary</license>

  <buildtool_depend>ament_cmake</buildtool_depend>

  <depend>rclcpp</depend>
  <depend>sensor_msgs</depend>
  <depend>my_cpp_lib</depend>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

`CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.8)
project(vision_detector_pkg)

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(my_cpp_lib REQUIRED)

add_executable(vision_detector_node
  src/vision_detector_node.cpp
)

ament_target_dependencies(vision_detector_node
  rclcpp
  sensor_msgs
)

target_link_libraries(vision_detector_node
  my_cpp_lib::my_cpp_lib
)

install(TARGETS vision_detector_node
  DESTINATION lib/${PROJECT_NAME}
)

ament_package()
```

This assumes the third-party library installs a CMake package config file so that this works:

```cmake
find_package(my_cpp_lib REQUIRED)
```

If the library does not provide a CMake config file, see the next section.

---

## 6.10 When the Third-Party Library Does Not Provide CMake Config Files

Some libraries only install headers and `.so` files, but do not provide a proper CMake package.

In that case, the consuming package may need to use `find_path()` and `find_library()`.

Example:

```cmake
find_path(MY_CPP_LIB_INCLUDE_DIR
  NAMES my_cpp_lib/my_cpp_lib.hpp
  PATHS /usr/local/include
)

find_library(MY_CPP_LIB_LIBRARY
  NAMES my_cpp_lib
  PATHS /usr/local/lib /usr/local/lib/x86_64-linux-gnu
)

if(NOT MY_CPP_LIB_INCLUDE_DIR OR NOT MY_CPP_LIB_LIBRARY)
  message(FATAL_ERROR "my_cpp_lib not found")
endif()

add_executable(vision_detector_node
  src/vision_detector_node.cpp
)

target_include_directories(vision_detector_node PRIVATE
  ${MY_CPP_LIB_INCLUDE_DIR}
)

target_link_libraries(vision_detector_node
  ${MY_CPP_LIB_LIBRARY}
)
```

However, the preferred approach is to improve the third-party library installation so that it exports a CMake package config file. This makes usage cleaner and less error-prone.

---

## 6.11 Optional: Custom rosdep Rule With Empty Install Mapping

Another approach is to define a custom rosdep key for manually installed dependencies. However, if the library is installed by a script and not by apt, using `--skip-keys` is usually simpler.

Recommended for this project:

```bash
--skip-keys "my_cpp_lib vendor_sdk"
```

Avoid hiding the dependency by removing it from `package.xml`.

---

## 6.12 Rules for Third-Party Installer Scripts

All scripts under `third_party/` should follow these rules:

1. Pin exact dependency versions.
2. Fail fast on errors.
3. Avoid interactive prompts.
4. Use `--no-install-recommends` for apt installs.
5. Remove temporary build directories.
6. Prefer Release builds.
7. Disable tests unless tests are needed during the image build.
8. Print clear log messages.
9. Do not install unrelated system packages.
10. Keep one script focused on one library.

Good version pinning:

```bash
LIB_VERSION="v1.2.3"
```

Avoid:

```bash
git clone https://github.com/example/my_cpp_lib.git
```

without checking out a tag or commit.

---

# 7. Case 3: Custom Internal Libraries Already Part of the ROS 2 Project

## 7.1 Description

If the library is developed by the project teams and is part of the ROS 2 workspace, it should be a normal ROS 2 package.

Do not install it manually in Docker.
Do not treat it as a third-party dependency.
Do not copy headers between modules.

Instead:

```text
Create a dedicated ROS 2 package for the library.
Declare dependencies through package.xml.
Let colcon build it in dependency order.
```

---

## 7.2 Example Structure

```text
src/
  common_module/
    geometry_utils/
      package.xml
      CMakeLists.txt
      include/
        geometry_utils/
          geometry_utils.hpp
      src/
        geometry_utils.cpp

  vision_module/
    vision_detector_pkg/
      package.xml
      CMakeLists.txt
      src/
        vision_detector_node.cpp

  planning_module/
    motion_planner_pkg/
      package.xml
      CMakeLists.txt
      src/
        motion_planner_node.cpp
```

Here, `geometry_utils` is a shared internal library package.

Both `vision_detector_pkg` and `motion_planner_pkg` may depend on it.

---

## 7.3 Internal Library Package Example

`geometry_utils/package.xml`:

```xml
<?xml version="1.0"?>
<package format="3">
  <name>geometry_utils</name>
  <version>0.1.0</version>
  <description>Shared geometry utility library</description>

  <maintainer email="common-team@example.com">Common Team</maintainer>
  <license>Proprietary</license>

  <buildtool_depend>ament_cmake</buildtool_depend>

  <depend>rclcpp</depend>
  <depend>eigen</depend>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

`geometry_utils/CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.8)
project(geometry_utils)

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(Eigen3 REQUIRED)

add_library(${PROJECT_NAME}
  src/geometry_utils.cpp
)

target_include_directories(${PROJECT_NAME}
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)

ament_target_dependencies(${PROJECT_NAME}
  rclcpp
)

target_link_libraries(${PROJECT_NAME}
  Eigen3::Eigen
)

install(
  DIRECTORY include/
  DESTINATION include
)

install(
  TARGETS ${PROJECT_NAME}
  EXPORT export_${PROJECT_NAME}
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
  RUNTIME DESTINATION bin
)

ament_export_targets(export_${PROJECT_NAME} HAS_LIBRARY_TARGET)
ament_export_dependencies(rclcpp Eigen3)

ament_package()
```

Example header:

`geometry_utils/include/geometry_utils/geometry_utils.hpp`:

```cpp
#pragma once

namespace geometry_utils
{

double normalize_angle(double angle_rad);

}  // namespace geometry_utils
```

Example source:

`geometry_utils/src/geometry_utils.cpp`:

```cpp
#include "geometry_utils/geometry_utils.hpp"

#include <cmath>

namespace geometry_utils
{

double normalize_angle(double angle_rad)
{
  while (angle_rad > M_PI) {
    angle_rad -= 2.0 * M_PI;
  }
  while (angle_rad < -M_PI) {
    angle_rad += 2.0 * M_PI;
  }
  return angle_rad;
}

}  // namespace geometry_utils
```

---

## 7.4 Consumer Package Example

`vision_detector_pkg/package.xml`:

```xml
<depend>geometry_utils</depend>
```

`vision_detector_pkg/CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.8)
project(vision_detector_pkg)

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(geometry_utils REQUIRED)

add_executable(vision_detector_node
  src/vision_detector_node.cpp
)

ament_target_dependencies(vision_detector_node
  rclcpp
  geometry_utils
)

install(TARGETS vision_detector_node
  DESTINATION lib/${PROJECT_NAME}
)

ament_package()
```

Now `colcon` understands that `geometry_utils` must be built before `vision_detector_pkg`.

No Docker-specific installation is needed for `geometry_utils` because it is part of the ROS 2 workspace.

---

# 8. Main Repository Responsibilities

The main repository should contain the integration and build logic.

Example:

```text
main_repo/
  Dockerfile
  project.repos
  third_party/
    install_dependencies.sh
    install_my_cpp_lib.sh
  rosdep/
    company.yaml
  scripts/
    build_workspace.sh
```

## 8.1 `project.repos`

The `project.repos` file defines which module repositories are included.

Example:

```yaml
repositories:
  src/vision_module:
    type: git
    url: git@github.com:company/vision_module.git
    version: main

  src/planning_module:
    type: git
    url: git@github.com:company/planning_module.git
    version: main

  src/control_module:
    type: git
    url: git@github.com:company/control_module.git
    version: main

  src/common_module:
    type: git
    url: git@github.com:company/common_module.git
    version: main
```

For reproducible builds, release branches should pin exact tags or commits instead of floating `main` branches.

---

## 8.2 Build Script

Example:

```text
main_repo/scripts/build_workspace.sh
```

```bash
#!/usr/bin/env bash
set -euo pipefail

source /opt/ros/humble/setup.bash

rosdep update

rosdep install \
  --from-paths /workspace/src \
  --ignore-src \
  --rosdistro humble \
  --skip-keys "my_cpp_lib vendor_sdk optimization_lib" \
  -r -y

colcon build --symlink-install
```

---

# 9. Recommended Build Flow

A typical CI or Docker build should follow this order:

```text
1. Start from a ROS 2 base image.
2. Install basic build tools.
3. Install third-party non-rosdep libraries using third_party/*.sh scripts.
4. Import or copy module repositories into src/.
5. Run rosdep install for normal dependencies.
6. Skip rosdep keys that were manually installed by third_party scripts.
7. Build the workspace using colcon.
8. Run tests.
```

Example:

```bash
vcs import src < project.repos

docker build -t company/robotics-dev:latest .
```

Inside Docker:

```bash
rosdep install --from-paths src --ignore-src --skip-keys "my_cpp_lib" -r -y
colcon build --symlink-install
colcon test
```

---

# 10. Dependency Ownership Rules

## 10.1 Package Owners

Package owners must:

- Declare every direct dependency in `package.xml`.
- Use `find_package()` or equivalent CMake logic in `CMakeLists.txt`.
- Avoid relying on undeclared global libraries.
- Add internal library dependencies as normal ROS package dependencies.
- Inform the integration team when a new third-party non-rosdep dependency is needed.

## 10.2 Module Teams

Module teams must:

- Keep module packages buildable in a clean workspace when dependencies are installed.
- Avoid installing dependencies from inside package build scripts.
- Avoid cloning third-party libraries from inside `CMakeLists.txt`.
- Avoid committing large external dependency source trees into module repositories unless explicitly approved.

## 10.3 Integration / Platform Team

The integration or platform team must:

- Maintain the main Dockerfile.
- Maintain `project.repos`.
- Maintain `third_party/` installer scripts.
- Maintain the list of `--skip-keys` for manually installed dependencies.
- Ensure CI builds the complete integrated workspace.
- Pin dependency versions where needed.

---

# 11. Good and Bad Patterns

## 11.1 Good Pattern: rosdep-available dependency

Package:

```xml
<depend>opencv</depend>
```

Docker:

```bash
rosdep install --from-paths src --ignore-src -r -y
```

## 11.2 Good Pattern: manually installed third-party dependency

Package:

```xml
<depend>my_cpp_lib</depend>
```

Docker:

```bash
/tmp/third_party/install_my_cpp_lib.sh
rosdep install --from-paths src --ignore-src --skip-keys "my_cpp_lib" -r -y
```

## 11.3 Good Pattern: internal project library

Consumer package:

```xml
<depend>geometry_utils</depend>
```

Workspace:

```text
src/common_module/geometry_utils
src/vision_module/vision_detector_pkg
```

Build:

```bash
colcon build
```

## 11.4 Bad Pattern: hidden dependency only in Dockerfile

Bad:

```dockerfile
RUN git clone https://github.com/example/my_cpp_lib.git && \
    cd my_cpp_lib && \
    cmake -S . -B build && \
    cmake --build build && \
    cmake --install build
```

Package has no dependency declaration:

```xml
<!-- missing <depend>my_cpp_lib</depend> -->
```

Problem:

The package uses `my_cpp_lib`, but this is invisible to developers and CI dependency analysis.

## 11.5 Bad Pattern: cloning dependencies inside package CMake

Avoid this:

```cmake
execute_process(COMMAND git clone https://github.com/example/my_cpp_lib.git)
```

Problem:

- Builds become slow and unreliable.
- Network access is required during package build.
- Version control is hidden.
- Multiple packages may clone different versions.

---

# 12. Versioning and Reproducibility

For reproducible builds:

- Pin the ROS 2 base image version.
- Pin third-party library tags or commits in installer scripts.
- Pin module repositories in `project.repos` for releases.
- Avoid floating branches for production images.
- Keep third-party install scripts under code review.

Example installer pin:

```bash
LIB_VERSION="v1.2.3"
```

Better for strict reproducibility:

```bash
LIB_COMMIT="abc123def4567890abc123def4567890abc123de"
git checkout "${LIB_COMMIT}"
```

Example `project.repos` release pin:

```yaml
repositories:
  src/vision_module:
    type: git
    url: git@github.com:company/vision_module.git
    version: 4f8e1c2a9b1d0a65c5a68f3e2f98e8b82d72a123
```

---

# 13. Recommended Policy

The project should use the following dependency policy:

```text
1. If a dependency is available through rosdep, declare it in package.xml and let rosdep install it.

2. If a dependency is an external third-party C++ library not available through rosdep or apt, install it in the Docker image using a script under third_party/.

3. Even if a third-party library is installed manually in Docker, the ROS package that uses it must still declare it in package.xml.

4. For manually installed third-party libraries, rosdep should be called with --skip-keys for those dependency names.

5. If a library is developed inside the project, make it a normal ROS 2 package and let colcon build it.

6. Do not clone or build third-party dependencies from inside package CMakeLists.txt files.

7. The main repository owns the Dockerfile, project.repos, and third_party installer scripts.

8. Module repositories own their package.xml and CMakeLists.txt dependency declarations.
```

---

# 14. Final Reference Example

Final integrated structure:

```text
main_repo/
  Dockerfile
  project.repos
  third_party/
    install_dependencies.sh
    install_my_cpp_lib.sh
  scripts/
    build_workspace.sh

workspace/
  src/
    vision_module/
      vision_detector_pkg/
        package.xml
        CMakeLists.txt

    planning_module/
      motion_planner_pkg/
        package.xml
        CMakeLists.txt

    common_module/
      geometry_utils/
        package.xml
        CMakeLists.txt
```

`vision_detector_pkg/package.xml`:

```xml
<depend>rclcpp</depend>
<depend>sensor_msgs</depend>
<depend>opencv</depend>
<depend>my_cpp_lib</depend>
<depend>geometry_utils</depend>
```

Meaning:

```text
rclcpp, sensor_msgs, opencv:
  Installed by rosdep.

my_cpp_lib:
  Installed by third_party/install_my_cpp_lib.sh and skipped by rosdep.

geometry_utils:
  Built by colcon because it is an internal ROS 2 package in the workspace.
```

Docker build order:

```text
1. Install base tools.
2. Run third_party/install_dependencies.sh.
3. Copy/import src modules.
4. Run rosdep install with --skip-keys for manually installed libraries.
5. Run colcon build.
```

This approach keeps dependency management scalable across multiple teams while still allowing module-specific dependencies and practical Docker-based builds.

