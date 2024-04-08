# AlmaLinux setup notes by Blair

## Rationale

I'd previously favoured Fedora (see [fedorasetup](https://github.com/blairw/fedorasetup)) for Linux purposes, but I'd found that it still moves relatively too quickly. Maybe it's because I'm now quite satisified with the featureset of something circa Fedora 34 (2021), which is good enough for most server and workstation use cases. In the past I went all the way to Arch Linux just to get all the newest features that I wanted...

AlmaLinux is now the de facto stable downstream distro based on Fedora, or at least [CERN seems to feel that way](https://linux.web.cern.ch/almalinux/). If it's good enough for particle physics then I'm certainly receptive of it. 🚀

## About these Setup Notes

Since this document is mostly for my own records, unless you are me, not everything below is certainly relevant for you. But maybe some of it is / maybe this might help someone else trying to get something working? 😊

## Install AlmaLinux

- Avoid secure boot if you intend to use VirtualBox - unless you are absolutely sure that it is able to work.
- If installing on a machine that previously had Linux, especially if you are dual-booting with Windows, the Anaconda installer might crash due to `/boot/efi` being full. If this happens:
    - Boot from the Live USB again, if you can. (If you can't, you might need to rebuild the Live USB.)
    - Use <kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>1</kbd> (or was it 2, or 3...?) to get access to the terminal.
    - `mkdir ~/boot-efi` to give yourself a mountpoint
    - `lsblk` to list the partitions
    - Use `mount` to mount the correct partition to `~/boot-efi`
    - Then you can use the terminal to go in and delete the stuff inside `/boot/efi` left over from the previous Linux installation. In my case, I was switching from Fedora to AlmaLinux, so I deleted all the old Fedora stuff.

## Sensible settings

In the GNOME settings app:

- About &rarr; Device Name
- Power
    - Automatic Screen Brightness: turn OFF (default: on)
    - Dim Screen: turn OFF (default: on)
    - Screen black: set to NEVER if this is an always-on machine
    - Automatic suspend: set to OFF if this is an always-on machine
- Removeable Media
    - Turn everything off, we don't want this autoplay nonsense!
- Apply Network profiles if needed (e.g., static IP addresses)

## Switch to zsh

(Git is needed for oh-my-zsh)

```bash
sudo dnf install -y git zsh
touch ~/.zshrc
chsh -s /bin/zsh
```

Then logout and log back in, then setup oh-my-zsh:

```zsh
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

## Setup dnf with RPMFusion and EPEL

**Caution:** If intending to use Virtualbox, you may need to follow the `--exclude=kernel*` instructions based on [fedorasetup](https://github.com/blairw/fedorasetup), rather than the update-all approach documneted here.

That said:

First, set `fastestmirror=True` in `/etc/dnf/dnf.conf` as per https://discussion.fedoraproject.org/t/how-to-change-default-dnf-server-country/73107/4?u=ankursinha

Next, run the tasks - note that `noglob` is required for zsh as per https://github.com/ohmyzsh/ohmyzsh/issues/31.

```zsh
# Update everything
noglob sudo dnf update -y

# Get RPMFusion and EPEL together
sudo dnf install --nogpgcheck https://dl.fedoraproject.org/pub/epel/epel-release-latest-$(rpm -E %rhel).noarch.rpm
sudo dnf install --nogpgcheck https://mirrors.rpmfusion.org/free/el/rpmfusion-free-release-$(rpm -E %rhel).noarch.rpm https://mirrors.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-$(rpm -E %rhel).noarch.rpm

# Refresh the cache
sudo dnf makecache --refresh

# Then do this one, which forces a bit of a cleanup
sudo dnf install ffmpeg obs-studio

# Then do the rest
sudo dnf install audacity vlc
sudo dnf install geany geany-plugins-addons geany-plugins-autoclose geany-plugins-automark geany-plugins-codenav geany-plugins-git-changebar geany-plugins-geanyprj geany-plugins-lineoperations geany-plugins-markdown geany-plugins-overview geany-plugins-projectorganizer geany-plugins-treebrowser geany-plugins-workbench
```

Adapted from https://rpmfusion.org/Configuration/

Use this method of installing EPEL from the Fedora project rather than the one in AlmaLinux, this will give us a newer version.

NOTE: If you're told that the update(s) about have installed a new kernel, reboot now.

## Setup dnf with CRB

CRB is needed for the R programming language and probably some other useful stuff too...

```zsh
sudo dnf config-manager --set-enabled crb
sudo dnf makecache --refresh
sudo dnf install R
```

## Install useful packages

```zsh
# Common install for workstation + server:
noglob sudo dnf install -y aria2 htop nano neofetch screen npm

# If this machine is a workstation:
noglob sudo dnf install -y chromium gedit-plugin-textsize gimp gnome-extensions-app gnome-tweaks libreoffice nextcloud-client-nautilus overpass-fonts
```

## Setup Cockpit

```zsh
sudo dnf install cockpit
sudo systemctl enable --now cockpit.socket
sudo firewall-cmd --add-service=cockpit
sudo firewall-cmd --add-service=cockpit --permanent
```

Adapted from https://cockpit-project.org/running

Once set up, go to Cockpit:

- Overview &rarr; View metrics and history &rarr; **Install cockpit-pcp**
- Then logout, log back in
- Overview &rarr; View metrics and history &rarr; **Metrics settings** and enable **Collect metrics** but maybe not Export to network (!).
- Applications &rarr; Machines &rarr; Install

## Setup Dropbox

Get this started sooner rather than later, so that it can start fetching your files in the background.

```zsh
# Install
cd ~ && wget -O - "https://www.dropbox.com/download?plat=lnx.x86_64" | tar xzf -

# Get Dropbox py
mkdir -p ~/00blair/dropboxpy
wget -O ~/00blair/dropboxpy/dropbox.py https://www.dropbox.com/download\?dl\=packages/dropbox.py
chmod +x ~/00blair/dropboxpy/dropbox.py 

# Run
~/.dropbox-dist/dropboxd

# Setup exclusions
~/00blair/dropboxpy/dropbox.py exclude add ~/Dropbox/blair-extras2
~/00blair/dropboxpy/dropbox.py exclude add ~/Dropbox/blair-extras3
~/00blair/dropboxpy/dropbox.py exclude add ~/Dropbox/blair-extras4
```

You will need to configure firewall to enable LAN sync properly:

```zsh
# get zone name e.g. `public`, `FedoraServer`
sudo firewall-cmd --get-active-zones

# confirm that dropbox-lansync is available
sudo firewall-cmd --get-services | grep dropbox

# enable, zone=public
sudo firewall-cmd --zone=public --add-service=dropbox-lansync --permanent

# enable, zone=FedoraServer
sudo firewall-cmd --zone=FedoraServer --add-service=dropbox-lansync --permanent

# reload
sudo firewall-cmd --reload
```

I use this version because it works across server and workstation environments, across desktop environments, with no dependency on package managers and so on.


## Samba (SMB) - server-side configuration (if this machine will host shares)

First, install the necessary packages:

```zsh
sudo dnf install samba samba-common samba-client
```

Next, setup your folder:

```zsh
mkdir /home/blair/mynetworkdrive
```

Then, configure the conf file (`/etc/samba/smb.conf`):

```markdown
[global]
...

# Force encryption as per https://serverfault.com/a/1088412
server signing = mandatory
server min protocol = SMB3
server smb encrypt = required

...

[mynetworkdrive]
path = /home/blair/mynetworkdrive
writeable = Yes
browseable = Yes
public = no
valid users = blair
create mask = 0644
directory mask = 0755
write list = user
```

Remove the _force encryption_ part if you're setting up a server for clients that cannot support SMB encryption for some very special reasons.

Next, go to Cockpit &rarr; Networking &rarr; Firewall &rarr; Add services (to the correct zone!) &rarr; Filter services to **samba**. Enable **samba** ("TCP: 139, 445; UDP: 138").

Finally:

```zsh

# Setup account
sudo smbpasswd -a blair

# Set correct folder permissions
sudo semanage fcontext -a -t public_content_t "/home/blair/mynetworkdrive(/.*)?"

# Activate correct folder permissions
sudo restorecon -R /home/blair/mynetworkdrive
chcon -t samba_share_t /home/blair/mynetworkdrive

sudo setsebool -P samba_domain_controller on
sudo setsebool -P samba_enable_home_dirs on
```

Note: systemctl commands...

```zsh
sudo systemctl enable smb
sudo systemctl restart smb
sudo systemctl status smb
```

... See also https://docs.fedoraproject.org/en-US/quick-docs/samba/


## Samba (SMB) - client-side configuration (if this machine will connect to shares)

First, `mkdir /media/myshare1`, then:

Add to `/etc/fstab`:

```
//myserver/myshare1 /media/myshare1 cifs credentials=/home/myusername/myshare-credentials.txt,noauto,user 0 0
```

(I found that I had to specify `/home/myusername/` instead of `~/`)

Credentials file formatted like so:

```
username=blair
password=mypasswordgoeshere
```

Then as per https://superuser.com/questions/1444883/user-cifs-mounts-not-supported-fedora-30, we need to set the suid bit:

```zsh
sudo chmod u+s /bin/mount
sudo chmod u+s /bin/umount
sudo chmod u+s /usr/sbin/mount.cifs
```

Then need to refresh systemd to use the updated fstab:

```zsh
sudo systemctl daemon-reload
```

Then can mount like:

```zsh
mount //myserver/myshare1
```

To check the properties of the Samba share, run `mount -t cifs`.




## Git / GitHub credentials saver

As per https://unix.stackexchange.com/questions/379272/storing-username-and-password-in-git

```zsh
git config --global credential.helper store

# to remove
git config --global --unset credential.helper
```

## Install Snaps

Adapted from https://www.liquidweb.com/kb/how-to-install-snapcraft-on-ubuntu-centos/

```zsh
sudo dnf install -y snapd
sudo systemctl enable --now snapd.socket
sudo ln -s /var/lib/snapd/snap /snap

# Close Terminal and open it again here

sudo snap install lolcat
```

## Install Flatpaks

Download the flatpak file and then just double-click it, it will prompt you to install using a little GUI.

- Handbrake: https://handbrake.fr/downloads.php and go to Flatpak section


## Install from GNOME Software Center

- Dialect
- gitg
- Do NOT use this for: Dropbox


## Fix rendering of some fonts (e.g., Monaco)

https://superuser.com/questions/93924/monaco-font-not-antialiased-in-some-gtk-apps


## Install some software manually, if needed

- Brave: https://brave.com/linux/#fedora-rockyrhel
- FSearch: https://copr.fedorainfracloud.org/coprs/cboxdoerfer/fsearch/
- RStudio: https://posit.co/download/rstudio-desktop/
- VSCodium: https://vscodium.com/#install-on-fedora-rhel-centos-rockylinux-opensuse-rpm-package


## Pasaffe

PATH needs to include `/home/blair/.local/bin` but otherwise confirmed working! pasaffe-0.58-1 on AlmaLinux 9.3 (python --version == Python 3.9.18).

Adapted from https://answers.launchpad.net/pasaffe/+question/658239

1. Download latest tar.gz from https://launchpad.net/pasaffe
2. `sudo dnf install -y rpmdevtools python3-pip python3-distutils-extra`
3. `pip3 install unidecode`
4. Navigate to the folder where the latest tar.gz was downloaded
5. `tar xvzf pasaffe-0.58.tar.gz` (replace with equivalent filename)
6. Navigate to extracted folder
7. `python setup.py bdist_rpm`
8. `find dist/*.rpm`
9. `sudo dnf install -y dist/pasaffe-0.58-1.noarch.rpm`
10. If needed, symlink an existing psafe3 database to `~/.local/share/pasaffe/pasaffe.psafe3`


## Setup folder for custom fonts

```zsh
mkdir ~/.local/share/fonts
# then copy whatever fonts you want
fc-cache -fv
```

## Replicate pbcopy

Adapted from https://ostechnix.com/how-to-use-pbcopy-and-pbpaste-commands-on-linux/

1. `sudo dnf install -y xclip xsel`
2. `nano ~/.zshrc` (unless you are using bash, in which case it's `~/.bashrc`)
3. Aliases...
	```
	alias pbcopy='xclip -selection clipboard'
	alias pbpaste='xclip -selection clipboard -o'
	```
4. Now you can use `pbcopy`! For example, grabbing the current directory onto the clipboard: `pwd | tr -d '\n' | pbcopy`

## GNOME Extensions

- https://extensions.gnome.org/extension/307/dash-to-dock/
- You may also consider installing some from https://www.debugpoint.com/gnome-40-extensions/



## GNOME prevent sleep-if-no-one-logs-in

Adapted from https://askubuntu.com/a/1132068

Not tested yet, might need `sudo dnf install dbus-x11`, TODO: to be tested...

```
blair@fedoraworkstation ~
❯ sudo -u gdm dbus-launch gsettings get org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type
'suspend'

blair@fedoraworkstation ~
❯ sudo -u gdm dbus-launch gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type "nothing"

blair@fedoraworkstation ~
❯ sudo -u gdm dbus-launch gsettings get org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type
'nothing'
```

## Ongoing Maintenance

### Checking NTP

See https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_basic_system_settings/configuring-time-synchronization_configuring-basic-system-settings#using-chrony_configuring-time-synchronization

