# Learning & debugging firejail

So, what it firejail?

> Firejail is a SUID program that reduces the risk of security breaches by restricting the running environment of untrusted applications using Linux namespaces and seccomp-bpf. It allows a process and all its descendants to have their own private view of the globally shared kernel resources, such as the network stack, process table, mount table.

It's basically an easy-to-use sandbox of the risk of running everything as root (because it's a setuid binary, even if it drops privileges later). I really like firejail, but there are some things I don't like:

- The website is outdated!? The man page [of the website](https://firejail.wordpress.com/features-3/man-firejail/) are older than the man page of the [Arch Linux package](https://man.archlinux.org/man/firejail.1). Newer features like `--nettrace` are not listed on the website. Things like `--audit` has been moved to the `jailcheck` tool.
- The docs are good, but sometimes not super clear. Some more examples and command line output would be nice. For example: how to disable _all_ [protocols](https://man.archlinux.org/man/firejail.1#protocol=protocol,protocol,protocol)? `protocol`, `protocol none `, `net none`? I still don't know.
- Sometimes it's hard to debug a failing profile

That's why I want to share some of my learnings.

# How to learn about firejail features

1. Create an empty firejail profile called `test`: `touch ~/.config/firejail/test.profile`
2. Run a shell in the `test` sandbox: `firejail --profile=test bash` 
3. Play around in the shell: what works and what not (some things like /boot are blacklisted by default)
4. Modify `test.profile`:  use `blacklist`, `whitelist`, `noblacklist`, `mkdir`, `private-bin`, `net none`, `private`, ...
5. Check out the [man page](https://man.archlinux.org/man/firejail.1#NAME) and find out, how things behave

# Debugging firejail profiles

### join a sandbox

You can start a shell in the sandbox of a running application. You can use `firejail list` and then `firejail --join=id` or start the program with a dedicated name (sometimes the program name works too - like `--join=keepassxc`).

```bash
kmille@linbox:tmp firejail --name=keepass keepassxc
Reading profile /etc/firejail/keepassxc.profile
Reading profile /home/kmille/.config/firejail/keepassxc.local
....
kmille@linbox:~ firejail --join=keepass
Switching to pid 124932, the first child process inside the sandbox
Changing root to /proc/124932/root
Child process initialized in 21.92 ms
linbox% ls
zsh: command not found: ls
```

If there are debugging tools (like ls) missing, the profile uses `private-bin`. You have to restart the application and add the desired debug tools:

```bash
kmille@linbox:tmp firejail --name=keepass --private-bin=ls keepassxc
Reading profile /etc/firejail/keepassxc.profile
...
kmille@linbox:~ firejail --join=keepass
Switching to pid 125282, the first child process inside the sandbox
Changing root to /proc/125282/root
Child process initialized in 22.68 ms
linbox% cd /usr/bin
linbox% ls
keepassxc  keepassxc-cli  keepassxc-proxy  ls
```

Then you can inspect the filesystem and check the whitelisted/blacklisted files.

You can also use `--join-filesystem` (there is also `--join-network`). This will give you a shell within the sandbox. It doesn't work in my case (probably because of other hardening things /o\\). If I try to join a `ping` process:

```bash            
kmille@linbox:~ firejail --join=74748  
Switching to pid 74750, the first child process inside the sandbox
Changing root to /proc/74750/root
Child process initialized in 2.27 ms
/usr/bin/zsh: error while loading shared libraries: libncursesw.so.6: cannot open shared object file: No such file or directory
kmille@linbox:~ sudo firejail --join-filesystem=74748
Switching to pid 74750, the first child process inside the sandbox
Changing root to /proc/74750/root
Error: cannot open /proc/self/fd directory
kmille@linbox:~ 
```

### tracelog filesystem violations

You can call `firejail --tracelog application` or use `tracelog` in the profile. Filesystem access violations will then be logged:

```bash
kmille@linbox:~ firejail --join=keepassxc
...
linbox% pwd
/run/user/1000
linbox% cd app
cd: permission denied: app
...
kmille@linbox:log journalctl -n0 -f
Nov 17 11:06:07 linbox firejail[126044]: blacklist violation - sandbox 126015, name keepassxc, exe 3, syscall stat, path .
Nov 17 11:06:07 linbox firejail[126044]: blacklist violation - sandbox 126015, name keepassxc, exe 3, syscall chdir, path /run/user/1000/app
Nov 17 11:06:07 linbox firejail[126044]: blacklist violation - sandbox 126015, name keepassxc, exe 3, syscall stat, path .
```

After the `keepasxc` update, the socket (to communicate with the browser) moved from `/run/user/1000/` to `/run/user/1000/app/` ([commit](https://github.com/keepassxreboot/keepassxc/commit/40316ac7b9bb5276c3d3ae1be2c3d808db503e3c)), but the firejail profile was not yet updated at that time. The directory was blacklisted in `disable-common.inc` (`blacklist ${RUNUSER}/app`). 

Some more details: in my case, the log entries you see in the `journalctl` output were generated by my manual debugging. Internally, `kepassxc` calls [`QDir().mkpath(subPath)` ](https://github.com/keepassxreboot/keepassxc/blob/12be175d583fbfac5a7b6b250a3bb5f792925285/src/browser/BrowserShared.cpp#L42) and the return value is not checked ([docs](https://doc.qt.io/qt-6/qdir.html#mkpath)). So `keepassxc` could have caught this error and print an error message to the user. Also, the call to `mkpath` (which results in a mkdir syscall) does not lead to a log entry by `--tracelog`.

### trace open, access and connect system calls

```bash
kmille@linbox:tmp firejail --name=keepass --trace=fire.log keepassxc
Reading profile /etc/firejail/keepassxc.profile
...
kmille@linbox:tmp cat fire.log| grep BrowserServer -A 1
15:keepassxc:unlink /run/user/1000/org.keepassxc.KeePassXC.BrowserServer:0
15:keepassxc:mkdir /run/user/1000/app/org.keepassxc.KeePassXC/.VKFFOA:-1
```

If we use `--trace`, we can see that the to call `mkdir` fails (-1 is the return value of the syscall, check [`man 2 mkdir`](https://man7.org/linux/man-pages/man2/mkdir.2.html)). I still don't know where the `.VKFFOA` is coming from.

## Some other debug techniques

- Run firejail with `--debug`. It shows you all it does (a lot)
- Proven to be reliable, but not the smartest approach: use trial and error by commenting lines the firejail profile.
  1. Run the application without firejail. If it also fails, firejail is not the problem
  2. Make a backup of the profile and modify the real one (you can also just copy it to `~/.config/firejail`)
     1. `sudo -s; cd /etc/firejail; cp keepassxc.profile keepassxc.profile.bak; vim keepassxc.profile `
  3. Try the first half: comment all lines with `include`, `blacklist` and `whitelist`. Problem solved? Use binary search to find out the exact line 
  4. Black/Whitelists are not responsible for the problem? Re-enable the original `blacklist`/`whitelist` configuration and comment the half of the lower part of the configuration (`dbus-user`, `private-*`, ...)
  5. move on until you find the responsible line in the profile
  6. fix it permanently by submitting an issue upstream or using `~/.config/firejail/application.local`

# Managing files in a sandbox

### Put a file into a sandbox

```bash
kmille@linbox:~ firejail --put=keepassxc /etc/passwd ~/passwd
Switching to pid 130345, the first child process inside the sandbox
```

```bash
kmille@linbox:~ firejail --ls=keepassxc ~                    
drwx------ 1000     1000             260 .
drwxr-xr-x root     root              60 ..
-rw------- 1000     1000             106 .Xauthority
drwxr-xr-x 1000     1000              80 .cache
drwxr-xr-x 1000     1000             260 .config
-rw------- 1000     1000              42 .histfile
-rw-r--r-- 1000     1000              26 .inputrc
drwxr-xr-x 1000     1000              60 .local
drwxr-xr-x 1000     1000              60 .mozilla
drwxr-xr-x 1000     1000              60 cloud
-rw------- 1000     1000            1790 passwd  <---- this file is new
```

```bash
kmille@linbox:~ firejail --cat=keepassxc ~/passwd
root:x:0:0::/root:/usr/bin/zsh
...
```

### Get a file out of a sandbox

```bash
kmille@linbox:~ firejail --get=keepassxc passwd                     
Switching to pid 130345, the first child process inside the sandbox
kmille@linbox:~ ls passwd                               
-rw------- 1 kmille kmille 1.8K Nov 17 11:31 passwd
```

### Using firejail with AppArmor

```bash
kmille@linbox:~ sudo systemctl start apparmor
kmille@linbox:~ sudo systemctl enable apparmor
kmille@linbox:~ sudo apparmor_parser -r /etc/apparmor.d/firejail-default
kmille@linbox:~ sudo aa-enforce firejail-default
kmille@linbox:~ sudo aa-status                
apparmor module is loaded.
64 profiles are loaded.
64 profiles are in enforce mode.
   /usr/lib/apache2/mpm-prefork/apache2
   ...
   firejail-default
0 profiles are in complain mode.
0 profiles are in kill mode.
0 profiles are in unconfined mode.
7 processes have profiles defined.
7 processes are in enforce mode.
   /usr/lib/firefox/firefox (95771) firejail-default
   /usr/lib/firefox/firefox (95844) firejail-default
   /usr/lib/firefox/firefox (95867) firejail-default
   /usr/lib/firefox/firefox (95913) firejail-default
   /usr/lib/firefox/firefox (95959) firejail-default
   /usr/lib/firefox/firefox (95962) firejail-default
   /usr/lib/firefox/firefox (95967) firejail-default
0 processes are in complain mode.
0 processes are unconfined but have a profile defined.
0 processes are in mixed mode.
0 processes are in kill mode.
```

### Run a second instance of signal-desktop

Run `signal-desktop` with a dedicated, persistent home directory. You have to create `/home/kmille/.config/Signal2` manually before. `signal-desktop` also supports `--user-data-dir` to use it with a different home directory/account.

```ini
kmille@linbox:~ cat ~/.local/share/applications/signal-desktop2.desktop 
[Desktop Entry]
Type=Application
Name=Signal2
Comment=Signal - Private Messenger
Comment[de]=Signal - Sicherer Messenger
Icon=signal-desktop
#Exec=signal-desktop -- %u
Exec=firejail --private=/home/kmille/.config/Signal2 --profile=/etc/firejail/signal-desktop.profile /usr/bin/signal-desktop
Terminal=false
Categories=Network;InstantMessaging;
StartupWMClass=Signal
MimeType=x-scheme-handler/sgnl;x-scheme-handler/signalcaptcha;
Keywords=sgnl;chat;im;messaging;messenger;sms;security;privat;
X-GNOME-UsesNotifications=true
```

```bash
kmille@linbox:~ alias signal2
signal2='firejail --private=/home/kmille/.config/Signal2 --profile=/etc/firejail/signal-desktop.profile /usr/bin/signal-desktop'
```

## jailjack

`sudo jailcheck` does a basic audit for all running sandboxes:

```bash
kmille@linbox:firejail sudo jailcheck
....
5459:kmille::/usr/bin/firejail /
132309:kmille:keepassxc:/usr/bin/firejail /usr/bin/keepassxc                                                                                                             
   Warning: AppArmor not enabled                                                                                                                                         
   Virtual dirs: /home/kmille, /tmp, /var/tmp, /dev, /etc, /bin, /usr/share                                              
   Networking: disabled
```

This means that `/home/kmille`, `/etc` and the other virtual directories are fresh. Only dedicated stuff is bound/copied to these directories. Things like `lib` looks like on the host system. More infos in the [man page](https://firejail.wordpress.com/man-jailcheck/).

### whitelist firejail users

You can reduce the risk of the setuid binary by whitelisting the users, firejail will work with. You can use [`firecfg --add-users`](https://man7.org/linux/man-pages/man1/firecfg.1.html). You can also use `force-nonewprivs yes` under special [circumstances](https://firejail.wordpress.com/documentation-2/basic-usage/#suid).

```bash
kmille@linbox:~ cat /etc/firejail/firejail.users          
kmille
kmille@linbox:~ sudo -u nobody bash                        
[sudo] password for kmille:               
[nobody@linbox ~]$ gnome-calculator 
Error: the user is not allowed to use Firejail.
Please add the user in /etc/firejail/firejail.users file,
either by running "sudo firecfg", or by editing the file directly.
See "man firejail-users" for more details.
Authorization required, but no authorization protocol specified
(gnome-calculator:99058): Gtk-WARNING **: 17:49:20.595: cannot open display: :0
[nobody@linbox ~]$ 
```

# How to create own profiles

- Use the template profile in [`/usr/share/doc/firejail/profile.template`](https://github.com/netblue30/firejail/blob/master/etc/templates/profile.template) and read the comments
- https://github.com/netblue30/firejail/wiki/Creating-Profiles
- https://firejail.wordpress.com/documentation-2/building-custom-profiles/
- https://github.com/netblue30/firejail/wiki/Creating-overrides
- You can use `--build`

```bash
kmille@linbox: firejail --build mpv ~/Downloads/test.mp3 
 (+) Audio --aid=1 (pcm_s24le 2ch 44100Hz)
AO: [pulse] 44100Hz stereo 2ch s32
A: 00:00:00 / 00:00:01 (69%)

Exiting... (End of file)
--- Built profile begins after this line ---
# Save this file as "application.profile" (change "application" with the
# program name) in ~/.config/firejail directory. Firejail will find it
# automatically every time you sandbox your application.
#
# Run "firejail application" to test it. In the file there are
# some other commands you can try. Enable them by removing the "#".

# Firejail profile for mpv
# Persistent local customizations
#include mpv.local
# Persistent global definitions
#include globals.local

### Basic Blacklisting ###
### Enable as many of them as you can! A very important one is
### "disable-exec.inc". This will make among other things your home
### and /tmp directories non-executable.
include disable-common.inc      # dangerous directories like ~/.ssh and ~/.gnupg
#include disable-devel.inc      # development tools such as gcc and gdb
#include disable-exec.inc       # non-executable directories such as /var, /tmp, and /home
#include disable-interpreters.inc       # perl, python, lua etc.
include disable-programs.inc    # user configuration for programs such as firefox, vlc etc.
#include disable-shell.inc      # sh, bash, zsh etc.
#include disable-xdg.inc        # standard user directories: Documents, Pictures, Videos, Music

### Home Directory Whitelisting ###
### If something goes wrong, this section is the first one to comment out.
### Instead, you'll have to relay on the basic blacklisting above.
whitelist ${HOME}/.config/pipewire
whitelist ${HOME}/Downloads
whitelist ${HOME}/Downloads/test.mp3
include whitelist-common.inc

### Filesystem Whitelisting ###
include whitelist-run-common.inc
whitelist ${RUNUSER}/pulse
include whitelist-runuser-common.inc
include whitelist-usr-share-common.inc
include whitelist-var-common.inc

#apparmor       # if you have AppArmor running, try this one!
caps.drop all
ipc-namespace
netfilter
#no3d   # disable 3D acceleration
#nodvd  # disable DVD and CD devices
#nogroups       # disable supplementary user groups
#noinput        # disable input devices
nonewprivs
noroot
#notv   # disable DVB TV devices
#nou2f  # disable U2F devices
#novideo        # disable video capture devices
protocol unix,
net none
seccomp !chroot # allowing chroot, just in case this is an Electron app
shell none
#tracelog       # send blacklist violations to syslog

#disable-mnt    # no access to /mnt, /media, /run/mount and /run/media
private-bin mpv,
#private-cache  # run with an empty ~/.cache directory
private-dev
private-etc machine-id,pulse,pipewire,fonts,mpv,login.defs,
#private-lib
private-tmp

#dbus-user none
#dbus-system none

#memory-deny-write-execute
kmille@linbox:firejail 
```

# Resources 

### About the benefits/risks of using firejail

- https://github.com/netblue30/firejail/issues/3046
- https://madaidans-insecurities.github.io/linux.html#firejail

### General

- https://wiki.archlinux.org/title/firejail
- https://github.com/netblue30/firejail/wiki/Debugging-Firejail
- https://firejail.wordpress.com/features-3/man-firejail/



