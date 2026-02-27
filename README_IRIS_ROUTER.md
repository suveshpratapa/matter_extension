# Building Matter Thread Router Lighting App with Project Iris Support

This document provides exact reproduction steps to build a Matter lighting application with the following characteristics:

1. Matter Thread-as-Router class device (Full Thread Device)
2. OpenThread and Matter logging at debug level
3. Shell enabled for running OT commands
4. Pairing mode support enabled (BLE commissioning)

## Repository Setup

### 1. Clone and Checkout

```bash
# Clone the repository (or navigate to existing clone to set up custom remote)
git clone git@github.com:suveshpratapa/matter_extension.git matter_extension
cd matter_extension

# Checkout the Project Iris matter branch
git checkout matter-iris
```

### 2. Initialize Submodules

```bash
# Initialize and update all submodules
git submodule update --init --recursive
```

**Note**: If you encounter submodule checkout errors, the submodules may need manual fixing. Verify that `third_party/simplicity_sdk`, `third_party/matter_sdk`, and `third_party/matter_support` are properly checked out.

## Environment Setup

### 3. Install Dependencies

The `sl_setup_env.py` script will automatically download and configure all required tools:

```bash
python3 slc/sl_setup_env.py
```

**Important**: During execution, you will be prompted (maybe multiple times) to answer whether you want to use SLT (Silicon Labs Tools) provided in this repo in case you already have a local development toolchain set up:
- Answer `y` (yes) to both prompts for simplest approach

The script will download and install:
- SLC-CLI v6.0.16 (Silicon Labs Configurator)
- Java 21 (OpenJDK 21.0.5)
- ARM GCC toolchain (arm-none-eabi-gcc 12.2.1) via Conan
- Ninja build system v1.12.1
- CMake 3.30.2
- Silicon Labs Commander
- ZAP tool v2025.12.02

This process creates a `.env` file at `slc/tools/.env` with all tool paths configured.

**Expected completion message**:
```
Environment setup completed successfully
```

## Project Creation

Note: The example project below, `ThreadRouterLightApp`, has been checked in to the repo, with an s37 as well, for testing purposes.

### 4. Create Thread Router Lighting App

```bash
python3 slc/sl_create_new_app.py ThreadRouterLightApp slc/apps/lighting-app/thread/lighting-app.slcp brd4187c
```

This command:
- Creates a new project directory: `ThreadRouterLightApp/`
- Uses the base Thread lighting app template
- Configures for board BRD4187C (EFR32MG24)
- Runs SLC component generation
- Generates ZAP cluster files

**Expected completion message**:
```
Generation for Matter Light to /Users/.../matter_extension/ThreadRouterLightApp has completed.
```

The base template already includes:
- `matter_shell` - CHIP Shell for CLI commands and OT commands
- `matter_thread` - Thread network layer (includes Full Thread Device stack)
- `matter_ble` - Bluetooth LE for commissioning (pairing mode)
- `matter_provision_default` - Default provisioning/commissioning support

## Configure Debug Logging

### 5. Edit Project Configuration Files

Debug logging requires changes to **two** configuration files:

#### 5a. Edit `lighting-app.slcp` for Matter Logging

Edit the file `ThreadRouterLightApp/lighting-app.slcp` to add Matter debug logging. If not already present, find the `define:` section (around line 169) and add these defines after the existing ones:

```yaml
- name: SL_MATTER_LOG_LEVEL
  value: SL_MATTER_LOG_DETAIL
- name: CHIP_PROGRESS_LOGGING
  value: '1'
- name: CHIP_DETAIL_LOGGING
  value: '1'
- name: CHIP_ERROR_LOGGING
  value: '1'
```

#### 5b. Edit `config/sl_openthread_features_config.h` for OpenThread Logging

**CRITICAL**: The OpenThread logging settings must be edited in the generated config header, as SLCP defines won't override this file.

Edit `ThreadRouterLightApp/config/sl_openthread_features_config.h` (around line 422-428):

Change:
```c
// <q>  DYNAMIC_LOG_LEVEL
#ifndef OPENTHREAD_CONFIG_LOG_LEVEL_DYNAMIC_ENABLE
#define OPENTHREAD_CONFIG_LOG_LEVEL_DYNAMIC_ENABLE  0
#endif

// <e>  Enable Logging
#define OPENTHREAD_FULL_LOGS_ENABLE                 0
```

To:
```c
// <q>  DYNAMIC_LOG_LEVEL
#ifndef OPENTHREAD_CONFIG_LOG_LEVEL_DYNAMIC_ENABLE
#define OPENTHREAD_CONFIG_LOG_LEVEL_DYNAMIC_ENABLE  1
#endif

// <e>  Enable Logging
#define OPENTHREAD_FULL_LOGS_ENABLE                 1
```

**Logging Configuration Explained**:

Matter Logging (in `.slcp`):
- `SL_MATTER_LOG_LEVEL = SL_MATTER_LOG_DETAIL` - Enables detailed Matter platform logging
- `CHIP_PROGRESS_LOGGING = 1` - Enables Matter SDK progress logs
- `CHIP_DETAIL_LOGGING = 1` - Enables Matter SDK detailed debug logs
- `CHIP_ERROR_LOGGING = 1` - Enables Matter SDK error logs

