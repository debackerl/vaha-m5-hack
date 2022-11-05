# vaha-m5-hack

Welcome to my repository where I document the reverse-engineering that I have done of the VAHA M5 mirror. Through this page, you will find my teardown of the device, and the guide to transform it into a giant Android tablet.

No copyright is infringed since we do not even try to reverse-engineer any of the software found in the device. Actually, the Single Board Computer from the device is fully discarded. This is also in line with the proposals from the European Commission on "Right to Repair". The goal of the project is to extend the lifetime of those devices by opening them up.

**Disclaimer**: By opening a VAHA mirror, you will void its warranty. Also, this experiment has been accomplished without VAHA's support. I own a VAHA mirror. Since it only allows me to run their app in addition to Firefox, Spotify, and Zoom, and I see much more potential as a general Android tablet, I decided to convert it.

## Internals

Accessing the internals is easy. The backplate of the M5 is only held by screws, no glue. Once open, everything is easily serviceable. You can tell that this is a low-volume device. They mostly combined generic off-the-shelves parts, instead of creating an integrated custom-made electronic board. This will make our job much easier, because we can just swap the pieces that we need.

Here is a description of the internals of the mirror:

### Camera

Interface: USB, this is the longer USB cable found inside.

Recognized as [0bda:3035](https://linux-hardware.org/?id=usb:0bda-3035) "Realtek Semiconductor Corp. USB Camera" by Linux.

### Audio system

Interface: USB, while this is a USB signal, it is connected to the Single Board Computer using a JST connector. I could remove that connector, and replace it with a standard USB-A 2.0 connector.

Recognized as 01c5:2710 "BATSound MicArray LHSJ103" by Linux. Surprisingly, this vendor ID 01c5 does not seem to be officially registered, and I cannot find much information about a "BATSound MicArray" device online, or anything with reference "LHSJ103". Even the main chip has some black tape glued on it. I suspect that it might cover an EPROM which would get erased if exposed to UV light, so do not try remove the tape. This might be some niche device that is never offered to end-user as such. Nevertheless, it is supported as a standard USB Audio interface.

This device includes the microphone array which is visible as 4 black dots at the top of the mirror, and the amplifier for the four speakers.

The whole audio system is made of three boards:

1. One is found at the very top and is in charge of the microphones, and handles USB communications.
2. One is the amplifier found at the bottom.
3. The last is the power supply for the amplifier, also found at the bottom. It is feeding 15.5V DC to the amplifier continuously, regardless if the mirror is turned on or off via the front power button.

The amplifier is controlled by one yellow wire going to the SBC. I measured 3V DC going through that wire to turn on the amplifier. _It is not clear if this is an on/off signal, or an analog one to control the amplification level._

### Touch panel

Interface: USB, this is the shorter USB cable found inside.

Recognized as [222a:0001](https://linux-hardware.org/?id=usb:222a-0001) "ILI Technology Corp. Multi-Touch Screen" by Linux.

### LCD Panel

Interface: LVDS 2-ports 8-bits signal + 12V. You might also find references online speaking about "LVDS 1920x1080-D8". There is a great [guide](https://www.darloxcn.com/news/Distinguish-the-definition-of-LVDS-screen-line-and.html) to learn more about LVDS signals.

The LCD panel seems to be something similar to [LC420DUE-FGP2](https://www.panelook.com/LC420DUE-FGP2_LG%20Display_42_LCM_parameter_24854.html), and its connector is likely a JAE FI-R51HL.

### Backlight driver

The backlight driver controls the brightness of the LCD backlight. It is controlled via two wires:

1. N/F pin (black wire going to the SBC): takes 5V to turn on the blacklight, 0V when turned off.
2. Adj pin (red wire going to the SBC): this is an analog signal to control the brightness level. It takes 0V for maximum brightness, and up to 5V for minimum brightness, or any voltage in-between to control for varying levels of brightness. _It is not clear if we can go beyond 5V for lower brightness._

### Main Power Supply Unit (PSU)

The PSU is feeding power to the backlight driver, SBC, and front Power button.

### Single Board Computer (SBC)

The SBC is fed with 12V DC by the main PSU. It is connected to the camera, audio system, and touch panel via USB. It drives the LCD panel using LVDS.

### Front Power Button

The power button is a capacitive sensor that is located just behind the mirror, at the top. It is fed with 5.3V though its black and red wires from the main PSU.

It puts the main PSU into standby through a white wire, and switches the SBC through the yellow wire. When the button is being pressed, that yellow wire is connected to the ground.

## Photos

### Back-side of the device open

![Back-side of the device open](photos/DSC04393b.JPG)

### Bottom view

![Bottom view](photos/DSC04401b.JPG)

### Backlight driver

![Backlight driver](photos/DSC04391b.JPG)

### Power Supply Unit

![Power Supply Unit](photos/DSC04407b.JPG)

### LCD Panel Controller Board

![LCD Panel Controller Board](photos/DSC04403b.JPG)

### Single Board Computer - Front side

![Single Board Computer](photos/DSC04397b.JPG)

### Single Board Computer - Back side

![Single Board Computer](photos/DSC04409b.JPG)

## Android Tablet Mod

The goal of this project is to turn the VAHA M5 mirror into a giant Android tablet.

This in done in several steps:

1. Remove the existing SBC.
2. Use the "2AV-1VGA-1HDMI-TTL-50PIN-LVDS-ACC" board found on AliExpress, to generate the LVDS signal from an HDMI signal. Many ARM processors have an LVDS interface, but most hobbyist SBCs do not expose it.
3. Put a new "open" SBC running Android.
4. Add a USB 3.0 Hub inside the mirror to get extra ports.

### The LVDS Driver

The "2AV-1VGA-1HDMI-TTL-50PIN-LVDS-ACC" board is the only one that I could find that is pin-for-pin compatible with the original LVDS cable provided in the mirror. From my research, each LCD panel vendor tend to have their own connector on the LCD controller board. It is therefor easier to reuse the existing cable which has the correct pin-out on the LCD controller side. With that many wires, you do not want to create your own cable :-)

Some AliExpress shops are lazy and do not explain how to configure the board, and do not always give enough jumper-caps. To select the 1920x1080-D8 mode on the HDMI-LVDS board, you need to put jumper-caps on positions A, 1, and 7. Those jumper-caps have a 2mm pitch instead of the usual 2.54mm. Some also recommend to glue a small heatsink on the main chip when doing 1920x1080. Since AliExpress shops do not always do a great job at doing strong solder joints, and repeated heat cycles is the enemy of those joints, I put a heatsink.

Thanks to this HDMI-LVDS converter, we can now use any SBC that outputs an HDMI signal. You might also just opt for an SBC which can send an LVDS signal instead of buying this converter, but those SBCs are not so easy to come by for hobbyists.

### The new SBC

We can be very flexible in terms of SBC since we now simply need USB, HDMI, and some GPIOs. It would also be easier if it can be fed with 12V like the original one.

For this project, I opted for the Khadas VIM4, since it is easy to install Android through their OOWOW setup tool. In addition, they offer detailed diagrams with the wiring of the board.

If you first power on your VIM4 though the USB-C port to configure it, and it randomly turns off, it might not be getting enough power from the adaptor. Since I didn't purchase their USB adaptor, I relied on the VIN power connector instead.

In a nutshell:

* VDDAO_3V3 (pin 27) will send current to the amplifier and optocoupler to drive 5V to backlight driver and LVDS converter. That pin only turns on when the CPU is running. That way, the backlight, amplifier, and LVDS converter only run when the CPU is on.
* The USB-C connector will be made available on the backplate of the mirror.
* The camera will be connected to the internal USB 2 headers (next to GPIOs), while the VIM4's USB 3 port will be connected to a new hub connecting touchscreen, audio, and an external USB 3 connector. That way, the internal USB 2 hub of the VIM4 is used only for the camera and cannot be saturated by another device.
* The PWD_HOLD_EXT pin on the header is not the same as the pads on the backside for an external button. We need the pad on the backside to be connected to the yellow wire coming from the power button. When the button is pressed, the wire will be set to ground level, and a "Power" signal will be processed by the SBC.

### Android Setup

An advantage of using Khadas' Android distribution, is that it comes with the Google Play Store out of the box. There is no need to use Open GApps, NikGapps, or the likes.

We may need to register our custom device with Google to allow it to interact with official Google Apps and your Google Account. Go to https://www.google.com/android/uncertified/ Note that the "Google Services Framework Android ID" will change each time that you reinstall Android on your device, and you can register at most 100 IDs with your Google Account.
