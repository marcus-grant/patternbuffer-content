# HowTo Setup Secure Remote Tunneling & Configuration with SSH, VNC, & Google Authenticator

I often find myself wishing I could configure, or simply perform mundane tasks on my server from the convenient location of anywhere. So I learned how to configure SSH & VNC servers and clients on my various systems that would enable me to do so. There is a big caveat of enabling access to your system from anywhere, it also opens up access for **anyone** anywhere. Without encrypted public key access being enforced on your system this convenience very quickly becomes a liability that endangers all your personal information to would-be hackers. Without proper security precautions, all that's required to hack into the server, where the `openssh` or VNC server of your choice is installed can fairly easily be hacked into using fairly rudimentary [brute-force attacks](https://en.wikipedia.org/wiki/Brute-force_attack). Admittedly the lengths I go through to secure my devices and network can be a bit paranoid, however, I'll be pointing out the **absolute minimum** steps you should take should you decide expose your server to the internet, on-top of the more paranoid approach of using [two-factor authentication](https://en.wikipedia.org/wiki/Multi-factor_authentication) using Google Authentication, which adds another layer of security on-top of the already *nearly* impossible to crack SSH public key. I'll also be going over some tips and tricks to make generating the public key on your client and passing it onto the server easier, on-top of all the installs and configurations of the server. So enjoy.



### Disclaimer

**I DO NOT** assume any responsibility in any missteps you may take in configuring your SSH or VNC server and exposing them to the world-*wild*-web. If your expose your server to the internet, be sure that you understand the things I am saying before even considering this guide, or at least ignore the parts that are specific to exposing your server to the internet, and stick to the parts that leave your VNC or SSH server accessible only on your local network, because at least that way the only way to hack your system through SSH or VNC would be actually get physically connected to your LAN either by hacking your wifi, or physically breaking in and connecting to your network, but then you probably have even more pressing concerns than your server's security.


### Installing & Configuring the SSH Server

Alright, sorry for the long disclaimer, but I want to hammer home how important it is to get this right if you plan to seriously implement these configurations. Let's get started by configure the SSH server by installing the `openssh` package. I'm demonstrating this for arch systems, but you can fairly easily adapt this guide to any unix-based operating system if you're familiar with how to administrate those systems. Once, the ssh-server is installed, the ssh server configuration should now be stored in `/etc/ssh/sshd_config`. It's good practice before you start playing with any configuration files to make a backup of the default version of the file that gets installed, and the way I always do this is to backup the original by appending the file suffix `*.default` at the end, then modifying the file that was just copied. That way, all I have to do to revert to the default, should anything go wrong, is to copy the default to the regular config file with the regular name like so:
```
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.default
```
And if you need to revert the config back to default, just do this:
```
sudo cp /etc/ssh/sshd_config.default /etc/ssh/sshd_config
```

Cool, now that the default is safely copied, let's configure the stuff inside. The default file is somewhat well-commented with the hash-tag symbol, most of the steps I will give you here just involve modifying the line slightly, or just un-commenting that particular line. If the line isn't in there, like for this step, just add it at the bottom of the file. First, let's configure what system users can access. Do this by finding the line below, and put your particular users in there. If this is truly a multi-user system, like a family computer, only allow the admin user accounts access, which is probably only going to be your user account. **DO NOT** give the root account access, just don't, it doesn't even bare discussion.

```
AllowUsers user1 user2
```

There is also another way of configuring who can access the server through SSH, by allowing a user group access. In Arch this is particularly helpful, because of the standard `wheel` user group, which is there as a specific group to define what users get [admin-like](https://wiki.archlinux.org/index.php/users_and_groups#User_groups) priveleges through sudo. If that's more convenient, alter the config at this line:
```
AllowGroups wheel
```

Uncomment the line below by deleting the hash-tag, to prevent retrying access after a failed attempt after 2 minutes, which is a preventative measure against brute-force attacks.
```
LoginGraceTime no
```

Next, as I just mentioned, root login from SSH is a terrible idea. Consider it completely non-optional from the point of view of this guide, so please prevent it with this:
```
PermitRootLogin no
```

If you'd prefer to handle your own SSH keys, which I'll cover later in the guide, this line in the configuration lets you specify the file that stores it, I would just stick with the default for simplicity's sake. LOOK INTO .ssh/authorized_keys
```
HostKey /etc/ssh/ssh_host_rsa_key
```

Here is an option, that you should only consider if you plan on exposing your system to the internet, if you plan on only making the server accessible from your local network, you can skip this part:
```
Port 12345
```

The above port number is rather predictable, I'm posting it mostly as just a placeholder, you should make up your own random number between 1000 and 65535 without predictable numbers like the one above, or say, "55555", because it makes scanning the port for openings easier. Also, simply changing the listening port on the server isn't enough to give yourself access to it from the internet. You will also have to make it accessible through your router. To do this, look up how to [port-forward](https://portforward.com) on your specific router if you don't know how. 

If you'd like to have a banner message presented to you when you login successfully to the your server through ssh, you can specify a banner message. The one below takes system information through the standard linux system information file `/etc/issue` which you can also customize with specific flags like host-name, or other plugins like component temperatures, or CPU load.
```
Banner /etc/issue
```

Here is another truly non-optional configuration you shouldn't ignore. If you're only letting the server loose on your local network, maybe it's optional because configuring keys for clients can be a pain, *unless you read this guide the whole way through and learn my trick to get over it*, but if this server is getting accessed outside your network **DO NOT** ignore this option:
```
PasswordAuthentication no
ChallengeResponseAuthentication no
```

The reason the above configuration is so important, is because of the way `openssh` is setup. If you try and login to an SSH server without this option, the default behavior when SSH key authentication fails is to default to password authentication. This password is the same one you used to login to your server, and this password after a failed attempt at access through SSH becomes exposed to **ADD REF HERE** fairly rudimentary brute-force attacks, which frankly makes these two options the most important in the guide to get right if security of the server is of concern, and it should be. 


### Running the sshd daemon

Great! Now with these configuration options out of the way, your system should now be quite secure, even if you're port-forwarding through the internet to access it. Now that it's safely configured, it's time to actually get it running through the system daemon. Again, being an Arch-based guide these steps could be different on your particular flavor of UNIX, Linux, or even macOS, so either apply these rules through whatever lens is necessary to adapt it your particular system, or find another guide to help you with this step. *Most* modern Linux distributions should be mostly applicable to these steps however. I do use the same steps on a Raspberry PI running Raspbian which is based on Debian, so that means all Debian derivatives like the ever-popular Ubuntu should also work.

The `sshd.service` service is what actually runs the ssh server in the background, and to enable, start, or stop it, replace `ACTION` below with `start`, `stop`, `enable`, or `disable` to manipulate it.
```
sudo systemctl ACTION sshd.service
```

Before the service ever can run, the `enable` action needs to be run first. And if you ever want to change the configuration file, you will need to restart the service to make the changes take effect. Restart by replacing `ACTION` above with `restart`. `systemctl`, if enabled will always start when the system boots up, so keep that in mind. If you want to have different behavior for this specific service, please read this [wiki] to find out how to do so. Me, I always just leave it running. I know it's secure, and I even try hacking it every once in a while to see if I can get in. If you need the service to stop for whatever reason, you've already configured how to get into the server safely by the end of the guide, so just ssh in, and type `sudo systemctl stop sshd.service`. There are also options for making `sshd` run with a sockets daemon, but I never host servers from home for multiple users to access, and really this option is out of scope for this article, but if you check the [arch ssh wiki] you can see for yourself how to do this. Incase, let's say you're running a cloud service on Amazon or something you'd like to SSH into.



### Generating SSH-Keys

As I mentioned, the only reason you'd ever want to have password authentication on is because it can be a pain to generate an SSH RSA key and passing it between clients and servers. This is especially true for one of my usage cases, which is to administrate my servers through my iPad through an SSH client app. I *could* store the key in the cloud, but I'm amazed this is even an option in the app because that key which is on an internet server gives total access to your server to anyone and I'd rather not risk the fact that someone could hack that cloud server or intercept communications between my iPad and that cloud server. So I came up with a nifty workflow to generate keys, sending the key to the from the client that just generated it and use it.

I do this by using the same `cp` command to copy the `sshd_config` file we just configured to yet another copy, this time with the suffix secure, like so:

```
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.secure
```

Then I copy that config file like below, with the suffix "NOTsecure".

```
sudo cp /etc/ssh/sshd_config.secure /etc/ssh/sshd_config.NOTsecure
```

I now have a copy of the configuration specifically meant to be insecure. This is because I want to be able to temporarily be able to change, create, or send configuration keys easily with a client, and when the new client has the generated key, switch back to the secure configuration, restart the service, and resume normal - secure oprations - but now with access being given to a new client, or with updated security keys. **Do not** forget to change back to the secure configuration when you do this for all the reasons I've already mentioned by copying `sshd_config.secure` to `sshd_config`. If for any reason you feel like you don't understand my method of swapping configuration file versions between the insecure or secure version, or like it's something you could forget about, just don't do it. Also, **switching back to the secured configuration won't take effect till you restart the sshd service**. Always make sure after you've switched configurations, to always try and login to your server without the key and see if it works, because it shouldn't.

Anyways, now that you've learned my trick to temporarily give the server open access to change or add key files, let's learn how to generate keys on your client and then send them. I'll show you how to do it on a unix based computer, and also how to do it on a mobile device using [server-auditor](https://serverauditor.com/).

On any unix based system that you want to give secured SSH access to your server, you will need to have the ssh client program, which will be installed in the same `openssh` package you've alread installed on the server. That package will also have a program 

Server-Auditor is nifty lil' app that's available on iOS, Android, and ChromeOS that turns my iPad & iPhone into truly useful server administration tools. Even with touch keyboards, although I rarely do much without a bluetooth keyboard hooked up. To generate keys for this device, go to the main menu, open the `Keychain` menu, hit the plus icon to add a new key, which will bring up a menu to help you with the process. Enter a key-name, perhaps the hostname of your server, give it a password you can remember, most of the security will come from the size of the public RSA key anyways so don't worry too much about it being a strong password, this is mostly just to make it harder for someone who's logged in on the device from just automatically being able to login to the server. Now select AES-128 or 256, and leave the bit-depth at 2048, or higher, it doesn't really matter, no one has yet to crack even a 800-bit RSA key. The 800-bit key crack was done by a team of white-cap security experts for research purposes and it took 2000 CPU-hours to accomplish, or 12 weeks on one of their top-of-the-line CPUs with the best in class algorithms to reduce the number of guesses to crack the key, and the difficulty to do so goes up exponentially with the key size.

With the key generated and stored on the mobile device with `server-auditor` it's time to add the server as a host with its IP-address, and/or it's publicly accessible URL if you've given it one. This article is already getting pretty huge, so I'm going to have to ask that you look up yourself how you get a public address with a dynamic DNS service, and how to configure your router to update the DNS with it's IP-address as it changes, which happens frequently unless you have a special installation with your ISP. Anyways, assuming you have done so, or are sticking to your local network, go to the main menu of `server-auditor` and go to the `hosts` menu. Add by tapping the `+` icon again to add a new host (your ssh-enabled server). Now specify a name, host address (URL or local IP-address), port number, accessible server username and password, and then shell preferences like colors and font size. Now we're ready to change the server to the insecure config, connect with server-auditor, securely send the key over, and then change the ssh-server to back to its secure config and restart.

Change to the NOTsecure config:
```
sudo cp /etc/ssh/sshd_config.NOTsecure /etc/ssh/sshd_config
```

OK, we can now easily connect without a key. Remember we're in the danger-zone now. We still need to restart the ssh service.
```
sudo systemctl restart sshd.service
```

Now connect to the SSH server with server-auditor. If it worked, you should see the command prompt of your server. If that worked, exit out of the prompt by typing `exit`, and bringing up the terminal menu to go back to the `Keychain` menu. From there, if you tap on your newly generated key within the keychain menu, you should see an option to export to one of your known hosts that you've define. Tap on the one you've just defined. This will bring you to a new view, that lets you verify your key and host settings, and will show you the script that will be executed upon the server receiving the key, and the location they will be stored on the server. They are stored in the default locations we covered before unless you explicitly tell it to go somewhere else. If everything looks good, tap `Export`. If it worked, you should get a prompt that tells you it succeeded or failed. If it worked, you server should now have the public key stored, and you should be ready to switch back to the safe configuration and restart the ssh service.

Change from the NOTsecure config to the secure one:
```
sudo cp /etc/ssh/sshd_config.secure /etc/ssh/sshd_config
```
You are still not secure until you restart the sshd service:
```
sudo systemctl restart sshd_config
```

Now assuming, everything went well, even though the secure configuration is active, you should still be able to connect with the server-auditor app. Find out if it does by reconnecting to your newly defined host within the app. You should also know how to verify if you're back on the secured configuration. I'm not sure how to do that with the app, but by the end of this next section, you will know how, by checking a command line client of SSH if it can connect to the server by renaming the key file. You could also try to SSH into the server with another SSH client without a key stored on the server.


### Paranoid-level security measure

If brute-force attacks scare you, there's several ways to bolster your servers security even further. One of them I will cover here, which is two-factor authentication through Google Authenticator, or other services like Authy which has a linux daemon that uses the texting service [Twilio], which will even text you a 6-digit code that you have 60 seconds to enter into the terminal before you get locked out. This makes brute-force attacks even more difficult, because now, even if you somehow manage to quickly crack the SSH key, you're time-limited to 60 seconds to get it done, and then also crack the 6-digit code within 60 seconds to get into the system. This is nearly impossible. Impossible enough that they'll either give up ur try hacking you in other ways, not related to your SSH configuration, if they really want what's inside your hard drive for whatever reason. Even though Authy on paper seems more convenient, it doesn't seem to get updated very often on their github page, and is generally not nearly as well supported as Google authenticator. And with a Google Authenticator client installed on whatever client you ssh with, it's still quite a convenient way to significantly improve your security. Also, it's Google, they don't screw around or half-ass their software. The other brute-force prevention measure, which I haven't played with yet, but definitely will at some point are programs like fail2ban or sshguard that automatically block IP-addresses that fail too many times to access the server. Finally you could also define a set of IP-Addresses as *white-listed* and only allow connections from those particular IP-addresses. I don't do this because I like to access my servers from mobile devices and their IP-addresses change all the time, but it certainly is an option to consider. 

### Good Information Security Habits

# References
- [Arch Wiki: SSH Keys](https://wiki.archlinux.org/index.php/SSH_keys)
- [Arch Wiki: Secure Shell](https://wiki.archlinux.org/index.php/Secure_Shell)
- [Arch Wiki: Google Authenticator](https://wiki.archlinux.org/index.php/Google_Authenticator)
- [DigitalOcean: How To Setup SSH Keys](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2)
- [DigitalOcean: How To Install & Configure VNC Remote Access](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-vnc-remote-access-for-the-gnome-desktop-on-centos-7)
- [Github: Authy](https://github.com/authy/authy-ssh)
