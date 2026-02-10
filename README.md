# ZMK Module for DASBOB

This repository contains a template for a ZMK module, as it would most frequently be used.

## Usage

Create a new repository for your ZMK config with at least the following files:

`build.yaml`:

```yaml
include:
  - board: nice_nano
    shield: dasbob_left
  - board: nice_nano
    shield: dasbob_right
  - board: nice_nano
    shield: settings_reset
```

`config/west.yml`:

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: PandaScience
      url-base: https://github.com/PandaScience
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: v0.3.0
      import: app/west.yml
    - name: zmk-module-dasbob
      remote: PandaScience
      revision: main
  self:
    path: config
```

`config/dasbob.keymap`:

```C
#include <dt-bindings/zmk/keys.h>

/ {

    keymap {
        compatible = "zmk,keymap";
        base {
            bindings = <
                &kp Q  &kp W  &kp E    &kp R     &kp T      &kp Y      &kp U   &kp I      &kp O    &kp P
                &kp A  &kp S  &kp D    &kp F     &kp G      &kp H      &kp J   &kp K      &kp L    &kp SEMI
                &kp Z  &kp X  &kp C    &kp V     &kp B      &kp N      &kp M   &kp COMMA  &kp DOT  &kp SLASH
                              &kp ESC  &kp BSPC  &kp SPACE  &kp ENTER  &kp DEL &kp TAB
            >;
        };
    };
};
```

`.github/workflows/build.yml`:

```yaml
on: [push, pull_request, workflow_dispatch]

jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3.0
```

## Helpful Resources

Official ZMK documentation on:

- [Modules](https://zmk.dev/docs/development/module-creation)
- [Boards & Shilds Overview](https://zmk.dev/docs/development/hardware-integration)
- [Shields](https://zmk.dev/docs/development/hardware-integration/new-shield)
- [Devicetree Files](https://zmk.dev/docs/config#devicetree-files)
- [Keyboard Scan Configuration](https://zmk.dev/docs/config/kscan)
- [Layout Configuration](https://zmk.dev/docs/config/layout)
- [Physical Layout Configuration](https://zmk.dev/docs/development/hardware-integration/physical-layouts)

For more info on modules, you can read through through the [Zephyr modules page](https://docs.zephyrproject.org/3.5.0/develop/modules.html) and [ZMK's page on using modules](https://zmk.dev/docs/features/modules). [Zephyr's west manifest page](https://docs.zephyrproject.org/3.5.0/develop/west/manifest.html#west-manifests) may also be of use.

Good resources for advanced configurations and integration of external modules:

- https://github.com/urob/zmk-config
- https://github.com/bmijanovich/zmk-config
- https://getreuer.info/posts/keyboards/index.html

## Local Build

To build locally, you will need to have the [ZMK toolchain](https://zmk.dev/docs/development/toolchain) installed in one or another way. We will follow the [containerized](https://zmk.dev/docs/development/local-toolchain/setup/container) approach here.

### Local Module Testing

In case there is no module repo yet or you want to test changes to the keyboard module, this setup might be helpful:

Clone ZMK on the same level you have this module folder (`zmk-module-dasbob`) and the optional config folder (`zmk-config`):

```bash
git clone https://github.com/zmkfirmware/zmk.git
git switch --detach v0.3.0 # adapt version as needed
```

Then run the podman compose file provided in this repo to setup the local build environment in a container and mount the relevant folders:

```bash
podman compose up -d --build
```

Exec into the container,

```bash
podman exec -it zmk-dev /bin/bash

