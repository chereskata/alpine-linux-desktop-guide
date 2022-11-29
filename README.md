# alpine-linux-desktop-guide
A memorization help for myself to remember what i have done to setup my laptop workstation

## Post install

### Issue: firefox(-esr) launches only in safe mode
As by suggestion of user ikke on [irc](https://wiki.alpinelinux.org/wiki/Alpine_Linux:IRC), you have to do 
hide the safe-mode .desktop entries from gnome-shell, because it maps them
even to the regular firefox and firefox-esr executables. So in fact firefox
does not launch in safe mode at all if you click the safe mode icon explicity
```sh
# for firefox -esr
cp /usr/share/applications/firefox-esr-safe.desktop ~/.local/share/applications/
echo "Hidden=true" >> .local/share/applications/firefox-esr-safe.desktop

# for firefox
cp /usr/share/applications/firefox-safe.desktop ~/.local/share/applications/
echo "Hidden=true" >> .local/share/applications/firefox-safe.desktop
```
