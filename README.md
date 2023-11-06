## Armbian for RK3528 TV-box

Tested on Vontar DQ08 and H96 Max M1.

* H96 Max M1 is not recommended due to lack of essential ports.

### Status

- HDMI-video: ok
- HDMI-audio: silent
- TV-video: ok
- TV-audio: ok
- GPU: ok, but it's old and slow
- Bluetooth: haven't tried
- Wi-Fi: there is a driver, you need firmware (can be taken from Android), haven't tried

### Building Armbian for RK3528 TV-box

This Armbian configuration is based on the one for `hinlink-h28k`, which is the first RK3528 device in Armbian. The kernel config is a mix of the configs from the preinstalled Android and the config for `hinlink-h28k`. I disabled the build of many drivers that, in my opinion, will not be used on a TV-box.

Download `armbian-build` and apply the patch:

```
$ git clone --depth=1 https://github.com/armbian/build armbian-build
$ cp -R armbian-patch/* armbian-build/
```

Build:

```
$ cd armbian-build
$ ./compile.sh build BOARD=rk3528-tvbox BRANCH=legacy BUILD_DESKTOP=yes BUILD_MINIMAL=no DESKTOP_APPGROUPS_SELECTED= DESKTOP_ENVIRONMENT=mate DESKTOP_ENVIRONMENT_CONFIG_NAME=config_base EXPERT=yes KERNEL_CONFIGURE=no KERNEL_GIT=shallow RELEASE=jammy
```

* These are the options I'm testing with. You can change them to your taste, but I haven't tested other options.

### Making .dtb

Go to the `devicetree` folder.

First prepare `*.dtsi` from Linux:

```
$ cp orig/*.dtsi .
$ patch -p1 -i rk3528-tvbox.patch
```

This will make 100% accurate `.dtb` file that I found in Android preinstalled on the device:

```
$ make NAME=rk3528-vontar-dq08
```

This will make `.dtb` adapted for Armbian build:

```
$ make NAME=rk3528-vontar-dq08 PRESET=LINUX
```

Then copy this `.dtb` to the first sector of the Armbian image in `<image-boot-part>/dtb/rockchip/`. And update the `.dtb` name in `armbianEnv.txt`.

### Booting from USB

Update U-Boot in EMMC to U-Boot from an Armbian image built using these patches. It was built with `CONFIG_ROCKCHIP_USB_BOOT=y` in U-Boot defconfig.

* USB 3.x devices do not work. Because for some reason USB 3.x flash drives are only compatible with USB 2.0, and for some reason U-Boot only detects devices that support USB 1.1. USB 2.0 flash drives support USB 1.1. USB 3.x devices will work with Linux after booting.

An example of how to extract `uboot.img` from an Armbian image:

```
$ dd if=Armbian.img bs=1M count=4 skip=8 > uboot.img
```

And how to write this in EMMC:

```
$ rkflashtool w 0x4000 0x2000 < uboot.img
```

* Rewriting U-Boot in EMMC on H96MAX M1 is dangerous - this TV box does not have an SD card slot, so you may break it with an incompatible U-Boot image. Do it at your own risk!

### Known Issues

#### No signal on HDMI

This only happens to me sometimes. I can access Armbian via SSH, but I don't know how to get it to use HDMI. This seems to happen when the HDMI connection is not detected during boot.

#### No HDMI-audio

The driver is loaded, `speaker-test` and `aplay` pretend to play sound on the device, but it's silent.
HDMI-audio works from Android, so it's not broken on the device.

#### Mismatch mclk (fixed)

You will see such errors in `dmesg`:

```
rockchip-sai ffb90000.sai: Mismatch mclk: 11289600, expected 2822400 (+/- 1Hz)
rockchip-sai ffb90000.sai: ASoC: error at snd_soc_dai_hw_params on ffb90000.sai: -22
rockchip-sai ffb70000.sai: Mismatch mclk: 5644800, expected 2822400 (+/- 1Hz)
rockchip-sai ffb70000.sai: ASoC: error at snd_soc_dai_hw_params on ffb70000.sai: -22
```

Use this to fix:

```
$ amixer -c 0 set 'Clk Auto' on
$ amixer -c 1 set 'Clk Auto' on
```

After this, TV-audio will work, but HDMI audio remains silent.

The setting is saved between boots.

But to make things easier, I made a [patch](armbian-patch/patch/kernel/rk3528-tvbox-legacy/sai_clk_auto_on.patch) that enables this option by default.

#### U-Boot crashes after it can't find a boot logo (fixed)

The device tree contains logo names, these logos must be located within the Rockchip resource format, and not just in files in the boot section. And if U-Boot can't find the logo, it may crash - there's a bug somewhere in the Rockchip part of U-Boot.

```
No resource file: 
VP0 fail to load kernel logo
No resource file: 
VP1 fail to load kernel logo
"Synchronous Abort" handler, esr 0x96000010
```

This is random, but depends on the garbage in memory, which depends on the specific Armbian build, if you don't fix this, then some Armbian builds will boot, while some will crash in the U-Boot.

It looks like you can disable this logo code by removing `CONFIG_DRM_ROCKCHIP=y` from U-Boot defconfig. But after this HDMI will not work, so the initialization that comes with this logo code seems to be important.

I decided not to search for the bug, but simply [replaced](armbian-patch/patch/u-boot/legacy/board_rk3528-tvbox/gray_square_logo.patch) the code that loads a logo image with code that returns a gray square. Ugly, but it works.

#### rkflashtool can't read past 0x10000 sectors (fixed)

This is an artificial limitation introduced into Rockchip's U-Boot code. This prevents you from making a backup of Android.

I [disabled](armbian-patch/patch/u-boot/legacy/board_rk3528-tvbox/no_read_limit.patch) the restriction.

#### No video with CONFIG_DRM_IGNORE_IOTCL_PERMIT (fixed)

Rockchip drivers have the hack to use hardware video encoding/decoding drivers without root rights:
`CONFIG_DRM_IGNORE_IOTCL_PERMIT=y`

* There is a typo in the name of this kernel setting, it should be **IOCTL**, but it's written as **IOTCL**. This is how it is committed to `linux-rockchip`.

But then the video driver crashes at the boot stage:
```
Call trace:
 drm_getunique+0x38/0x9c
 drm_ioctl_kernel+0xac/0x100
 drm_ioctl+0x2f8/0x344
 vfs_ioctl+0x30/0x50
 __arm64_sys_ioctl+0x80/0xb4
 el0_svc_common.constprop.0+0xdc/0x18c
 do_el0_svc+0x8c/0x98
 el0_svc+0x20/0x30
 el0_sync_handler+0xb4/0x134
 el0_sync+0x1a0/0x1c0
```

This [patch](armbian-patch/patch/kernel/rk3528-tvbox-legacy/fix_drm_getunique_crash.patch) fixes it.

#### Graphics artifacts in the performance build

When I tried to build a kernel optimized for performance:

```
CONFIG_CC_OPTIMIZE_FOR_PERFORMANCE=y
# CONFIG_CC_OPTIMIZE_FOR_SIZE is not set
```

Then I noticed the mouse cursor blinking and strange artifacts that it leaves on the desktop background.

There seems to be some code in the graphics drivers that is not compiled correctly.