```

and initialize west:

```bash
west init -l /app
west update
cd app
```

From now on, run all build commands from within the `app` folder with the following flags and options:

```bash
west build -d build/left  -p auto -b nice_nano -- -DSHIELD=dasbob_left    -DZMK_EXTRA_MODULES=/workspaces/zmk-module-dasbob -DZMK_CONFIG=/workspaces/zmk-config/config
west build -d build/right -p auto -b nice_nano -- -DSHIELD=dasbob_right   -DZMK_EXTRA_MODULES=/workspaces/zmk-module-dasbob -DZMK_CONFIG=/workspaces/zmk-config/config
west build -d build/reset -p auto -b nice_nano -- -DSHIELD=settings_reset -DZMK_EXTRA_MODULES=/workspaces/zmk-module-dasbob -DZMK_CONFIG=/workspaces/zmk-config/config
```

or set some CMAKE options permanently so that the build command can be simplified:

```bash
# this will persist the options to zmk/.west/config
west config build.cmake-args -- "-DZMK_EXTRA_MODULES=/workspaces/zmk-modules -DZMK_CONFIG=/workspaces/zmk-config/config"
west build -d build/left  -p auto -b nice_nano -- -DSHIELD=dasbob_left
west build -d build/right -p auto -b nice_nano -- -DSHIELD=dasbob_right
west build -d build/reset -p auto -b nice_nano -- -DSHIELD=settings_reset
```

For convenience, copy over the firmware binaries and name them properly:

```bash
cp build/left/zephyr/zmk.uf2 /workspaces/zmk-config/build/dasbob_left.uf2
cp build/right/zephyr/zmk.uf2 /workspaces/zmk-config/build/dasbob_right.uf2
cp build/reset/zephyr/zmk.uf2 /workspaces/zmk-config/build/dasbob_reset.uf2
```

### Local Config Testing (ZMK Workspace)

In case you only want to test the config locally, possibly while integrating even more modules, you only need to clone the ZMK firmware. For the container build, you can reuse the podman compose file, though we only need to mount the config and ZMK paths.

Then include all external modules in the `zmk/app/west.yml` file before initializing west and fetching all dependencies:

```yaml
manifest:
  remotes:
    - name: zephyrproject-rtos
      url-base: https://github.com/zephyrproject-rtos
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: urob                                          # <- added
      url-base: https://github.com/urob
    - name: PandaScience                                  # <- added
      url-base: https://github.com/PandaScience
  projects:
    - name: zmk-adaptive-key                              # <- added
      remote: urob
      revision: v0.3 # Should match ZMK release.
    - name: zmk-module-dasbob                             # <- added
      remote: PandaScience
      revision: main
    - name: zephyr
      remote: zmkfirmware
      revision: v4.1.0+zmk-fixes
      clone-depth: 1
      ...

```

Build commands are identical to the ones above except you can omit the `-DZMK_EXTRA_MODULES` options as the modules are now included in the west manifest and will be fetched from their remotes.

### Local Config Testing (Config Workspace)

This is the easiest and most common way to test your config locally, as it closely resembles the Github workflow with only minor adjustments.

**NOTE:** All build-related files and folders will be created in the config repo instead of the ZMK repo. To prevent cluttering, it is advisable to add them to the `.gitignore` file:

```gitignore
.west/
build/
modules/
zephyr/
zmk/
zmk-adaptive-key/
zmk-module-dasbob/
```

No need for a podman compose here, we can directly startup the container via

```bash
podman run -it --rm --security-opt label=disable -v $(pwd):/zmk-config -w /zmk-config zmkfirmware/zmk-dev-arm:3.5 /bin/bash
```

Initialize west, fetch dependencies and build the firmware:

```bash
west init -l config
west update

export "CMAKE_PREFIX_PATH=/zmk-config/zephyr:$CMAKE_PREFIX_PATH"

west build -s zmk/app -p auto -b nice_nano -d build/left  -- -DZMK_CONFIG=/zmk-config/config -DSHIELD=dasbob_left
west build -s zmk/app -p auto -b nice_nano -d build/right -- -DZMK_CONFIG=/zmk-config/config -DSHIELD=dasbob_right

ln -s build/left/zephyr/zmk.uf2 build/dasbob_left.uf2
ln -s build/right/zephyr/zmk.uf2 build/dasbob_right.uf2
```
