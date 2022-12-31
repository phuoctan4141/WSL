# Building your own USB/IP

Update resources (assuming apt, you may need to use yum or another package manager).

```sh
sudo apt update && sudo apt upgrade
```

Install prerequisites.

```sh
sudo apt install build-essential flex bison libssl-dev libelf-dev libncurses-dev autoconf libudev-dev libtool
```

```sh
sudo apt install dwarves
```

Clone kernel that matches wsl version. To find the version you can run.

```sh
uname -r
```

The kernel can be found at: https://github.com/microsoft/WSL2-Linux-Kernel

Clone the kernel repo, then checkout the branch/tag that matches your kernel version; run ```uname -r``` to find the kernel version.

```sh
git clone https://github.com/microsoft/WSL2-Linux-Kernel.git
cd WSL2-Linux-Kernel
git checkout linux-msft-wsl-5.10.43.3
```

Copy current configuration file.

```sh
sudo cp /proc/config.gz config.gz
sudo gunzip config.gz
sudo mv config .config
```

You may need to set CONFIG_USB=y in .config prior to running menuconfig to get all options enabled for selection.

Run menuconfig to select kernel features to add.

```sh
sudo make menuconfig
```

Select different modules according to your own needs. (Press space to select or deselect.)

```sh
These are the necessary additional features in menuconfig.
Device Drivers->USB support[*]
Device Drivers->USB support->Support for Host-side USB[M]
Device Drivers->USB support->Enable USB persist by default[*]
Device Drivers->USB support->USB Modem (CDC ACM) support[M]
Device Drivers->USB support->USB Mass Storage support[M]
Device Drivers->USB support->USB/IP support[M]
Device Drivers->USB support->VHCI hcd[M]
Device Drivers->USB support->VHCI hcd->Number of ports per USB/IP virtual host controller(8)
Device Drivers->USB support->Number of USB/IP virtual host controllers(1)
Device Drivers->USB support->USB Serial Converter support[M]
Device Drivers->USB support->USB Serial Converter support->USB FTDI Single Port Serial Driver[M]
Device Drivers->USB support->USB Physical Layer drivers->NOP USB Transceiver Driver[M]
Device Drivers->Network device support->USB Network Adapters[M]
Device Drivers->Network device support->USB Network Adapters->[Deselect everything you dont care about]
Device Drivers->Network device support->USB Network Adapters->Multi-purpose USB Networking Framework[M]
Device Drivers->Network device support->USB Network Adapters->CDC Ethernet support (smart devices such as cable modems)[M]
Device Drivers->Network device support->USB Network Adapters->Multi-purpose USB Networking Framework->Host for RNDIS and ActiveSync devices[M]
```

```sh
These are additional features required for the camera.
Device Drivers->Multimedia support[M]
Device Drivers->Multimedia support[M]->Filter media drivers[*]
Device Drivers->Multimedia support[M]->Auto ancillary drivers[*]
Device Drivers->Multimedia support[M]->Media device types->Camera and video grabbers[*]
Device Drivers->Multimedia support[M]->Video4Linux options->V4L2 sub-device userspace API[*]
Device Drivers->Multimedia support[M]->Media drivers->Media USB Adapters[*]
Device Drivers->Multimedia support[M]->Media drivers->Media USB Adapters[*]->USB Video Class(UVC)[*]
Device Drivers->Multimedia support[M]->Media drivers->Media USB Adapters[*]->UVC input evnets device support[*]
```

⚠️ These instructions have changed.
Previously, it was recommended to enable "Debug messages for USB/IP". However, debug messages have a huge negative performance impact on bulk transfers. Enabling debug messages is no longer recommended.

In the following command the number '8' is the number of cores to use; run getconf _NPROCESSORS_ONLN to find the number of cores.

```sh
sudo make -j 8 && sudo make modules_install -j 8 && sudo make install -j 8
```

Build USB/IP tools.

```sh
cd tools/usb/usbip
sudo ./autogen.sh
sudo ./configure
sudo make install -j 8
```

Copy tools libraries location so usbip tools can get them.

```sh
sudo cp libsrc/.libs/libusbip.so.0 /lib/libusbip.so.0
```

Install usb.ids so you have names displayed for usb devices.

```sh
sudo apt-get install hwdata
```

From the root of the repo, copy the image.

```sh
cp arch/x86/boot/bzImage /mnt/c/Users/<user>/usbip-bzImage
```

Create a ```.wslconfig``` file on ```/mnt/c/Users/<user>/``` and add a reference to the created image with the following.

```pwsh
[wsl2]
kernel=c:\\users\\<user>\\usbip-bzImage
```

From an administrator command prompt on Windows, run this command. It will list all the USB devices connected to Windows.

```pwsh
> usbipd wsl list
BUSID  VID:PID    DEVICE                                                        STATE
2-4    04d9:a115  USB Input Device                                              Not attached
2-5    5986:212b  Integrated Camera                                             Not attached
```

Select the bus ID of the device you’d like to attach to WSL and run this command. You’ll be prompted by WSL for a password to run a sudo command.

```pwsh
> usbipd wsl attach --busid 2-5
[sudo] password for user:
```

From an administrator bash on Linux, run this command.

```sh
sudo usbip list --remote=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')
sudo usbip attach --remote=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}') --busid=2-5 
```

At this moment, it can be found that it has appeared and can be tested with ```/dev/video0```

```sh
sudo apt install v4l-utils ffmpeg
```

```sh
Examine device access.
 v4l2-ctl --list-devices
```

```sh
Allow access by using
sudo chmod 777 /dev/video0
```

```sh
ffplay -f video4linux2 -input_format mjpeg -framerate 30 -video_size 640*480 /dev/video0
```

Or you can also use OpenCV for testing.

```py
# import the opencv library
import cv2


# define a video capture object
camera = cv2.VideoCapture(0, cv2.CAP_V4L2)
camera.set(cv2.CAP_PROP_FOURCC, cv2.VideoWriter_fourcc('M', 'J', 'P', 'G'))

while(True):
	
	# Capture the video frame
	# by frame
	ret, frame = camera.read()

	# Display the resulting frame
	cv2.imshow('frame', frame)
	
	# the 'q' button is set as the
	# quitting button you may use any
	# desired button of your choice
	if cv2.waitKey(1) & 0xFF == ord('q'):
		break

# After the loop release the cap object
camera.release()
# Destroy all the windows
cv2.destroyAllWindows()
```

```cpp
#include <opencv2/opencv.hpp>
#include <iostream>

using namespace std;
using namespace cv;

int main()
{
    VideoCapture cap;
    cap.open(0, CAP_V4L2);
    cap.set(CAP_PROP_FOURCC, VideoWriter::fourcc('M', 'J', 'P', 'G'));

    if( !cap.isOpened() ) {

	    cerr << "couldn't open capture."<<endl;
	    return -1;
    }

    Mat frame;
    namedWindow("frame", WINDOW_AUTOSIZE);

    while(true)
    {
       cap >> frame;
       
       imshow("frame", frame);

        if(waitKey(1) == 27) break; // Wait for 'esc' key press to exit
    }

    cap.release();

    return 0;
}
```

```sh
g++ camera.cpp -o camera `pkg-config --cflags --libs opencv4`
```
