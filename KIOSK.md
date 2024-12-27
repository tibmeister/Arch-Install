![Arch Linux](https://archlinux.org/static/logos/archlinux-logo-dark-90dpi.ebdee92a15b3.png)
# Arch Linux Installation Guide - i3

## Introduction
The goal of this Arch Linux installation guide is to provide steps to install a simple Window Manager and get a GUI up and running.  This example will be to have Firefox startup after autologin and connect to a Home Assistant interface using the Kiosk Mode of Firefox.

This guide is a mix of knowledge and information as listed in the [Appendix A - Resources](#appendix-a---resources) section.  At the end of the day, this is how I do things, your mileage may vary.


## After Initial Boot - Window Manager
We will be installing the i3 Window Manager due to it's very light weight resource requirement.
Start by installing a bunch of packages.
```
pacman -S git base-devel yay webkit2gtk gcr xorg-server-utils xorg-server xorg-xinit i3-wm glances neofetch i3-gaps i3blocks i3lock numlockx zsh kitty
pacman -S lightdm lightdm-gtk-greeter --needed
sudo pacman -S noto-fonts ttf-ubuntu-font-family ttf-dejavu ttf-freefont ttf-liberation ttf-droid ttf-roboto terminus-font
```

## Appendix A - Resources  
[Desktop Environment](https://wiki.archlinux.org/title/Desktop_environment)  
[i3](https://wiki.archlinux.org/title/I3#Installation)
[LightDM Greeter](https://wiki.archlinux.org/title/LightDM)  