OpenThread Logging (in `sl_openthread_features_config.h`):
- `OPENTHREAD_FULL_LOGS_ENABLE = 1` - Enables OpenThread logging infrastructure
- `OPENTHREAD_CONFIG_LOG_LEVEL_DYNAMIC_ENABLE = 1` - Allows runtime control of log levels via `otcli log level <level>`
- `OPENTHREAD_CONFIG_LOG_LEVEL = OT_LOG_LEVEL_DEBG` - Sets default log level to 5 (debug) when FULL_LOGS_ENABLE is 1

## Build the Application

### 6. Build with CMake

Remove all the three occurances of the `--specs=nano.specs` from the `ThreadRouterLightApp/cmake_gcc/lighting-app.cmake` to build the app with full `newlib` support for the TimeSync feature.

```bash
cd ThreadRouterLightApp/cmake_gcc
source ../../slc/tools/.env
cmake --workflow --preset project
```

**Build process**:
1. `source ../../slc/tools/.env` - Loads environment variables for toolchain paths
2. `cmake --workflow --preset project` - Runs full CMake workflow (configure, build, post-build)

**Expected output**:
```
[1014/1014] Linking CXX executable base/lighting-app.out
...
Ram usage       :   ...
Flash usage     :   ...
DONE
```

**Build artifacts** (located in `build/base/`):
- `lighting-app.out` - ELF executable with debug symbols
- `lighting-app.s37` - S-record file for flashing
- `lighting-app.hex` - Intel HEX file
- `lighting-app.bin` - Binary file
- `lighting-app.map` - Memory map file

### Rebuilding After Configuration Changes

If you modify the `.slcp` file after the initial build (e.g., changing configurations or sources), you can perform a clean rebuild:

```bash
cd ThreadRouterLightApp/cmake_gcc
rm -rf build
source ../../slc/tools/.env
cmake --workflow --preset project
```

## Verification

### 7. Configuration

The built application should have:

**Thread Router (Full Thread Device)**:
- Default behavior when `matter_thread` component is included
- Component `ot_stack_ftd` is automatically required (unless using ICD)
- Device can route Thread network traffic
- Support for Project Iris feature as a mesh forwarder router role

**Debug Logging Enabled**:
- Matter logs at DETAIL level
- OpenThread logs at DEBUG level
- Logs output via UART (configured by `matter_log_uart` component)

**Shell Enabled**:
- CHIP Shell CLI available
- OpenThread CLI commands available via `otcli` prefix
- Example commands: `otcli state`, `otcli ipaddr`

**Pairing Mode Support**:
- BLE commissioning enabled via `matter_ble` component
- QR code image generated: `qr_code_img.png`
- Device can be commissioned via Apple Home app or chip-tool

## Troubleshooting

### Environment Setup Issues

**Problem**: ZAP download fails with network error
- **Solution**: Re-run `python3 slc/sl_setup_env.py` - the script is idempotent and will retry failed downloads

**Problem**: Missing `.env` file error during project creation
- **Solution**: Always run `sl_setup_env.py` before creating projects

### Build Issues

**Problem**: ARM GCC compiler not found
- **Solution**: Ensure you sourced the .env file: `source ../../slc/tools/.env`

**Problem**: CMake errors about missing presets
- **Solution**: Verify project was created successfully and you're in the `cmake_gcc` directory

### Submodule Issues

**Problem**: Submodules not checked out properly
- **Solution**: Run `git submodule update --init --recursive` from repository root
- Verify `third_party/simplicity_sdk`, `third_party/matter_sdk`, and `third_party/matter_support` exist

## Testing the Application

### Flash to Device

```bash
# Using Silicon Labs Commander
commander flash build/base/lighting-app.s37 --device EFR32MG24B220F1536IM48
```

### Serial connection

Connect to the device UART (typically `/dev/tty.usbmodem*` on macOS or `/dev/ttyACM*` on Linux), or using telnet with IP address.

In the UART console:
```
matter help          # List Matter shell commands
otcli help           # List OpenThread commands
```

You should see detailed debug logs from Matter and OpenThread and OpenThread logging level can be adjusted using `otcli log level`

### Commission the Device

Use the generated QR code in `qr_code_img.png` to commission the device with:
- Apple Home app (iOS/macOS)
- chip-tool (Matter controller)
- Google Home app (Android)

## Additional Resources

- Main README: [README.md](README.md)
- Matter SDK Documentation: `third_party/matter_sdk/docs/`
- Component Reference: `slc/component/`
- Silicon Labs Thread PAL: `third_party/simplicity_sdk/openthread/`
- Silicon Labs Thread Apps: `third_party/simplicity_sdk/openthread_app/`
- OpenThread: `third_party/simplicity_sdk/protocol/openthread_stack/`

## Version Information

- Matter Extension: v2.8.0
- Simplicity SDK: v2025.12.0 with Project Iris support added
