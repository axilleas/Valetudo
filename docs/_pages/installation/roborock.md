---
title: Roborock
category: Installation
order: 10
---
# Roborock Installation Guide

This guide applies to the following robot models
* Gen 1 Xiaomi Mi SDJQR02RR aka Mi Robot Vacuum *rockrobo.vacuum.v1*
* Gen 2 Roborock S50/S51/S55 (depending on color) *roborock.vacuum.s5*

Everything else is unrootable (yet) and therefore not supported by Valetudo.<br/>
This includes the S6 as well as the S5 Max.

## Preamble
Valetudo is not a custom firmware.
It is simply an alternative App implementation + mock cloud which runs on the robot itself.<br/>

To do that, some secret data is required. Those being the `did`, the `cloudKey` and the current `local token`.
Running on the robot itself enables Valetudo to access those as well as work while in AP mode.

It's also very neat to have a completely self-contained appliance with a webinterface.

Therefore, installing Valetudo simply means taking the stock firmware and injecting Valetudo into it.<br/>
Sadly though, this process has to be done by each user indivually because hosting firmware images with Valetudo preinstalled would probably be copyright infringement.

## Building the Firmware Image
For this step, a Linux based operating system is required, since we need to mount the *ext4 file System image* of the stock firmware.

Sadly, neither OSX nor WSL (the Windows Subsystem for Linux) contain ext4 drivers so you definitely need some kind of Linux installation.
A VM should be sufficient to build the firmware image, though.

### Alternatives
If you don't have a Linux based operating system at hand or you don't want to build the image yourself, you can skip the Image Building steps here by using Dennis's Dustbuilder: https://builder.dontvacuum.me/


### Dependencies
There are a few dependencies required for building the image. Please refer to your Linux distributions documentation to find out how to install them.
* bash
* openssh (for ssh-keygen)
* ccrypt
* sed
* dos2unix

### Root Access
If you plan on being able to connect to the robot via SSH, you will need a public/private ssh keypair. **This is not required to run valetudo.**
It's useful to fetch logs and assist the development if you encounter any bugs, though.

If you do not have a keypair yet, you can generate one with the following command
```
ssh-keygen -C "your_email@example.com"
```
Per default, the generated keys will be created in `~/.ssh`. 
If you choose to create the keys in another location, remember your chosen location for later.

### Fetching the original firmware
It is recommended to fetch the firmware from the official sources.

**Gen1**

```
https://cdn.awsbj0.fds.api.mi-img.com/updpkg/[package name]
https://cdn.awsde0.fds.api.mi-img.com/updpkg/[package name]

Example: https://cdn.awsbj0.fds.api.mi-img.com/updpkg/v11_004004.amhd98763.fullos.pkg
```

**Gen2**

```
https://dustbuilder.xvm.mit.edu/pkg/s5/[package name]
https://cdn.awsbj0.fds.api.mi-img.com/rubys/updpkg/[package name]
https://cdn.cnbj2.fds.api.mi-img.com/rubys/updpkg/[package name]
https://cdn.cnbj0.fds.api.mi-img.com/rubys/updpkg/[package name]
https://cdn.awsde0.fds.api.mi-img.com/rubys/updpkg/[package name]

Example: https://dustbuilder.xvm.mit.edu/pkg/s5/v11_002008.fullos.fd043420-6ddb-4e54-bdb7-a8deec19f0fd.pkg
```

### Image Building
It is recommended to use [https://github.com/zvldz/vacuum](https://github.com/zvldz/vacuum) to build the image.

`--valetudo-path` expects a path to a folder containing two things:

 * The source code of [Valetudo](https://github.com/Hypfer/Valetudo)
 * And a binary named `valetudo`. Refer to https://github.com/Hypfer/Valetudo/releases to fetch the latest valetudo binary.

You can create a folder with all the needed things with the commands like:

```
git clone https://github.com/Hypfer/Valetudo.git
cd ./Valetudo
wget https://github.com/Hypfer/Valetudo/releases/latest/download/valetudo
```

Please refer to this command-line example and edit it according to your setup:
```
./builder_vacuum.sh     --run-custom-script=ALL \
                        --timezone=Europe/Berlin \
                        --ntpserver=pool.ntp.org \
                        --public-key=~/.ssh/id_rsa.pub \
                        --enable-greeting \
                        --disable-logs \
                        --replace-adbd \
                        --valetudo-path=./Valetudo \
                        --replace-miio \
                        --enable-dns-catcher \
                        -f path_to_firmware.pkg
```

## Flashing the firmware image

After the successful build of the firmware image, we can tell the robot to download and flash it.

First, we need to create a virtual environment for it in python. For this the following packages need to be installed:

* python3
* python3-pip
* python3-venv

```
cd ..
mkdir flasher
cd flasher
python3 -m venv venv
```

and install the required miio python packages:

```
source venv/bin/activate
pip3 install wheel
pip3 install python-miio
cd ..
```

Connect to your robot's Wi-Fi Access Point and run the following command to aquire your token:
`mirobo --debug discover --handshake true`

If your robot doesn't show up check if you have multiple connected network interfaces. Either disable all other (those not connected to your robots Wi-Fi) or use a VM which you explicitly connect to your hosts Wi-Fi interface. Another possibility is an internal firewall blocking it. On RedHat-based Linux systems using Firewalld (CentOS, Fedora, etc.), make sure the firewall zone for your connection to the robot's Wi-Fi Access Point is set to "trusted" instead of "public".

```
mirobo --ip 192.168.8.1 --token XXXXXXXXXXXXXXXX update-firmware --ip YOUR_IP_ADDRESS path/to/built/image.pkg
```

If you're upgrading Valetudo to a new version, you need to replace `192.168.8.1` with the robot's current IP address. Also please keep the distance between your Wi-Fi antenna and your robot as short as possible or the connection might get lost.

After the successful transfer of the image to the robot, the robot will start flashing the image. This will take about 5~10 minutes. After the process is done, the robot will state that the update was successful.
You should then reboot the Robot either via ssh command `ssh root@192.168.8.1` and typing `reboot` or simply by taking it out of dock an push the ON switch to prevent valetudo stuck on LOADING STATE???

### Firmware Installation fails
#### ... before the download bar appears:

 * Firewall active? - Disable your personal firewall.
 * Using a VM to flash the image? - Try to flash the image from your Host (just copy the firmware image)
 * Token wrong? - Did you initiate a WiFi reset on the robo? Then you have to refetch the token, see above.
 * Your PC does not know how to route, is more than one network interfaces active? Maybe disable LAN?
 * Wrong IP address on your WiFi? - Check that DHCP is active on your WiFi device.

#### ... after the download bar appeared:

 * Did you make an update of the robot firmware via the Xiaomi App? Then go back to original using factory reset: while holding the plug button shortly press the reset button.
 * Distance between Wifi devices is to big. Try putting the robo near your PC.
 * Battery is lower than 20%. Please Charge. Place the Vacuum in the dock.

## Connect your robot to your Wifi

To connect the robot to your home Wifi, just connect to http://192.168.8.1 and use Valetudos settings dialog to enter your wifi credentials. Please note that only *WPA2-PSK* is supported.
After updating the Wifi settings, you should reboot your robot. 

## Open Valetudo
You need to get the IP of your robot (e.g. from your router) and connect to it using your browser e.g. http://192.168.Y.Z