# Norns-Fates-on-RPi-Cheat-Sheet
A cheat sheet for installing Norns/Fates on Raspberry Pi

Choose a precompiled .img and burn it  to SD.  Add your necessary screen and audio codec overlays to /boot/overlays if they arent already present.
Boot up.   Should display  some type of “jack fail” message.

Change/set up jack service:
Change the alsa device to your device in these files: 
 /etc/systemd/system/norns-jack.service
 ./norns-image/config/norns-jack.service    (not necessary)
./fates/install/norns/files/norns-jack.service   (not necessary)

**Norns: if your codec does not have amixer controls, disable the amixer call:  
 /etc/rc.local
Add a # before the amixer line

Build a custom encoder overlay using one of these as a template:
https://github.com/okyeron/fates/tree/master/overlays
Build the overlay file from the .dts  with dtc  and copy it to /boot/overlays
Enable it in /boot/config.txt
As shown here:  https://github.com/AkiyukiOkayasu/RaspberryPi_I2S_Master

Fix screen 
./norns/matron/src/hardware/screen.c

Change resolution in this line to your screen’s resolution in this line:
surface = cairo_image_surface_create(CAIRO_FORMAT_ARGB32, 480, 320);

Then…“Add this line after the function block with your scale factor
where 5 = 640/128
(x = device res width/original cairo surface width)
and 7.5 = 480/64
(x = device res height/original cairo surface height)”

cairo_scale(cr, 3.75,5);     (for waveshare 3.5)

**Norns: (if your screen is showing a terminal window -(pi3))
 ./norns/matronrc.lua
change fb0 to fb1 

Then recompile Norns with waf
 https://monome.org/docs/norns/compiling/

cd norns
./waf configure
./waf -j 1

Enable uart midi:
Follow the steps here:
https://github.com/okyeron/shieldXL

Remove blinking cursor:
https://techoverflow.net/2021/10/19/how-to-hide-all-boot-text-blinking-cursor-on-raspberry-pi/

 /boot/cmdline.txt
change :  console=tty3  (tty1 to tty3)
Add at the end of the line:   loglevel=3 quiet logo.nologo vt.global_cursor_default=0

Add Orac/Sidekick
Install orac :   https://llllllll.co/t/orac-sidekick-pure-data-and-sc-for-norns/26198

If your sidekick display is “invisible”:
create a restart script  for sidekick:
sudo nano sidekickrestart.sh

With these 2 lines:
#!/bin/bash
sudo systemctl restart sidekick

open crontab (crontab -e)  add a line to the bottom:
@reboot sh sidekickrestart.sh


