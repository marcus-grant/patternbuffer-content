### Infinality & Font Configurations

Add these to /etc/pacman.conf

```

[infinality-bundle]
Server = http://bohoomil.com/repo/$arch

If you want to access multilib libraries in x86_64 architecture as well, add also this:

[infinality-bundle-multilib]
Server = http://bohoomil.com/repo/multilib/$arch

Additionaly, if you want to use a comprehensive collection of free fonts from the infinality-bundle-fonts repository, append

[infinality-bundle-fonts]
Server = http://bohoomil.com/repo/fonts
```

Next, import and sign the key:

```

### Binding Specific Programs to Workspaces and Terminating GUIs!

With workspaces defined the way we want them, it might be nice if certain applications only opened up in the workspace name that is associated with it. For example, I use rhythmbox as my go to music player, when I feel like working with a GUI. So how is that done? Open up a free terminal window, and I'll show walk you through it. Every window in i3 has a class name for each application, and if we find out the name of that class, we can make a config option that sticks programs of a certain class name to a specific workspace. 

In the free terminal window, install if you haven't alread a music player, or another program you will tie to a workspace. Then open it with the terminal. Now type in `xprop` in the command line and your cursor will change into a corshair type icon. This will allow you to click on an open window and in the window you typed `xprop` you will see some output including ascii versions of that application's icon and some properties that the xserver uses to keep track of open applications. Within that list of properties there will be a line with `WM_CLASS(STRING)` that will have the program's class name. Copy it or remember it, and if there's more than one class name, just go with the first one. Open the i3 config again, and create a new section after the `#move focused container to workspace` section and create a new one for rssigning apps to workspaces, and then use the `assign` config option to define where the program should open by default like so:
```
# workspace application assignments
assign [class="Rhythmbox"] $workspace4
```

This tells i3 that whenever Rhythmbox opens, that it should be opening in with the workspace of name $workspace4, which in the previous section was given the name "Media". Save the config file, log out then log back in and try to launch your program with the `dmenu`. Where does it open? Why on the new workspace of course, awesome!

This isn't the only customization that specific workspaces can receive. It's also possible to give workspaces icons using the font-awesome library of awesome icons that are font compatible. When we're done installing font-awesome, it will simply be a matter of copying and pasting the particular icon wanted into the config file, then restarting i3 to make the workspace display those fonts from then on for that particular workspace.

First, we'll need the font-awesome font icons on our system. I like to keep them in a set place since they can be used in many different ways, so I would make a folder in your home directory if there isn't one there already:
```
mkdir ~/.fonts
```
Then copy [this](https://github.com/FortAwesome/Font-Awesome.git) to the font-awesome github page. And use the `git clone` command inside the newly defined `~/.fonts/` folder to download it.
```
cd ~/.fonts/
git clone https://github.com/FortAwesome/Font-Awesome.git 
```

There should now be a folder inside `.fonts` that holds the font-awesome font icons library. We'll be needing only one file in there inorder to make font-awesome icons work with i3, and that is a `*.tff` file stored within `Font-Awesome/fonts. The rest can be deleted with the `rm -rvf` command below, otherwise if you'd like to use font-awesome for something else leave it by ignoring that command
```
cp ~/.fonts/Font-Awesome/fonts/*.tff ~/.fonts/
rm -rvf ~/.fonts/Font-Awesome/
```

Finally, all you need to do know is go the font-awesome [cheat-sheet](http://fontawesome.io/cheatsheet/), find the icon you'd like to use for the given workspace, copy it, then paste it into the workspace name definition variables defined previously like so:
```
set $workspace4 "Media "
```
Now this could easily be shown as an unknown unicode character, and there could be several reasons for this, but it doesn't really matter for the scope of this article. If you save the config and restart i3 everything should look as expected if you open the workspace with the icon defined.


Being such a terminal focused window manager, it might be nice if i3 could somehow 


### Other packages that are straight forward either through AUR or Arch Official
- ranger
- gpicviewer
- vimperator (firefox addon)
- zathura



### TODO
- Add section for feh and exec_always and how to use it to make background image displays
- Join stuff in byword for this article on key binds
- Add section on workspace naming and use of i3 config file variables
- Add section on configuring xfce4-terminal with color scheme, also link to bashrc article




References
- [BBS on Arch for Fixed Infinality Install](https://bbs.archlinux.org/viewtopic.php?id=162098)
- [Digital Ocean Guide For VNC with SSH Tunnel](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-vnc-remote-access-for-the-gnome-desktop-on-centos-7)
- [Arch Wiki: TigerVNC](https://wiki.archlinux.org/index.php/TigerVNC)
- [Arch Wiki: SSH](https://wiki.archlinux.org/index.php/Secure_Shell)
- [5](
