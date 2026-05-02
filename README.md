Note: I'm switching this to Armbian Noble, as there are many more precompiled binaries that are much easier to work with. You can still find the original guide in the Armbian-Trixie branch. 

# Gaming on the Khadas Edge 2 in 2026
I have spent a few weeks figuring all this stuff out, as there is little to no new documentation on this subject. Allow me to change that. This is a guide to running arm64 Steam with the latest mesa and DXVK. In theory most of this guide should work on all SBCs with RK3588 chips, but I haven't got any other than the Khadas Edge 2, so testing has not been done on other SBCs.

## Installing Linux
For this guide we're going to be using Armbian Ubuntu Noble 24.04 with the current kernel. We need the current or edge kernel, as they offer better compatibility with the bleeding edge. Downloading is easy enough; you can find it in the OOWOW or [here](https://armbian.com/boards/khadas-edge2) on Armbian's website. You can flash it in the OOWOW.

## Setting up Armbian
The first thing you're going to want to do after installing Armbian is, of course, update:
```
sudo apt update && sudo apt -y upgrade
```
I don't know about other RK3588 versions of Armbian, but for some reason this version is missing a sound server. You can install it with this command:
```
sudo apt install pipewire-audio libcanberra-pulse
```
After installing the audio server you'll need to restart it.
```
systemctl --user restart pipewire pipewire-pulse wireplumber
```
Sound should be working now.

## Installing Mesa
To install the latest Mesa drivers, we can use the mesaaco repo, which gives us bleeding-edge Mesa.
```
sudo add-apt-repository ppa:ernstp/mesaaco
sudo apt update && sudo apt -y upgrade
sudo apt install vulkan-tools mesa-vulkan-drivers
```
## Installing Box64
Box64 tends to offer better performance than FEX on RK3588 chips. Not only that, but FEX is included in the arm64 version of Steam; we can use that version instead of installing FEX globally.
```
sudo wget https://ryanfortner.github.io/box64-debs/box64.list -O /etc/apt/sources.list.d/box64.list
wget -qO- https://ryanfortner.github.io/box64-debs/KEY.gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/box64-debs-archive-keyring.gpg 
sudo apt update && sudo apt install -y box64-rk3588

```

## Installing Steam
To Install the arm native version of Steam you first need to install the x86_64 version of Steam:
```
wget https://raw.githubusercontent.com/ptitSeb/box64/main/install_steam.sh
bash install_steam.sh
```
You can run Steam with:
```
box64 steam
```
Once you reach the login screen close Steam completely.

You'll need this dependency for arm64 Steam:
```
sudo apt install libopenal1
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

You should now be able to run Steam from the command below, although you may have to restart Steam a few times to get it to work.
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

To run a game with Valve's integrated FEX, you just need to start a game with `Proton 11.0 (ARM64, Local)`.

It seems that running the Steam client natively on ARM breaks support for running 32-bit games with box64. To fix this, enable wow64.
```
PROTON_USE_WOW64=1 %command%
```

## DXVK and Zink
Here is where we encounter my biggest headache and the thing that took me the longest to figure out. The thing is the Mali G610, the GPU used by the RK3588, is missing components that make it compatible with normal Vulkan applications, mainly Vulkan extensions, but it's also missing multiple hardware descriptors, the two most important being `fillModeNonSolid`, and `shaderClipDistance`. A lot of Vulkan renderers use these two missing Vulkan hardware descriptors, including DXVK and Zink. However, all hope is not lost; thanks to a community of gamers on the Orange Pi 5, we now have patched versions of DXVK as well as a way to run Zink.

To run DXVK, you're going to need one of the builds you can find below. I personally recommend trying the DXVK-Sarek-Zayada build, as it's the one I've had the best results with, but any of them should work. Note: Some games may have better compatibility with some DXVK versions; for example, Left for Dead 2 works best with DXVK-stripped v1.5.5.

<table>
	<tr>
		<td>DXVK-Stripped</td>
		<td><a href="https://github.com/khanh-it/dxvk/releases/tag/releases">Download</a></td>
	</tr>
</table>

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
MESA_GL_VERSION_OVERRIDE=4.6 MESA_GLSL_VERSION_OVERRIDE=460 MESA_LOADER_DRIVER_OVERRIDE=zink GALLIUM_DRIVER=zink LIBGL_KOPPER_DRI2=true %command%
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
