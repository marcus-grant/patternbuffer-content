# Configure & Install Arch with the i3 Window Manager
** TODO: Add Intro and other stuff here **

# xinit Install
`xinit` is the software stack that handles how the [Xorg](https://wiki.archlinux.org/index.php/xorg) server initializes. Inorder to install any desktop environment or window manager which includes i3, or the more common gnome, MATE, unity, KDE, XFCE, etc. The primary file involved here is the `~/.xinitrc` file, which is what defines how desktop environments and window managers initialize when starting the X server. If these terms are unfamiliar to you, I highly recommend you reference the
[Xorg](https://wiki.archlinux.org/index.php/xorg) article on the Arch Wiki, which goes into tremendous detail on the subject.

First you are going to need to install the `xorg-init` package using pacman with this line below.
```
sudo pacman -S xorg-xinit
```
From now on, I'm going to assume you know about the [pacman](https://wiki.archlinux.org/index.php/pacman) package manager which is the default package manager for Arch Linux, and that you know about and have setup [sudo](https://wiki.archlinux.org/index.php/sudo). I will mention packages that require install like so `package_name` where you will have to install the package with the name *package_name* above in that formatting.

Anyways, you will need to install the package `xorg-xinit` before you proceed anymore, so do so. With that done it's time to move on to its containing Xorg package. So install the `xorg-server` package. And here is one of the reasons I'm writing this guide. Although the Arch wiki is generally terrific, it misses a crucial detail here in the install of `xorg` which is that you will be presented with two options for the input driver: `xf86-input-evdev` and `xf86-input-libinput`. Through
some searching I've found out that it is supposedly best to use `libinput` because that is where the future of linux xserver installs are going, so chose that. Also supposedly `libinput` is better for newer hardware, and the other option is more for legacy hardware installs. And Arch maintainers, **please** be sure to include details like this or at least a link to the relevant pages to inform yourself on these topics, because it makes Arch particularly unfriendly to newcomers and
it wastes unecessary time to people who just want to get up and running.

Next we will want the additional `xorg` packages that aren't included in the base packager: `xorg-server-utils` and `xorg-apps`. You will want to include all dependencies listed in the install, so hit enter.

Now, we need to install our graphics drivers and associated OpenGL library. Refering back to the XOrg wiki, look up the table that shows what the driver and library packages go with what graphics hardware manufacturer. I'm installing this only on the integrated Intel GPU. So first verify that you have `xorg` drivers and modules for your GPU with `lspci | grep -e VGA -e 3D`. You should be seeing your particular GPU in there if your `xorg` install worked, and possibly even multiple graphics
cards if you have an integrated GPU ontop of a seperate Graphics card, or even an SLI or Crossfire system. The command `pacman -Ss xf86-video` searches the Arch Repository for all available drivers to you. Use the previously mentioned table in the Arch wiki to find out what driver to use. My intel integrated GPU warrants the `xf86-video-intel` driver, so if that's your setup, enter `sudo pacman -S xf86-video-intel`, and if you have something else pick from the list of available drivers
previously printed on the terminal and do the same but with that driver. Then install the appropriate OpenGL library for your card, in my case `mesa-libgl`. This will be different if you plan on using nVidia or AMD.

Now we are ready to install the i3 window manager. Install `i3-wm`, `i3status`, and `i3lock`. The extras aren't necessary, but they are really lightweight, and unless you are installing on a very old computer or these new single board computers should you ever consider omiting them.
 
  **below add a link to xinit**
   
   Next, we need to edit the `~/.xinitrc` file so that the xserver knows to run with our newly installed window manager. If you want to know more about what xinit is, go to the wiki [here](https://wiki.archlinux.org/index.php/Xinit). To make this happen, simply add the line `exec i3` to the file like so: 
   ```
   touch ~/.xinitrc
   echo "exec i3" >> ~/.xinitrc
   ```

   Now we need to install an additional display manager, `lxdm`. `lxdm` is a very lightweight display manager, and we will be using it to supplement i3 for non-terminal application windows. After installing the package, enable it using the `systemctl` command. Another pet-peeve I have, is that the Arch documentation refuses to repeat **any** standard statement previously covered by the wiki. Here is a statement concerned with the system daemon which handles several programs, services and
   daemons that run automatically with the system. If you want more details look at this [wiki page](https://wiki.archlinux.org/index.php/Enable), but basically, you simply type `systemctl ACTION SERVICE_NAME` where *ACTION* is what you want the system to do with the service, be it `start`, `stop`, `enable`, `disable`, or any other action you can find in the wiki, and *SERVICE_NAME* is the service you want the system to perform the action to. In this case we want to *enable* the systemd
   service, `lxdm` that we just installed with the command `systemctl enable lxdm.service`. See Arch, it really isn't hard or particularly time consuming to explicitly write out these commands on a web-site, provide the link to the wiki as an extra if people wish to learn more.

   Now if we want to configure `lxdm` we can do that with the base configuration of `lxdm`. All config files are located in the `/etc/lxdm/` directory. The main config file is `lxdm.conf` which has commented documentation, and there is also the `Xsession` file which should never be touched. We won't be doing that in this article, but I will likely be writting another one to customize i3 and lxdm.

   Next we should reboot to check that the install of everything and the base configurations are working. Do this with `sudo shutdown -r now`.

   Hopefully everything is running OK and you are presented with this screen **ADD PIC**. If not, you should still be presented with the command line and you can go through the instructions again.

   On the bottom left corner, there should now exist the `i3 manager debug log` option, select that and login with your username. You should see a dialog box telling you that you haven't configure i3 yet, hit enter to go through a guided configuration of i3. All the options are fairly easy to understand, if not go to the [i3 documentation](https://i3wm.org/docs/userguide.html) for a run-down of all the options.

   **OOOPS!!!** We didn't install a terminal emulator for `xserver` to use, so when we login all you will likely see is a light blue blank screen. We also can't easily exit out of this screen without some trickery. Luckily I ran into this mistake already and found the way out! Thanks to [this](http://unix.stackexchange.com/questions/251456/cant-exit-i3-because-no-sensible-terminal-emulator-is-installed) stack post, I know now that I hadn't installed a terminal emulator, and that I can go into
   a terminal view by pressing `CTRL+ALT+F(1-7)` where 7 seems to be the common defualt. Now you will have to login as before, and install `xterm` to get us running inside i3. If you want another emulator, feel free to install it as you please, but for now I'm sticking to the simple and known xterm, and will change later when it comes time to configure i3 to my liking. I know there's a better way to reboot into the x environment, but I want to check that everything starts up properly from a
   reboot so you should restart the system as I did with the power button. If anyone wants to chime in on the *proper* way to do this, feel free in the comments.

   OK, back at the graphical login screen, let's try this again. Hmmm... Still on a blank screen, so I searched for an answer, and duh... that is the default for i3 and all we need to do is hit the previously defined `MOD` button, `SUPER` in my case, but we will refer to it as `MOD` from now on. Anyways, hit `MOD+Enter` together. Refer to this quick and dirty [cheatsheet](http://i3wm.org/docs/refcard.html) as we go through this guide. And finally, some real results! There should now be a
   blank terminal on display.

   Alriiiight, there should now be a functional, if not kind of shitty i3 environment setup that you can configure to your liking, or read **this** guide to configure the way that I prefer to do it. Even if you don't like my configuration, you should at least take a look to learn a bit about how you actually configure it.

   As always, comment if you have questions, or criticisms (constructive only as always). And live long and prosper llap.




# References
- [Arch Wiki for xinit](https://wiki.archlinux.org/index.php/Xinit)
- Add references later, and use reference links in markdown so you only need one link for each
- [i3 Cheat Sheet](http://i3wm.org/docs/refcard.html)
- [Arch Wiki: Xinit](http://i3wm.org/docs/refcard.html)
- [Arch Wiki: Display Managers](https://wiki.archlinux.org/index.php/Display_manager#Session_list)
- 
- [i3 Official User Guide](https://i3wm.org/docs/userguide.html)
- [Code Cast i3wm Youtube Video](https://www.youtube.com/watch?v=j1I63wGcvU4)
- 
