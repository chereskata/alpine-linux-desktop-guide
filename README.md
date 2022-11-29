# alpine-linux-desktop-guide
A memorization help for myself to remember what i have done to setup my laptop workstation

## Post install

### Issue: no autologin in gdm

### Issue: nftables missing
This is the successor of iptables. Really miss [pf](https://why-openbsd.rocks/fact/pf/) of OpenBSD btw.
```sh
# install and enable nftables
su -l
apk add nftables
rc-update add nftables boot

# note: you still should review the default ruleset
```

### Issue: gnome-keyring is auto-launching on login
As i prefer to do authentication based stuff in console, i want to disable
the gnome keyring functionality otherwise leads to an popup, which is blocking
the desktop till you have entered the password. This is a usability drawback 
for me.
```sh
(cat /etc/xdg/autostart/gnome-keyring-ssh.desktop; echo Hidden=true) > ~/.config/autostart/gnome-keyring-ssh.desktop
(cat /etc/xdg/autostart/gnome-keyring-pkcs11.desktop; echo Hidden=true) > ~/.config/autostart/gnome-keyring-pkcs11.desktop
(cat /etc/xdg/autostart/gnome-keyring-secrets.desktop; echo Hidden=true) > ~/.config/autostart/gnome-keyring-secrets.desktop
```

### Issue: firefox(-esr) launches only in safe mode
As by suggestion of user ikke on [irc](https://wiki.alpinelinux.org/wiki/Alpine_Linux:IRC), you have to 
hide the safe-mode .desktop entries from gnome-shell, because it maps them
even to the regular firefox and firefox-esr executables. So in fact firefox
does not launch in safe mode at all if you click the safe mode icon explicity
```sh
# for firefox -esr
(cat /usr/share/applications/firefox-esr-safe.desktop; echo Hidden=true) > ~/.local/share/applications/firefox-esr-safe.desktop

# for firefox
(cat /usr/share/applications/firefox-safe.desktop; echo Hidden=true) > ~/.local/share/applications/firefox-safe.desktop
```

