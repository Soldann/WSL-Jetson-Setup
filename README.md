# WSL-Jetson-Setup
Setup USB passthrough and Internet passthrough to setup a Jetson through WSL

# JETSON SETUP WITH WSL2

Follow the Microsoft guide to install WSL2 and Ubuntu onto your Windows machine.

Throughout this guide, you will see headings for HOST and JETSON. HOST means execute on the host windows computer, JETSON means to run on the Jetson (You will need a keyboard, and monitor)

## Build Custom WSL Kernel for USB Forwarding
If you have Windows 11, you may skip this step. Otherwise, follow along to enable USB passthrough with WSL


Open Ubuntu in WSL by navigating to the start menu and opening Ubuntu

Update sources:
```
~$ sudo apt update
```

Install prerequisites to build Linux kernel:
```
~$ sudo apt install build-essential flex bison libssl-dev libelf-dev libncurses-dev autoconf libudev-dev libtool
````

Find out the name of your Linux kernel:
```
~$ uname -r
5.10.16.3-microsoft-standard-WSL2
```
Clone the WSL 2 kernel. Typically kernel source is put in /usr/src/[kernel name]:
```
~$ sudo git clone https://github.com/microsoft/WSL2-Linux-Kernel.git /usr/src/5.10.16.3-microsoft-standard-WSL2
~$ cd /usr/src/5.10.16.3-microsoft-standard-WSL2
```

Checkout your version of the kernel:
```
/usr/src/5.10.16.3-microsoft-standard-WSL2$ sudo git checkout linux-msft-wsl-5.10.16.3
```

Copy in your current kernel configuration:
```
/usr/src/5.10.16.3-microsoft-standard-WSL2$ sudo cp /proc/config.gz config.gz
/usr/src/5.10.16.3-microsoft-standard-WSL2$ sudo gunzip config.gz
/usr/src/5.10.16.3-microsoft-standard-WSL2$ sudo mv config .config
```

Run menuconfig to select what kernel modules you'd like to add:
```
/usr/src/5.10.16.3-microsoft-standard-WSL2$ sudo make menuconfig
```

Navigate in menuconfig to select the USB kernel modules you'd like. These are the features I enabled in the menuconfig:
```
Device Drivers->USB support[*]
Device Drivers -> USB Support -> USB announce new devices
Device Drivers->USB support->Support for Host-side USB[M]
Device Drivers->USB support->Enable USB persist by default[*]
Device Drivers->USB support->USB Modem (CDC ACM) support[M]
Device Drivers->USB support->USB Mass Storage support[M]
Device Drivers->USB support->USB/IP support[M]
Device Drivers->USB support->VHCI hcd[M]
Device Drivers->USB support->VHCI hcd->Number of ports per USB/IP virtual host controller(8)
Device Drivers->USB support->Number of USB/IP virtual host controllers(1)
Device Drivers -> USB Support -> USB/IP -> Debug messages for USB/IP
Device Drivers->USB support->USB Serial Converter support[M]
Device Drivers->USB support->USB Serial Converter support->USB FTDI Single Port Serial Driver[M]
Device Drivers->USB support->USB Physical Layer drivers->NOP USB Transceiver Driver[M]
Device Drivers->Network device support->USB Network Adapters[M]
Device Drivers->Network device support->USB Network Adapters->[Deselect everything you don't care about]
Device Drivers->Network device support->USB Network Adapters->Multi-purpose USB Networking Framework[M]
Device Drivers->Network device support->USB Network Adapters->CDC Ethernet support (smart devices such as cable modems)[M]
Device Drivers->Network device support->USB Network Adapters->Multi-purpose USB Networking Framework->Host for RNDIS and ActiveSync devices[M]
```

Build the kernel and modules with as many cores as you have available (-j [number of cores]). This may take a few minutes depending how fast your machine is:
```
/usr/src/5.10.16.3-microsoft-standard-WSL2$ sudo make -j 8 && sudo make modules_install -j 8 && sudo make install -j 8
```

After the build completes you'll get a list of what kernel modules have been installed. Mine looks like:
```
  INSTALL drivers/hid/hid-generic.ko
  INSTALL drivers/hid/hid.ko
  INSTALL drivers/hid/usbhid/usbhid.ko
  INSTALL drivers/net/mii.ko
  INSTALL drivers/net/usb/cdc_ether.ko
  INSTALL drivers/net/usb/rndis_host.ko
  INSTALL drivers/net/usb/usbnet.ko
  INSTALL drivers/usb/class/cdc-acm.ko
  INSTALL drivers/usb/common/usb-common.ko
  INSTALL drivers/usb/core/usbcore.ko
  INSTALL drivers/usb/serial/ftdi_sio.ko
  INSTALL drivers/usb/phy/phy-generic.ko
  INSTALL drivers/usb/serial/usbserial.ko
  INSTALL drivers/usb/storage/usb-storage.ko
  INSTALL drivers/usb/usbip/usbip-core.ko
  INSTALL drivers/usb/usbip/vhci-hcd.ko
  DEPMOD  5.10.16.3-microsoft-standard
```

Build USB/IP tools.
```
cd tools/usb/usbip
sudo ./autogen.sh
sudo ./configure
sudo make install -j 8
```
Copy tools libraries location so usbip tools can get them.
```
sudo cp libsrc/.libs/libusbip.so.0 /lib/libusbip.so.0
```
Install usb.ids so you have names displayed for usb devices.
```
sudo apt-get install hwdata
```

Make a script in your home directory to modprobe in all the drivers. Be sure to modprobe in usbcore and usb-common first. I called mine `startusb.sh`. Mine looks like:
```
#!/bin/bash
sudo modprobe usbcore
sudo modprobe usb-common
sudo modprobe hid-generic
sudo modprobe hid
sudo modprobe usbnet
sudo modprobe cdc_ether
sudo modprobe rndis_host
sudo modprobe usbserial
sudo modprobe usb-storage
sudo modprobe cdc-acm
sudo modprobe ftdi_sio
sudo modprobe usbip-core
sudo modprobe vhci-hcd
echo $(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')
```
The last line spits out the IP address of the Windows host. Helpful when you are attaching devices from it.
If you see some crap about /bin/bash^M: bad interpreter: No such file or directory it's because you have cr+lf line endings. You need to convert your shellscript to just lf.

Mark it as executable:
```
~$ sudo chmod +x startusb.sh
```


From the root of the repo, copy the image.
```
cp arch/x86/boot/bzImage /mnt/c/Users/<user>/usbip-bzImage
```

You now need to point WSL to use the kernel we just compiled
Create a .wslconfig file on /mnt/c/Users/<user>/ and add a reference to the created image with the following.
```
[wsl2]
kernel=c:\\users\\<user>\\usbip-bzImage
```

Restart WSL. In a CMD window in Windows type:
```
C:\Users\rpasek>wsl --shutdown
```

Open WSL again by going to start menu and opening Ubuntu and run your script:
```
~$ ./startusb.sh
```
Note that any time you restart your computer (or WSL) you will have to run `./startusb.sh` again.

Check in dmesg that all your USB drivers got loaded:
```
~$ sudo dmesg
````

If so, cool, you've added USB support to WSL. 


If you are running into issues, see the source for this step for possible troubleshooting:
[GitHub - rpasek/usbip-wsl2-instructions](https://github.com/rpasek/usbip-wsl2-instructions)
[WSL support · dorssel/usbipd-win Wiki · GitHub](https://github.com/dorssel/usbipd-win/wiki/WSL-support)


## Attach USB to WSL

### POWERSHELL
We will have to install USBIP in Windows to get it to forward the USB to WSL.
```
winget install usbipd
```

Once installed, you can generate a list of usb devices attached to Windows:
```
C:\WINDOWS\system32>usbipd list -l
```

Prior to flashing, the Jetson will be labelled `APX` in the list printed above. After flashing, it will be labelled `Remote NDIS Compatible Device`. 

The BUSID of the device I want to bind to is 2-9. Bind to it with: 
```
C:\WINDOWS\system32>usbipd bind --busid=2-9
```
If you run `usbipd list` again after binding, you should see the state become `Shared` assuming everything went correctly

You will have to do repeat this twice: Once before flashing and once after flashing.


### WSL

Now on Linux get a list of available USB devices: The remote IP should be the result printed out by the `./startusb.sh` script.
```
~$ sudo usbip list --remote=172.30.64.1
```

The busid of the device I want to attach to is 2-9. Attach to it with:
```
~$ sudo usbip attach --remote=172.27.144.1 --busid=2-9
```

Your USB device should be usable now in Linux. Check dmesg to make sure everything worked correctly:
```
~$ sudo dmesg
```

You will need to write a script to spam attach the Jetson, as it will periodically disconnect and reconnect during the flashing process.
Note that in the following script, there are two `usbip attach` commands. That's because the Jetson will have a different `busid` before and after flashing. You will need to repeat this process after flashing.
```
#!/bin/bash
while true
do
    sudo usbip attach --remote=172.27.144.1 --busid=2-9
    sudo usbip attach --remote=172.27.144.1 --busid=2-9
    sleep 0
done
```

To verify that everything is working, run
```
lsusb
```
You should see either an APX or NVIDIA device attached.

## Flash to Jetson
Install SDKManager for Ubuntu using the `.deb` file provided by NVIDIA.

Run
```
sdkmanager --cli --query
```
to show all possible commands to flash to the Jetson. I used:
```
sdkmanager --cli install --logintype devzone --product Jetson --version 5.0.1 --targetos Linux --host --target JETSON_AGX_XAVIER_TARGETS --flash all --additionalsdk 'DeepStream 6.1'
```

Stop SDKManager once it finishing flashing but before installing packages into the Jetson. You will have to go back to the above steps and re-forward the USB connection to the JETSON once it finishes rebooting.

## Setup SSH Connection to JETSON via USB

After flashing, SDKMANAGER's auto-setup to connect the JETSON to the host computer will not work, so we will have to setup the SSH and internet connection manually. On each device, run:

### HOST
```
sudo ip link set usb0 up
sudo ifconfig usb0 192.168.55.100 netmask 255.255.255.0
```

### JETSON
```
sudo ip link set usb0 up
sudo ifconfig usb0 192.168.55.1 netmask 255.255.255.0
```

`ifconfig` should show the corresponding ip under the `usb0` interface.

NOTE: If there are multiple `usb0` interfaces, change `usb0` in the above commands to the interface corresponding with the MAC address of the NVIDIA device. To find the MAC address of the JETSON, run `dmesg --follow` while plugging in the USB.

## SETUP INTERNET PASSTHROUGH
### HOST
We will have to edit iptables to forward WSL's internet connection to the Jetson.
```
sudo iptables -A FORWARD -o eth0 -i usb0 -s 192.168.55.0/24 -m conntrack --ctstate NEW -j ACCEPT
sudo iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -t nat -F POSTROUTING
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```
 
In `/etc/sysctl.conf`, uncomment:
```
#net.ipv4.ip_forward=1
```
... so that it reads:
```
net.ipv4.ip_forward=1
```

IF you want to save the iptables so they persist among restarts, run:
```
sudo iptables-save | sudo tee /etc/iptables.sav
iptables-restore < /etc/iptables.sav
```

### JETSON
Run `sudo ip route list` and see if you can find `default via 192.168.55.100`. You can skip this step if that is true.

Disable networking
```
sudo /etc/init.d/network-manager stop
```
Configure routing
```
sudo ip route add default via 192.168.55.100
```
Re-enable networking
```
sudo /etc/init.d/network-manager restart
```

## Finish Jetson Setup

Finally, use the same command you used to flash to the JETSON, but change the `--flash` argument to `skip`. This will resume immediately at the installation point of sdkmanager

```
sdkmanager --cli install --logintype devzone --product Jetson --version 5.0.1 --targetos Linux --host --target JETSON_AGX_XAVIER_TARGETS --flash skip --additionalsdk 'DeepStream 6.1'
```
