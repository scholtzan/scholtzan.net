+++
title = "Setup Guide for Embedded Systems Programming with Rust on MacOS"
date = "2019-01-06"
type = "post"
tags = ["Rust", "Embedded Systems", "Raspberry Pi", "MacOS"]
+++

It's been a while since I last was tinkering with my Raspberry Pi and worked on some electronics project. 
After a long break, I was now itching to get back to play around with some electronics and do some embedded systems programming - but this time using Rust. 
I also bought an ESP8266 and wanted to see if I can use Rust for it.
So I spent many hours during the past few days setting up my system and development environment.
Unfortunately, the setup process did not go as smoothly as I was hoping for, so I took some notes that will hopefully help me or someone else in the future when I want to repeat the setup.

## Raspberry Pi

I am using a Raspberry Pi Model A as well as a Raspberry Pi Model B+. 
The very first step was to install the newest version of [Raspbian](https://www.raspberrypi.org/downloads/raspbian/) which, in my case, was Raspbian Stretch Lite.
For this, I just followed the [installation guide](https://www.raspberrypi.org/documentation/installation/installing-images/mac.md) on the official website.

### Connecting to Raspberry Pi via Ethernet and `ssh`

I didn't want to use an external keyboard and screen to access my Raspberry Pi, I simply wanted to connect using `ssh`. 
Since both the model A does not have built-in WiFi I wanted to connect it via Ethernet to my MacBook.
The following steps enabled me in the end to establish an `ssh` connection over Ethernet:

1. Since `ssh` is by default disabled on Raspbian, a file with the name `ssh` has to be created on the SD card.
2. On MacOS, enable internet sharing: _System Preferences → Sharing_ and turn on _Internet Sharing to computers using Thunderbolt Ethernet_
3. Go to _System Preferences → Network → Thunderbolt Ethernet_ and wait for a while until Thunderbolt Ethernet shows "Self-assigned IP".
4. `ifconfig`: `bridge100` shows IP address for ssh access. The actual IP address is the one that is shown plus 1 for the last digit.
5. Connect via `ssh pi@<ip+1>`, for example `ssh pi@192.168.2.2`

### Sharing Files Between Raspberry Pi and MacOS

I wanted to get easy access to the files stored on my Raspberry Pi as well as have a simple way of transfering files from my MacBook to my Pi. 
The easiest way I found was to set up `netatalk` on my Pi and simply connect to it over Finder.

1. Install `netatalk` on Raspberry Pi: `sudo apt-get update && sudo apt-get install netatalk`
	* It might be necessary to restart netatalk: `sudo /etc/init.d/netatalk restart`
2. After few seconds "Network" should show up in Finder.
3. To connect click on "Connect as" and enter user name and password.

### Rust Cross Compilation

While it is quite easy to install Rust on the Raspberry Pi and build projects right there, the build process takes a very long time.
It is faster to instead cross compile the projects on the MacBook for the Raspberry Pi and then transfer and run the binaries.

Setting up cross compilation for the model A and B+ is similar but not the same, since the model uses ARMv6 and B+ uses ARMv7.
I decided to use [crosstool-NG](https://github.com/crosstool-ng/crosstool-ng) which will build the toolchain for both Raspberry Pis.

1. Install crosstool-NG: `brew install crosstool-ng`
2. Add new targets. For model A: `rustup target add arm-unknown-linux-gnueabihf` and for model B+: `rustup target add armv7-unknown-linux-gnueabihf`.
3. crosstool-NG can only build toolchains on case-sensitive file systems. Since my default file system is not case-sensitive I created a new one: `hdiutil create ~/bin/crosstool.dmg -volname "ctng" -size 10g -fs "Case-sensitive HFS+"`
4. To mount created file system: `hdiutil mount ~/bin/crosstool.dmg`
5. Switch to mounted file system on command line.
6. crosstool-NG supports quite a lot of toolchains which can be listed using `ct-ng list-samples`
7. Set toolchain to be built, for model A: `ct-ng armv6-rpi-linux-gnueabi` and B+: `ct-ng armv7-rpi2-linux-gnueabihf`
8. Configure build: `ct-ng menuconfig`
	* _Paths and misc options → crosstool-NG behavior_ set path to where toolchain should get installed (somewhere on the case-sensitive file system)
9. Build the toolchain: `ct-ng build`
	* The build process can take very long (30min - 1.5h). Unfortunately, during this process I encountered several errors which could be resolved by simply googling them, but it still took quite some time. The errors I encountered were:
		* [https://github.com/crosstool-ng/crosstool-ng/issues/315](https://github.com/crosstool-ng/crosstool-ng/issues/315)
		* [https://github.com/crosstool-ng/crosstool-ng/issues/1002](ttps://github.com/crosstool-ng/crosstool-ng/issues/1002)
		* [https://github.com/crosstool-ng/crosstool-ng/issues/723](https://github.com/crosstool-ng/crosstool-ng/issues/723)
10. After the build is done add to `$PATH`: `export PATH="${PATH}:/path/to/toolchain/armv6-rpi-linux-gnueabi/bin"`
11. Add linker to `~/.cargo/config`:
	```
	[target.arm-unknown-linux-gnueabi]
	linker = "/path/to/toolchain/armv6-rpi-linux-gnueabi/bin/armv6-rpi-linux-gnueabi-gcc"

	[target.armv7-unknown-linux-gnueabihf]
	linker = "/path/to/toolchain/armv7-rpi2-linux-gnueabihf/bin/armv7-rpi2-linux-gnueabihf-gcc"
	```
12. To build cargo project: `cargo build --target=arm-unknown-linux-gnueabi`


### Enabling GPIO Pins

When running a Rust project on my Pi that accesses the GPIO pins I got the following error: `thread 'main' panicked at 'called Result::unwrap() on an Err value: Io(Os { code: 13, kind: PermissionDenied, message: "Permission denied" })', libcore/result.rs:1009:5`.
The fix was to explicitly export the pins I access and set the direction:

* `echo "18" > /sys/class/gpio/export`
* `sudo sh -c 'echo out > /sys/class/gpio/gpio18/direction'`

For accessing the GPIO pins I am using the [`sysfs-gpio`](https://github.com/rust-embedded/rust-sysfs-gpio) crate.


## ESP 8266

I found some [instructions](https://github.com/emosenkis/esp-rs) to install the toolchain for building ESP8266 firmware in Rust. After being rather unsucessful to get it working on my MacBook, I decided to give the docker container a try. While creating the image and even after the image was running, I encountered a lot of error and problems.
Due to this painful experience, I actually wanted to publish the created image on [Docker Hub](https://hub.docker.com/) so that it would be easier for others to get started. However, it seems that they have some [issues for a while](https://github.com/docker/hub-feedback/issues/631), where you cannot create a new account there. 

1. Install bootloader utility: `pip3 install esptool` 
2. Clone repo from [https://github.com/emosenkis/esp-rs](https://github.com/emosenkis/esp-rs)
3. Before building the image increase memory to 6GB and swap to 1.5GB
4. `docker build .`
5. Run image: `docker run -i -v /local/path:/home -w /home -t --entrypoint /bin/bash 53a34e87fe3f`
6. Resolve issues:
	* Many of the issues were discussed in [https://github.com/emosenkis/esp-rs/issues/15](https://github.com/emosenkis/esp-rs/issues/15)
	* libclang issue with bindgen
		* `apt-get install clang`
	* `export USER=root`
	* `rustup component add rustfmt`
	* error: `ln: failed to create symbolic link '.esp-rs-compiled-lib': File exists`
		* `rm .esp-rs-compiled-lib`
7. To use add following to `Cargo.toml`:
	```
	[dependencies]
	embedded-hal = { version = "0.2.1", features = ["unproven"] }
	esp8266-hal = "0.0.1"
	libc = { version = "0.2.42", default-features = false }
	```
	* remove `Rust = 2018`
8. Run `build.sh` in project directory
9. Result in `.pioenvs/nodemcuv2/firmware.bin`
10. Can be flashed onto ESP using `esptool.py --port /dev/tty.wchusbserial1410 erase_flash`

For accessing GPIO pins the crate [`esp8266-hal`](https://github.com/emosenkis/esp8266-hal) needs to be used instead of `sysfs-gpio`.

## Closing Remarks

While I have only tested the setup with a Raspberry Pi model A and B+, I am confident that the setup for other models is almost the same. 
In general, the whole setup process was mostly running into errors, using Google to find a fix for the issue, fixing some configuration and repeat.
I was hoping it would be a relatively quick process, but in the end it took several days and hours.
Nevertheless, I am pretty happy that everything seems to be working now and that I can start working on some cool projects.
