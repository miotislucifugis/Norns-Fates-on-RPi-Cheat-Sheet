# Norns-Fates-on-RPi-Cheat-Sheet  2023
I recently spent a bit of time trying to figure out how to install Norns on an Rpi with custom hardware and thought I'd share all the steps: file names, locations, edits, and links in one convienent cheat sheet.  Its not really a tutorial per se; but if you know some basic command line navigation, it should be pretty simple.   (hint: "sudo nano filename" to edit files,  "cd directory name" to change directories)     

Tested with both norns and fates on a Pi3, but it should be basically the same formula for pi4. Im using a waveshare 3.5A display,  a diy codec (pcm5102 Dac + pmc1802 Adc (in "master" mode) + external 12.288 mhz crystal for the system clock - using this dtoverlay: https://github.com/AkiyukiOkayasu/RaspberryPi_I2S_Slave), and encoders/switch wired directly to the GPIO. 

Most of the info here was found at:  https://llllllll.co/t/norns-on-raspberry-pi/14148    (The thread is long!  Check here for how to get hdmi or full headless w/ vnc working.   A lot of good stuff, but there was a big change in the norns code in 2022, so some of the info/steps posted prior to that may no longer be relavant or work.)

# Get Norns (or Fates) .img
*Choose the appropriate precompiled .img and burn it to SD.  

     https://github.com/monome/norns-image/releases
     https://github.com/okyeron/fates/blob/master/install/norns/Norns_disk_image_install.md


(Norns vs Norns shield vs Fates?  Essentially they are all the same with minor changes for minor hardware revisions.   For installing on Rpi w/ custom hardware, other than choosing the Pi3 or Pi4 .img, it shouldn't matter much which version you choose.  Norns is the "official" version and is maintained by Monome, while Fates is a (currently up to date) fork. Both do the same thing, Fates seems a little easier to install- Norns may involve a few extra steps as noted with **.)  If using norns, perhaps use an .img made for the "shield" version (which use pi3) as opposed to the .img for the "original" norns (which used a compute module) to avoid any (extra) compatability problems. 

*Afterward, add your necessary screen and audio codec overlays to /boot/overlays if they arent already present and enable them in /boot/config.txt.  If you are installing on zynthian hardware, you can copy these lines directly from the zynthian config.txt.   (Some screens and codec may require further setup/ drivers installed)

*Boot it up.   If you have a screen attached and it is enabled with the proper overlay, it should  display  some type of “jack fail” message.  Congratulations. This is what we want to see.

*SSH in and expand the disk as explained in the norns/fates install instructions.


# Build a custom encoder overlay using one of these as a template  (for GPIO-wired encoders only!!): 
*Create a new .dts file:

     sudo nano your__overlay.dts

*copy/paste one of the encoder overlay templates from the following link inside and replace the GPIO pin numbers with the your encoder GPIOs
https://github.com/okyeron/fates/tree/master/overlays.    Save. Exit 

*Build the overlay file from the .dts  with dtc and copy it to /boot/overlays

      dtc -@ -H epapr -O dtb -o your_overlay.dtbo -Wno-unit_address_vs_reg your_overlay.dts
      
      sudo cp your_overlay.dtbo /boot/overlays

*Enable it in  
     
     /boot/config.txt


# Change/set up jack service:
norns will not start if jack doesnt connect, so we need to configure jack service to use our codec.

*check to see if you codec is recognized and available:
         
         aplay -l
         arecord -l

*Change the alsa device to your device in these files: 
 
       /etc/systemd/system/norns-jack.service
 
      ./norns-image/config/norns-jack.service    (not necessary)
 
      ./fates/install/norns/files/norns-jack.service   (not necessary)

Just change the bit about the device (-d hw:sndrpimonome) to:

      -d hw:your_card_name
      
My codec appears as the same card but has different device numbers for the playback and capture, so I need to state specifically the capture (-C) and playback (-P)  devices.   For my custom codec, the line looks like this :

      ExecStart=/usr/bin/jackd -R -P 95 -d alsa -Chw:GenericStereoAu,1 -Phw:GenericStereoAu,0 -r 48000 -n 3 -p 128 -S -s


**Norns: if your codec does not have amixer controls, disable the amixer call in:  
 
           /etc/rc.local

  (Add a # before the amixer line)



# Fix screen 
This works with a cheap Waveshare 3.5-style SPI touchscreen display.  I believe that getting HDMI screens to work involves more steps.  Check out the lines forum (link above) for more details.
Norns expects a 128x64 display.  if you are using something larger, Norns will appear tiny! so we need to change the resolution and scale:

    ./norns/matron/src/hardware/screen.c

*Change resolution in this line (around line 86) to your screen’s resolution:

     surface = cairo_image_surface_create(CAIRO_FORMAT_ARGB32, 120, 60);

*Then…(quoting from lines forum)  “Add this line after the function block with your scale factor
where 5 = 640/128
(x = device res width/original cairo surface width)
and 7.5 = 480/64
(x = device res height/original cairo surface height)”    ....so thats (cr,5, 7.5) for a 640x480 display,   (cr,3.75, 5) for a 480x320  ... etc

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



*After editing screen.c (and/or ./norns/matronrc.lua)  - recompile Norns with waf
 
 https://monome.org/docs/norns/compiling/

  cd norns

 ./waf configure

 ./waf -j 1


*reboot and see if you are greeted with the sparkle screen

*if so, you can set up wifi and upgrade norns via the norns "settings" menu.... however....

# Heads up -  
You may (very likely) have to redo the jack and screen configurations steps again after Updating Norns!




# Enable uart midi:
"real" norns only have Usb midi. To enable uart midi, follow the steps here (mid-page) :

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

If your sidekick display is “invisible”? - launching sidekick seems to freeze your norns?   Sidekick is actually there but I think it is "talking" to the wrong screen. I think it is a bug, but here is a hacky work-around- just restart sidekick.   here is how to make this happen automatically evrytime you boot:

*create a restart script  for sidekick:

     sudo nano sidekickrestart.sh

*Include these 2 lines:

     #!/bin/bash

     sudo systemctl restart sidekick

*make the script exicutable:

      sudo chmod u+x sidekickrestart.sh


*open crontab 

     crontab -e  

*add a line to the bottom:

     @reboot sh sidekickrestart.sh

Now you can see the sidekick menu and launch Orac!


To do:   Resize orac/sidekick to fit screen resolution... Currently the display is 128x64.   Change it by editing the scale in nuilite code (NuiDevice.cpp)  and recompiling.

Currently, Im not getting midi in to orac, but I havent messed with it enough, could just be user error.
 


