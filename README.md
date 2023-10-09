# Norns-Fates-on-RPi-Cheat-Sheet  2023
I recently spent a bit of time trying to figure out how to install Norns on custom hardware and thought I'd share all the file names, edits, and links in one convienent cheat sheet. 
Most of the info here was found at:  https://llllllll.co/t/norns-on-raspberry-pi/14148    (The thread is long!  Check here for how to get hdmi or full headless working.   A lot of good stuff, but there was a big change in the norns code in 2022, so some of the info/steps posted prior to that may no longer be relavant or work.)

# Get Norns (or Fates) .img
Choose the appropriate precompiled .img and burn it  to SD.  

(Norns vs Norns shield vs Fates?  Essentially they are all the same with minor changes for minor hardware revisions.   For installing on Rpi w/ custom hardware, other than choosing the Pi3 or Pi4 .img, it shouldn't matter much which version you choose.  Norns is the "official" version and is maintained by Monome, while Fates is a (currently up to date) fork. Both do the same thing.  Fates seems a little easier to install- Norns may involve a few extra steps as noted with **.)

Afterward, add your necessary screen and audio codec overlays to /boot/overlays if they arent already present and enable them in /boot/config.txt.  If you are a zynthian user, you can copy these lines directly from the zynthian config.txt.   (Some screens and codec may require further setup/ drivers installed)
Boot it up.   If you have a screen attached and it is enabled with the proper overlay, it should  display  some type of “jack fail” message.  SSH in and expand the disk as explained in the norns/fates install instructions.

# Change/set up jack service:
norns will not  start if jack doesnt connect propery

*Change the alsa device to your device (as listed aplay and arecord) in these files: 
 
       /etc/systemd/system/norns-jack.service
 
      ./norns-image/config/norns-jack.service    (not necessary)
 
      ./fates/install/norns/files/norns-jack.service   (not necessary)

**Norns: if your codec does not have amixer controls, disable the amixer call in:  
 
           /etc/rc.local

  (Add a # before the amixer line)

# Build a custom encoder overlay using one of these as a template  (for GPIO-wired encoders only!!): 
*replace the GPIO pin numbers with the your GPIOs
https://github.com/okyeron/fates/tree/master/overlays

*Build the overlay file from the .dts  with dtc  and copy it to /boot/overlays
*Enable it in  
     /boot/config.txt

As shown here:  https://github.com/AkiyukiOkayasu/RaspberryPi_I2S_Master

# Fix screen 
norns expects a 128x64 display.  if you are using something larger, Norns will appear tiny! so we need to change the resolution and scale:

    ./norns/matron/src/hardware/screen.c

*Change resolution in this line (around line 86) to your screen’s resolution:

     surface = cairo_image_surface_create(CAIRO_FORMAT_ARGB32, 120, 60);

*Then…(quoting from lines forum)  “Add this line after the function block with your scale factor
where 5 = 640/128
(x = device res width/original cairo surface width)
and 7.5 = 480/64
(x = device res height/original cairo surface height)”

     cairo_scale(cr, 3.75,5);
   

the whole section should look like this (for waveshare 3.5)

    void screen_init(void) {
    surface = cairo_image_surface_create(CAIRO_FORMAT_ARGB32, 480, 320;
    cr = cr_primary = cairo_create(surface);

    status = FT_Init_FreeType(&value);
    if (status != 0) {
        fprintf(stderr, "ERROR (screen) freetype init\n");
        return;
    }
    cairo_scale(cr, 3.75,5);
    
    strcpy(font_path[0], "04B_03__.TTF");
    strcpy(font_path[1], "liquid.ttf");
    
!remember, the original screen is 2x1, so if you want to keep the proportions the same, use the same number for the scaler.  

**Norns: (if your screen is showing a terminal window -(pi3))

           ./norns/matronrc.lua      
           
    *change fb0 to fb1 



*After changing screen.c - recompile Norns with waf
 
 https://monome.org/docs/norns/compiling/

  cd norns

 ./waf configure

 ./waf -j 1

# Heads up -  
You may (very likely) have to redo the jack and screen configurations steps again after Updating Norns!

# Enable uart midi:
*Follow the steps here:

https://github.com/okyeron/shieldXL

# Remove blinking cursor:
after getting the screen working (for Norns) i still had a command line cursor present in the background. 
to remove:

https://techoverflow.net/2021/10/19/how-to-hide-all-boot-text-blinking-cursor-on-raspberry-pi/

     /boot/cmdline.txt
 
*change :  console=tty1 to console=tty3

*Add at the end of the line:  

     loglevel=3 quiet logo.nologo vt.global_cursor_default=0



# Add Orac/Sidekick
Install orac :   https://llllllll.co/t/orac-sidekick-pure-data-and-sc-for-norns/26198
(I reccommend doing this even if you arent interested in Orac.  Norns "seems" to run better, smoother afterward.   ?)

If your sidekick display is “invisible”:

*create a restart script  for sidekick:

     sudo nano sidekickrestart.sh

*Include these 2 lines:

     #!/bin/bash

     sudo systemctl restart sidekick

*open crontab 

     crontab -e  

*add a line to the bottom:

     @reboot sh sidekickrestart.sh

To do:   Resize orac/sidekick to fit screen resolution... Currently the display is 128x64.   Change it by editing the scale in nuilite code (NuiDevice.cpp)  and recompiling. 
 


