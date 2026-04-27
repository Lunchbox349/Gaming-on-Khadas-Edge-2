# Gaming on the Khadas Edge 2 in 2026
I have spent a few weeks figuring all this stuff out, as there is little to no new documentation on this subject. Allow me to change that. This is a guide to running arm64 Steam with the latest mesa and DXVK. Just a fair warning: this guide is not for the faint of heart; we are going to get deep into the technical weeds with this one. In theory most of this guide should work on all SBCs with RK3588 chips, but I haven't got any other than the Khadas Edge 2, so testing has not been done on other SBCs.

## Installing Linux
For this guide we're going to be using Armbian Debian 13 Trixie with the current kernel. This nets us both the latest stable kernel and the latest Debian version; this is very important for what we're doing. Downloading is easy enough; you can find it in the OOWOW or [here](https://armbian.com/boards/khadas-edge2) on Armbian's website. You can flash it in the OOWOW.

## Setting up Armbian
The first thing you're going to want to do after installing Armbian is, of course, update:
```
sudo apt update && sudo apt -y upgrade
```
After that you're going to need a desktop as well as a web browser. The `rtkit` package is to fix an error with pipewire.
```
sudo apt install gnome-core firefox-esr rtkit
```
After installing GNOME, reboot your system, and you should be greeted with the GNOME login manager. Once you log in, you're probably going to notice that the WiFi is missing. Don't worry, it's still working; you just need to change it to use `NetworkManager` in order for Gnome to be able to see the WiFi settings.
```
sudo apt install network-manager
```
Now you need to edit the Netplan settings to tell it to use `NetworkManager` instead of `Networkd`.
```
sudo nano /etc/netplan/10-dhcp-all-interfaces.yaml
```
Next you just need to change the renderer to `NetworkManager`. It should go from:
```
renderer: networkd
```
to
```
renderer: NetworkManager
```
After that you just need to apply the new configuration.
```
sudo netplan apply
```
Now after a few seconds you should have the WiFi stuff pop up and be functional.

## Compiling and Installing Mesa
In order to get the latest version of Mesa, you need to compile and install it yourself. Now we're not going to install it to the `usr` directory as that's bad practice and can potentially break your OS install. Instead we're going to install to the `opt` directory to prevent it from overwriting system files.

First things first you need to install the dependencies:
```
sudo apt install build-essential git clang \
	cmake pkg-config gedit \
	bison mesa-utils vulkan-tools \
	libopengl0 meson python3-packaging \
	python3-mako flex byacc libclc-19-dev \
	libdrm-dev libudev-dev llvm-dev \
	llvm-spirv-19 libllvmspirvlib-19-dev spirv-tools \
	libclang-cpp-dev libwayland-dev libwayland-egl-backend-dev \
	libxcb1-dev libxcb-randr0-dev libx11-dev \
	libxext-dev libxfixes-dev libxcb-glx0-dev \
	libxcb-shm0-dev libx11-xcb-dev libxcb-dri3-dev \
	libxcb-present-dev libxshmfence-dev libxxf86vm-dev \
	libxrandr-dev libclang-dev
```
If you run into issues with `libclc-19-dev` `llvm-spirv-19` `libllvmspirvlib-19-dev` it probably means that Debian has updated which version of `clang` and `llvm` they use. To fix this, you just need to change the 19 in each package to whatever version the current `clang` is.

You can figure out which version of `clang` Debian installed by running this command:
```
clang --version
```
The next step is to clone the Mesa repository and set up the build directory:
```
git clone https://gitlab.freedesktop.org/mesa/mesa
cd mesa
rm -rf builddir
```
After this you just need to run the command to configure Meson:
```
meson setup \
 -Dprefix=/opt/mesa \
 -Dgles2=enabled \
 -Dgallium-drivers=panfrost,zink \
 -Dvulkan-drivers=panfrost \
 -Dtools=drm-shim \
 -Dbuildtype=release \
 builddir/
```
You need to compile it on 1 thread in order to prevent a race condition that can potentially cause the build to fail.
```
meson compile -j 1 -C builddir/
```

Once it's done compiling, you need to install it:
```
meson install -C builddir/
```

Once Mesa is installed you can remove the source code:
```
rm -r ~/mesa
```

Now you need to tell Linux where the new Mesa install is:
```
echo 'LD_LIBRARY_PATH="/opt/mesa/lib/aarch64-linux-gnu"' | sudo tee -a /etc/environment.d/10-mesa.conf
echo 'VK_DRIVER_FILES="/opt/mesa/share/vulkan/icd.d/panfrost_icd.aarch64.json"' | sudo tee -a /etc/environment.d/10-mesa.conf
```
After that you need to reboot.

To test the freshly installed Mesa, you can run these 4 commands:

1. Check if the Vulkan version is correct:
```
vulkaninfo --summary
```
2. Check if the Opengl version is correct:
```
glxinfo | grep "OpenGL version"
```
3. Check if Vulkan is working correctly:
```
vkcube
```
4. Check if Opengl is working correctly:
```
glxgears
```
If all those are good, you have successfully updated Mesa without overwriting your Debian system libraries.

## Installing FEX
Install Dependencies (Note: This does not include dependencies already installed when compiling mesa.)
```
sudo apt install  binfmt-support lld libssl-dev \
	g++-x86-64-linux-gnu libgcc-14-dev-amd64-cross \
	libgcc-14-dev-i386-cross libstdc++-14-dev-i386-cross \
	squashfs-tools squashfuse qt6-declarative-dev \
	qml6-module-qtquick-controls qml6-module-qtquick-dialogs \
	qml6-module-qtquick-layouts qml6-module-qtqml-workerscript \
	qml6-module-qtquick-templates qml6-module-qtquick-window \
	libxcb-dri2-0-dev libasound2-dev libegl1-mesa-dev
```
Setup Build Directory:
```
git clone --recurse-submodules https://github.com/FEX-Emu/FEX.git
cd FEX
mkdir Build
cd Build
```
Configure with `cmake`:
```
CC=clang CXX=clang++ cmake \
	-DCMAKE_INSTALL_PREFIX=/opt/fex-emu \
	-DCMAKE_BUILD_TYPE=Release \
	-DUSE_LINKER=lld \
	-DENABLE_LTO=True \
	-DBUILD_TESTING=False \
	-DENABLE_ASSERTIONS=False \
	-DBUILD_THUNKS=True \
	-G Ninja ..
```
Build with `ninja`:
```
ninja -j4
```
Install:
```
sudo ninja install
```
Once installed you can remove the source code:
```
rm -r ~/FEX
```

Add the FEX binaries to the linux path:
```
echo 'export PATH=$PATH:/opt/fex-emu/bin' | sudo tee -a /etc/profile.d/fex-emu.sh
```
After that you need to log out and log back in one more time.

Now you need to install a rootfs:
```
FEXRootFSFetcher
```
I tested this with the Arch Linux rootfs, so I know that one should work. Make sure to choose to extract it.

Now you need to link the `~/.local/share/fex-emu` to `~/.fex-emu`
```
ln -s ~/.local/share/fex-emu ~/.fex-emu
```
Configure FEX with:
```
FEXConfig
```
Make sure to enable the Arch Linux rootfs as well as enable Vulkan and GL under libraries. Remember to hit file and then save before closing.

### Optional: Enabling binfmt for FEX (Note: Don't do this if you plan to have Box64 installed.)

Link the binfmt configs to the binfmt.d folder:
```
sudo ln -s /opt/fex-emu/lib/binfmt.d/FEX-x86_64.conf /etc/binfmt.d/
sudo ln -s /opt/fex-emu/lib/binfmt.d/FEX-x86.conf /etc/binfmt.d/
```
Restart `systemd-binfmt`:
```
sudo systemctl restart systemd-binfmt
```
## Installing Steam
To Install the arm native version of Steam you first need to install the x86_64 version of Steam:
```
wget https://raw.githubusercontent.com/ptitSeb/box64/main/install_steam.sh
bash install_steam.sh
```
You can run Steam with:
```
FEXBash steam
```
Once you reach the login screen close Steam completely.

You'll need these dependancies for arm64 Steam as well as unzip:
```
sudo apt install libgtk2.0-0t64 libsdl2-mixer-2.0-0 unzip
```
Download the Steam arm64 manifest:
```
wget  https://client-update.steamstatic.com/bins_linuxarm64_linuxarm64.zip.f523fa87fc6b9b5435a5e7370cb0d664ef53b50b  ; mv bins_linuxarm64_linuxarm64.zip.f523fa87fc6b9b5435a5e7370cb0d664ef53b50b bins_linuxarm64_linuxarm64.zip
```
Enable the Steam Beta:
```
mkdir -p ~/.local/share/Steam/package && echo publicbeta > ~/.local/share/Steam/package/beta
```
you need to extract the `bins_linuxarm64_linuxarm64.zip` to `~/.local/share/steam`.
```
unzip ./bins_linuxarm64_linuxarm64.zip -d ~/.local/share/Steam/
```

Next you need to change the permisions on the `steamrtarm64` folder.
```
chmod -R u+rwx ~/.local/share/Steam/steamrtarm64/
```
You need to link `libvpx` to prevent an error.
```
sudo ln -s /usr/lib/aarch64-linux-gnu/libvpx.so.9 /usr/lib/aarch64-linux-gnu/libvpx.so.6
```

You should now be able to run Steam from:
```
~/.local/share/Steam/steamrtarm64/steam
```
In order to launch games from the arm64 version of Steam, you need the arm64 version of the Steam Linux runtime.
```
wget https://archive.org/download/arm-64proton-runtime-64.tar/ARM64proton-Runtime64.tar.gz
```

Extract to `~/.local/share/Steam/compatibilitytools.d/`.
```
tar -xvzf ./ARM64proton-Runtime64.tar.gz -C ~/.local/share/Steam/compatibilitytools.d/
```

Now you need to create this symlink:
```
ln -s "$HOME/.local/share/Steam/linuxarm64" "$HOME/.steam/sdkarm64"
```
Remove leftover files:
```
rm ~/ARM64proton-Runtime64.tar.gz ~/bins_linuxarm64_linuxarm64.zip ~/install_steam.sh
```

To run a game with FEX add this to the Steam launch options:
```
FEXBash %command%
```
And the launch options for Box64:
```
box64 %command%
```

To run a game with Valve's integrated FEX, you just need to start a game with `Proton 11.0 (ARM64, Local)`. Make sure to remove `box64` and `FEXBash` from your launch options.

It seems that running the Steam client natively on ARM breaks support for running 32-bit games with Box32. You can either run the game in FEX or run Steam with `box64 steam` if you have Box64 installed.

## DXVK and Zink
Here is where we encounter my biggest headache and the thing that took me the longest to figure out. The thing is the Mali G610, the GPU used by the RK3588, is missing a few things that make it compatible with normal Vulkan applications, mainly Vulkan extensions, but it's also missing two Vulkan hardware descriptors `fillModeNonSolid`, and `shaderClipDistance` this makes running DXVK and Zink tricky as well as running Vulkan native games much harder. Specifically, `fillModeNonSolid` is the most important of the 2 missing hardware descriptors, as this was included in Vulkan 1.0 and basically every Vulkan game uses it. However, all hope is not lost; thanks to a community of gamers on the Orange Pi 5, we now have patched versions of DXVK as well as figured out how to run Zink.

To run DXVK, you're going to need one of the builds you can find [here.](https://github.com/khanh-it/dxvk/releases/tag/releases) I personally recommend trying the DXVK-Sarek-Zayada build, as it's the one I've had the best results with, but any of them should work.

To install you just need to extract the zip file and move the DLLs you need to the folder with the exe file for the game you're trying to run.

If it's a 64-bit game, use the DLLs in the x64 folder, and if it's a 32-bit game, use the DLLs in the x32 folder.

The DLLs necessary for each DirectX version are as follows:

DX9: `d3d9.dll`

DX10: `d3d10core.dll` `d3d11.dll` `dxgi.dll`

DX11: `d3d11.dll` `dxgi.dll`

To use Zink, you'll need to add the usual envars with an aditional `LIBGL_KOPPER_DRI2=true` to your Steam launch options.
```
MESA_LOADER_DRIVER_OVERRIDE=zink GALLIUM_DRIVER=zink LIBGL_KOPPER_DRI2=true %command%
```
If the game fails to start with Zink, you can try to force a higher OpenGL version.
```
PAN_MESA_DEBUG=gl3 MESA_GL_VERSION_OVERRIDE=4.6 MESA_GLSL_VERSION_OVERRIDE=460 MESA_LOADER_DRIVER_OVERRIDE=zink GALLIUM_DRIVER=zink LIBGL_KOPPER_DRI2=true %command%
```
## Optional: Compiling and Installing Box64
Box64 tends to get better performance on RK3588 chips. Unfortunately, however, box64 is hardcoded to install into the `usr` directory; this can cause issues with, for example, FEX's binfmt support.

First you need to set up the build environment:
```
git clone https://github.com/ptitSeb/box64
cd box64
mkdir build
cd build
```
Configure the build using `cmake`:
```
cmake .. -D ARM_DYNAREC=ON -D RK3588=1 -D CMAKE_BUILD_TYPE=RelWithDebInfo -D BOX32=ON -D BOX32_BINFMT=ON
```
Compile the build:
```
make -j4
```
Install Box64:
```
sudo make install
```
Restart systemd-binfmt:
```
sudo systemctl restart systemd-binfmt
```
Once installed you can remove the source code:
```
rm -r ~/box64
```

## Optional: Sound Fix
If you have issues with sound stutering try using this command:
```
echo 'export PULSE_LATENCY_MSEC=60' | sudo tee -a /etc/profile.d/soundfix.sh
```
After that, just log out and log back in, and the sound should be fixed.
## Special Thanks
**VennStone**, from the interfacing Linux forum for figuring out how to run the arm64 version of Steam.

**KhanhDTP**, from the Armbian forum for their work on dxvk-stripped as well as their help getting games running on the RK3588. 

Obviously the developers of FEX and Box64 as well.
