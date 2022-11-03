# vaha-m5-hack

Welcome to my repository where I document the reverse-engineering that I have done of the VAHA M5 mirror. Throught this page, you will find my teardown of the device, and the guide to transform it into a giant Android tablet.

## Internals

Here is a split up of the internals of the mirror:

### Camera

Interface: USB, this is the longer USB cable found inside.

Recognized as [0bda:3035](https://linux-hardware.org/?id=usb:0bda-3035) "Realtek Semiconductor Corp. USB Camera" by Linux.

### Audio system

Interface: USB, while this is a USB signal, it is connected to the Single Board Computer using a JST connector. I could remove that connector, and replace it with a standard USB-A 2.0 connector.

Recognized as 01c5:2710 "BATSound MicArray LHSJ103" by Linux.

It includes the microphone array which is visible as 4 black dots at the top of the mirror, and the amplifier for the four speakers.

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

* Remove the existing SBC.
* Use the "2AV-1VGA-1HDMI-TTL-50PIN-LVDS-ACC" board from AliExpress to generate the LVDS signal from an HDMI signal.
* Put a new "open" SBC running Android.

### The LVDS Driver

The "2AV-1VGA-1HDMI-TTL-50PIN-LVDS-ACC" board is the only one that I could find that is pin-for-pin compatible with the original LVDS cable provided in the mirror. From my research, each LCD panel vendor tend to have their own connector on the LCD controller board. It is therefor interesting to reuse the existing cable which has the correct pin-out on the LCD controller side.

To select the 1920x1080-D8 mode on the HDMI-LVDS board, you need to put jumper-caps on positions A, 1, and 7. Those jumper-caps have a 2mm pitch instead of the usual 2.54mm.

Thanks to this LVDS driver, we can now use any SBC that outputs an HDMI signal. You might also just opt for an SBC which can send an LVDS signal instead of buying this converter, but those SBCs are not so easy to come by for hobbyists.

### The new SBC

We can be very flexible in terms of SBC since we now simply need USB, HDMI, and some GPIOs. It would also be easier if it can be fed with 12V like the original one.

For this project, I opted for the Khadas VIM4, since it can be easily setup using Android through their OOWOW setup tool. In addition, they offer detailed diagrams with the wiring of the board.

In a nutshell:

* VDDAO_3V3 (pin 27) will send current to the amplifier and optocoupler to drive 5V to backlight driver and LVDS converter. That pin only turns on when the CPU is running. That way, the backlight, amplifier, and LVDS converter only run when the CPU is on.
* The USB-C connector will be made available on the backplate of the mirror.
* The camera will be connected to the internal USB 2 headers (next to GPIOs), while the VIM4's USB 3 port will be connected to a new hub connecting touchscreen, audio, and an external USB 3 connector. That way, the internal USB 2 hub of the VIM4 is used only for the camera and cannot be saturated by another device.
* The PWD_HOLD_EXT pin on the header is not the same as the pads on the backside for an external button. We need the pad on the backside to be connected to the yellow wire coming from the power button. When the button is pressed, the wire will be set to ground level, and a "Power" signal will be processed by the SBC.
