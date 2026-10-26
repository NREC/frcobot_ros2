# Development notes

Fork of [FAIR-INNOVATION/frcobot_ros2](https://github.com/FAIR-INNOVATION/frcobot_ros2).


## Branches

- `main`: read-only mirror of upstream `main`.
- `dev`: development branch, i.e. `main` with custom features and bug fixes. Upstream changes should be merged from
  `main`. Never rebase or force-push, since downstream workspaces can pin to specific commits.

```bash
git remote add upstream https://github.com/FAIR-INNOVATION/frcobot_ros2.git   # once per clone
git fetch upstream
git checkout main && git merge --ff-only upstream/main && git push origin main
git checkout dev && git merge main
```

New upstream hardware packages do not have a `COLCON_IGNORE`. After merging, exactly one `fairino_hardware*` package
must remain buildable:

```bash
for d in fairino_hardware*/; do [ -e "$d/COLCON_IGNORE" ] || echo "$d"; done   # must print one line
```


## Versioning

Three independent version numbers are used in upstream:

- Robot/controller software (`4.0.0`): return value of `GetSoftwareVersion()`. It is used to name the hardware packages
  (`fairino_hardware_v4_0_0`).
- C++ SDK (`2.4.0`): the vendored `libfairino.so.2.4.0`. Each SDK tag pairs it with a controller version
  (`v2.4.0-robot4.0.0`).
- ROS driver (`3.0.0`): the release of upstream's packages. It appears only in upstream commit messages.


## Hardware packages

Upstream ships one `fairino_hardware_vX_Y_Z` package per controller firmware, each vendoring the matching `libfairino`
SDK build. The ros2_control plugin source is identical in all of them. They all export the same plugin name
(`fairino_hardware/FairinoHardwareInterface`) and install `libfairino.so.2`. With several packages installed, pluginlib
silently loads the alphabetically first package, and `libfairino.so.2` resolves through `LD_LIBRARY_PATH`. So only the
selected version is built, and every other `fairino_hardware*` package has a `COLCON_IGNORE`.

The unversioned `fairino_hardware` package is upstream's runtime dispatcher for the service API (`ros2_cmd_server`),
and is unrelated to ros2_control.

The selected version must match the controller software on every robot using this workspace. To change it:

1. Upgrade the controller software on all robots to the new version.
2. Move the `COLCON_IGNORE` from the new package to the old one.
3. Port the old package's `dev` changes (`git diff main dev -- fairino_hardware_vOLD/`) to the new package.
4. Build the aarch64 `libfairino` for the new SDK version (see below).
5. In downstream workspaces, delete the old package's `build/` and `install/` directories. A stale install still
   registers the plugin.


## libfairino for aarch64

Upstream vendors only an x86_64 Linux build. Build the aarch64 library from
[fairino-cpp-sdk](https://github.com/FAIR-INNOVATION/fairino-cpp-sdk), at the tag that matches the vendored
`libfairino.so.2.A.B` (for example, `v2.4.0-robot4.0.0`). Build natively on an aarch64 machine or in a container
running the target's Ubuntu release, so glibc matches:

```bash
git clone --depth 1 --branch v2.A.B-robotX.Y.Z https://github.com/FAIR-INNOVATION/fairino-cpp-sdk.git
cmake -S fairino-cpp-sdk/libfairino -B build
cmake --build build --target fairino -j"$(nproc)"   # -> fairino-cpp-sdk/libfairino/LinuxBuild/bin/libfairino.so.2.A.B
```

The Python SDK's `Robot.c` is the Cython-generated source for the Python module, and does not correspond to this
library.


## dev changes

- `COLCON_IGNORE` in every `fairino_hardware*` package except the selected version, and in the `*_moveit2_config`
  packages.
- This file.


## Upstream limitations (checked at v4.0.0)

- The plugin exports only `position` command and state interfaces; velocity and effort are commented out. `on_init`
  rejects joints whose first command interface is not `position`.
- The SDK provides joint speeds and torques (`GetActualJointSpeedsDegree`, `GetJointTorques`, or
  `actual_qd`/`jt_cur_tor` in `ROBOT_STATE_PKG`) and a torque servo (`ServoJT`), but no joint-velocity servo.


## To do

- [ ] Reset `main` to `upstream/main` (force-push) to drop merge commit `1d8079e`.
- [ ] Add per-architecture libraries (`libfairino/lib/{x86_64,aarch64}/`) for the selected hardware package, chosen by
  `CMAKE_SYSTEM_PROCESSOR` as on `hotfix/support-aarch64`.
- [ ] Delete `hotfix/support-aarch64` once `dev` has been tested with downstream.
