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

# pacman-key -r 962DDE58
# pacman-key --lsign-key 962DDE58

Refresh the package list and upgrade the system if necessary:

# pacman -Syyu

Finally, install the bundle with

# pacman -S infinality-bundle
```

### Install & Configure VNC, SSH & Encrypted VNC through SSH Tunnels

First let's take care of SSH as that gives the most utility for remote configuration anyways, and being the security nutjob that I am anyhow, I'd like to make sure everything is pretty secure from the get-go.
Install openssh package.
Before we configure anything about the ssh daemon, let's make a backup of the default configuration:
```
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.default
```
Cool, now the default is there and safe, and we can always cp back to it if things go awry. First, let's configure which users get access to the system while ssh'ing into it. It is **NOT AT ALL** recommended to give root access to ssh tunellers, only allow the user account of the admin of the system(you) user access. You can still access the root with sudo when doing remote admin tasks.
Inside `/etc/ssh/sshd_config` use this line:
```
AllowUsers user1 user2
```
And change it to whatever admin user accounts should have access. **Alternatively** you could also enable access with user groups, and since in the previous articles it was suggested that the useradd of the account for the system should be included in the `wheel` group, *the standard admin user group for arch*, it might be easier and more convenient to ignore the above line and give access to the wheel user group like below:
```
AllowGroups wheel
```
Then I **highly** recommend to disable root login with this line:
```
PermitRootLogin no
```
If you'd like to print out a banner to welcome someone into the system upon tunneling in, use the `Banner` option. Here it takes the text from the `/etc/issue` file, which is the standard linux way to display system information, which can be modified to suit whatever itch needed:
```
Banner /etc/issue
```
If you'd prefer to generate your own keys and have them be required to connect to the server, here is where you show the file that stores the key. The file shown is the standard one, but literally any name of file can be used so long it holds a properly formatted SSH key
```
HostKey /etc/ssh/ssh_host_rsa_key

```
If you plan on port forwarding on your router to allow SSH connections from the outside internet, it's a good idea to change the port it listens to to from the standard port 22, to a random higher one that's smaller than the maximum of  65535. Remember, the router on your network needs to port-forward to this port and the IP-Address of the computer for this to work, so look up how to port-forward online for your particular router make or model.
```
Port 29987
```
Since we are exposing our computers to the world-wide-web, it is important to be as secure as possible. Although SSH encryption and the protocol itself are quite safe, there are defaults that should be changed to reduce the likelyhood of hacking significantly. One such configuration are password logins. Because if they are enabled, even with a safe key in place, the server will default to password login if the key authentication fails, which opens the system up to fairly easy brute-force hacks. The way we prevent this is by using a config to force public key authentication, which means the only way to brute force hack into the system, is to crack the SSH key which by default is a 2048-bit key. this is essentially impossible to crack without **insane** luck, or an ability to hack the client that connects to it.
```
PasswordAuthentication no
ChallengeResponseAuthentication no
```

With this feature on, it can be a real PITA to pass SSH keys between the clients you'll use to connect to this server and the server itself, so a trick I use myself is to save a version of `/etc/sshd_config` called `/etc/sshd_config.insecure` that doesn't have the above configs that enforce public key authentication so I can use the ssh client's 



### Other packages that are straight forward either through AUR or Arch Official
- ranger
- gpicviewer
- vimperator (firefox addon)
- zathura







References
- [BBS on Arch for Fixed Infinality Install](https://bbs.archlinux.org/viewtopic.php?id=162098)
- [Digital Ocean Guide For VNC with SSH Tunnel](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-vnc-remote-access-for-the-gnome-desktop-on-centos-7)
- [Arch Wiki: TigerVNC](https://wiki.archlinux.org/index.php/TigerVNC)
- [Arch Wiki: SSH](https://wiki.archlinux.org/index.php/Secure_Shell)
