# Arduino App Lab — Raspberry Pi / Linux ARM64
## Build (unofficial)

This is an **unofficial** fork of [arduino/arduino-app-lab](https://github.com/arduino/arduino-app-lab), providing a working build for Raspberry Pi (Linux ARM64). The upstream project does not yet publish an ARM64/Raspberry Pi release.

All source code is unmodified from upstream. This fork only adds Raspberry Pi build/packaging instructions and one runtime fix (see below). License: GPL-3.0 (see [LICENSE](LICENSE)), same as upstream.

<img width="1920" height="1025" alt="Arduino-App-Lab" src="https://github.com/user-attachments/assets/16a1dfad-fc22-46a8-afa5-baf3ce603410" />

## Quick install (Raspberry Pi OS, 64-bit)

1. Download the `.deb` from the [Releases](../../releases) page.
2. Install it:
   ```bash
   sudo apt install ./arduino-app-lab_*.deb
   ```
3. Launch "Arduino App Lab" from the Applications menu, or from a terminal:
   ```bash
   ArduinoAppLab
   ```
   (the rendering fix below is already applied automatically — no extra flags needed)

## Known issue: garbled window on launch

On Raspberry Pi, WebKitGTK's hardware-accelerated compositing can render the window as corrupted/garbled until it's resized. Setting `WEBKIT_DISABLE_COMPOSITING_MODE=1` fixes it, **but** App Lab rewrites its own `~/.local/share/applications/ArduinoAppLab.desktop` file from scratch on every launch (to register its `arduino-app-lab://` protocol handler), silently discarding any fix placed only in that file's `Exec=` line or in the session environment.

The reliable fix is to put the environment variable on the binary's own path, so it applies no matter how the app is launched. This package already does it: the real executable is installed as `/usr/bin/ArduinoAppLab.bin`, and `/usr/bin/ArduinoAppLab` is a small wrapper script:

```bash
#!/bin/sh
export WEBKIT_DISABLE_COMPOSITING_MODE=1
exec "$(dirname "$0")/ArduinoAppLab.bin" "$@"
```

## Building from source

Tested on Raspberry Pi OS (64-bit, Debian 12 "bookworm"), Raspberry Pi 4.

### 1. Check you're on 64-bit

```bash
uname -m   # must print "aarch64"
```

### 2. System dependencies

```bash
sudo apt-get update
sudo apt-get install -y git wget jq unzip zip build-essential pkg-config libwebkit2gtk-4.1-dev libgtk-3-dev
```

### 3. Go 1.26.3+ (the version in apt is too old)

```bash
cd /tmp
wget https://go.dev/dl/go1.26.3.linux-arm64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.26.3.linux-arm64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin:$(go env GOPATH)/bin' >> ~/.bashrc
source ~/.bashrc
go version
```

### 4. Node 26.1.0 + Yarn 4.18.0 (via nvm)

Node 25+ no longer bundles Corepack, so it's installed separately.

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install 26.1.0
nvm use 26.1.0
npm install -g corepack
corepack enable
corepack prepare yarn@4.18.0 --activate
```

### 5. Wails CLI

```bash
go install github.com/wailsapp/wails/v2/cmd/wails@latest
```

### 6. Clone, fetch resources, and build

```bash
cd ~
git clone https://github.com/<your-fork-or-upstream>/arduino-app-lab
cd arduino-app-lab
yarn

cd standalone-apps/app-lab-desktop/internal/board/
./download_resources.sh linux_arm64

cd ../../internal/lsp/scripts
./download_lsps.sh linux_arm64

cd ../../..
wails build -tags webkit2_41
```

The binary is produced at:
```
standalone-apps/app-lab-desktop/build/bin/ArduinoAppLab
```

### 7. Apply the rendering fix permanently

See "Known issue" above for why this is needed. Rename the real binary and put a wrapper script in its place:

```bash
cd standalone-apps/app-lab-desktop/build/bin
mv ArduinoAppLab ArduinoAppLab.bin
cat > ArduinoAppLab << 'EOF'
#!/bin/sh
export WEBKIT_DISABLE_COMPOSITING_MODE=1
exec "$(dirname "$0")/ArduinoAppLab.bin" "$@"
EOF
chmod +x ArduinoAppLab
```

Then launch `./ArduinoAppLab`.

## Credits

All code is Arduino's (GPL-3.0). This fork exists only to document a working Raspberry Pi build path until official ARM64/Raspberry Pi support ships upstream.

**Full Changelog**: https://github.com/flavioscutti/arduino-app-lab-raspberry-pi/commits/v0.0.0

---

# Arduino App Lab

This is the Arduino App Lab desktop application. It's a cross-platform application built with [Wails](https://wails.io)

<img src="docs/app-lab-screenshot.png" alt="Arduino App Lab" width="600">

## Issues

GitHub issue reports for both user and contributor facing bugs of App Lab are welcome. 

## Setup

Required:

- `node.js`
- `yarn`
- `direnv`

### direnv

This project uses [direnv](https://direnv.net/) to execute it in a dedicated shell and isolate it from the system. This gives us full control of the project and makes it easily reproducible.
This means that you do not need to have anything special installed on your machine (neither node, nor yarn), but the necessary engines will be handled automatically and will be isolated only in this project.
All you need to do is:

#### macOS and Linux
1. Make sure direnv is installed on your machine. It is usually shipped with most distributions, if not in your case please follow these instructions for installation https://direnv.net/docs/installation.html and setup for your shell https://direnv.net/docs/hook.html .
2. Run `direnv allow` the first time you enter the project.

#### Windows
([Git Bash](https://gitforwindows.org/) or an equivalent shell that supports bash is required).
1. Follow the instructions contained in this [gist](https://gist.github.com/rmtuckerphx/4ace28c1605300462340ffa7b7001c6d) from point 1, to 7. When asked to download the direnv release keep in mind that the version tested and working with our project is the `v2.19.2`.
2. Follow the installation guide from [this repository](https://github.com/AppalachiaInteractive/bash-preexec#installation) (if you are curious to know the reason for these additional steps, you can read [here](https://github.com/direnv/direnv/issues/796)).
3. Run `direnv allow` the first time you enter the project.

#### Verify
Once you've confirmed you have direnv and you've setup your environment after doing `direnv allow`, you can verify that node and yarn are isolated for your environment by typing in the shell `which node`, `node -v`, `which yarn` and `yarn -v`.
You should receive paths inside the project and respectively the version of node 26.1.0 and yarn 4.18.0


## Building (local machine)

To build a redistributable, production mode package:

1. Install Yarn with `sudo npm install --global yarn`
1. Setup npm package with `yarn`.
1. Install Wails with `go install github.com/wailsapp/wails/v2/cmd/wails@latest`.
1. Install `wget` and `jq` globally on your machine, using the right procedure for your operating system. 
    
    E.g. on macOS, you could use `brew install wget jq`.    
    On a Debian based machine you could use `apt-get install wget jq -y`.

1. `cd standalone-apps/app-lab-desktop/internal/board/`
1. Download the tools to bundle in the wails app by running: `./download_resources.sh`.
1. Run `wails build` to make the final build.

To correctly build the app on Linux, the webkit library is required. At the time of writing, on Ubuntu 24.04 `libwebkit2gtk-4.1-dev` package is available, and the corresponding wails app should be built with the command `wails build -tags webkit2_41`. You may adjust these values as needed for your OS version.

## Live Development

To run in live development mode, run `yarn start-app-lab-desktop` from the root of the project. This will run a Vite development server that will provide very fast hot reload of your frontend changes. If you want to develop in a browser
and have access to your Go methods, there is also a dev server that runs on `http://localhost:34115`. Connect
to this in your browser, and you can call your Go code from devtools.

## A note for Linux users
Some users have reported issues selecting the Arduino Q board in App Lab on Linux. A solution can be found in this [post on the Arduino forum](https://forum.arduino.cc/t/solution-arduino-app-lab-ubuntu-does-nothing-when-selecting-the-board/1411373)

