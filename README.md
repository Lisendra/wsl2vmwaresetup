# wsl2vmwaresetup
WSL2 &amp; VMWare/Vagrant as a development environment

This is for WSL2! check you WSL version.

1 : Setup your reps for Debian testing & update your shell & install some core features

1.1 : Setup your reps & comment out if you need sources, contrib or non-free 
```
sudo nano /etc/apt/source.list
```

```
deb http://deb.debian.org/debian/ testing main #contrib non-free
#deb-src http://deb.debian.org/debian/ testing main #contrib non-free

deb http://deb.debian.org/debian/ testing-updates main #contrib non-free
#deb-src http://deb.debian.org/debian/ testing-updates main #contrib non-free

deb http://deb.debian.org/debian-security testing-security main
#deb-src http://deb.debian.org/debian-security testing-security main
```
1.2 : Update your shell
```
sudo apt update && sudo apt upgrade
```
1.3 : Install some core features
```
sudo apt install curl wget apt-transport-https dirmngr htop git
```

2 : Setup Auto-start/services (```systemd``` and ```snap``` support)

There various ways to autostart process in WSL2, after trials & errors i have choosen this solution provide by [shayne](https://github.com/shayne "Shayne profile on Github") at [wsl2-hacks on Github](https://github.com/shayne/wsl2-hacks "wsl2-hacks on Github")

2.1 : install aditional packages

```
sudo apt install dbus policykit-1 daemonize
```

2.2 : Create a fake-```bash```

This fake shell will intercept calls to ```wsl.exe bash ...``` and forward them to a real bash running in the right environment for ```systemd```. If this sounds like a hack-- well, it is. However, I've tested various workflows and use this daily. That being said, your mileage may vary.

```sudo touch /usr/bin/bash && sudo chmod +x /usr/bin/bash && sudo nano /usr/bin/bash```

Add the following, be sure to replace ```<YOURUSER>``` with your WSL2 Linux username

```
#!/bin/bash
# your WSL2 username
UNAME="<YOURUSER>"

UUID=$(id -u "${UNAME}")
UGID=$(id -g "${UNAME}")
UHOME=$(getent passwd "${UNAME}" | cut -d: -f6)
USHELL=$(getent passwd "${UNAME}" | cut -d: -f7)

if [[ -p /dev/stdin || "${BASH_ARGC}" > 0 && "${BASH_ARGV[1]}" != "-c" ]]; then
    USHELL=/bin/bash
fi

if [[ "${PWD}" = "/root" ]]; then
    cd "${UHOME}"
fi

# get pid of systemd
SYSTEMD_PID=$(pgrep -xo systemd)

# if we're already in the systemd environment
if [[ "${SYSTEMD_PID}" -eq "1" ]]; then
    exec "${USHELL}" "$@"
fi

# start systemd if not started
/usr/sbin/daemonize -l "${HOME}/.systemd.lock" /usr/bin/unshare -fp --mount-proc /lib/systemd/systemd --system-unit=basic.target 2>/dev/null
# wait for systemd to start
while [[ "${SYSTEMD_PID}" = "" ]]; do
    sleep 0.05
    SYSTEMD_PID=$(pgrep -xo systemd)
done

# enter systemd namespace
exec /usr/bin/nsenter -t "${SYSTEMD_PID}" -m -p --wd="${PWD}" /sbin/runuser -s "${USHELL}" "${UNAME}" -- "${@}"
```
2.3 : Set the fake-```bash``` as our ```root``` user's shell

We need ```root``` level permission to get systemd setup and enter the environment. WSL2 need to default to the ```root``` user and when ```wsl.exe``` is executed the fake-```bash``` will do the right thing.

Edit the ```/etc/passwd``` file:

```
sudo nano /etc/passwd
```

Find the line starting with ```root:```, it should be the first line. Change it to:

```
root:x:0:0:root:/root:/usr/bin/bash
```

Exit out of ```/``` & close the WSL2 shell

Shutdown WSL2 and to change the default user to ```root```.

In a ```PowerShell``` terminal run:

```
wsl --shutdown
<Yourshellname> config --default-user root
```
Re-open WSL2

Everything should be in place. Fire up WSL via the MS Terminal or just ```wsl.exe```. You should be logged in as your normal user and systemd should be running

You can test by running the following in WSL2:

```
systemctl is-active dbus
active
```

Create /etc/rc.local (optional)

If you want to run certain commands when the WSL2 VM starts up, this is a useful file that's automatically ran by ```systemd```.

```sudo touch /etc/rc.local && sudo chmod +x /etc/rc.local && sudo editor /etc/rc.local```
Add the following:
```
#!/bin/sh -e

# your commands here...

exit 0
```
/etc/rc.local is only run on "boot", so only when you first access WSL2 (or it's shutdown due to inactivity/no-processes). To test you can shutdown WSL via PowerShell/CMD wsl --shutdown then start it back up with wsl.
