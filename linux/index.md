# Linux Cheat Sheet


* command cheatsheet
  #+begin_src sh
curl cheat.sh/command | bat
  #+end_src


* shell
** for loop

#+begin_src shell
colors=(red orange white "light gray")
for item in ${colors[@]}; do
    echo $item;
done
#+end_src

** time
#+begin_src shell
date "+%F %H:%M:%S.%N"
# 2023-01-25 23:55:00.152982295
#+end_src

*** set timezone
#+begin_src sh
sudo timedatectl set-timezone America/Vancouver
#+end_src

#+begin_src sh
sudo ln -s /usr/share/zoneinfo/America/Vancouver /etc/localtime
#+end_src



*** use RTC in UTC
#+begin_src sh
sudo timedatectl set-local-rtc 0
#+end_src


*** System clock synchronized
#+begin_src sh
sudo timedatectl set-ntp true
#+end_src


** copy to clipboard
#+begin_src shell
echo "copy" | xclip -set clip
#+end_src

** sort
*** options
| *-k* [n, m] | 选择区间           |
| *-t*        | g分隔符            |
| *-M*        | 排序月份           |
| *-n*        | 以数值型进行排序   |
| *-h*        | 可读的数字(带单位) |
| *-f*        | 忽略大小写         |
*** example

** path

*** get script path
#+begin_src sh
DIR=$(dirname $(readlink -f "$0"))
#+end_src

*** get invocation path

#+begin_src sh
INV_DIR=$"PWD"
#+end_src

** background

*** start a bg process and kill it

#+begin_src sh
sleep 60 &

# Get last bg pid
pid=$!

# ...
kill $pid
#+end_src


** sed
*** replace
#+begin_src shell
sed -E -i 's/SEARCH_REGEX/REPLACEMENT/g' INPUTFILE
sed -E -i 's/Xft.dpi: [0-9]+/Xtf.dpi 144/g' ~/.Xresources
#+end_src

+ ~-i~ - Edit a file and don't write the output to stdout.
+ ~s~ - The substitute command.
+ ~///~ - Delimiter character. It can be any character but usually the slash (/) character is used.
+ ~g~ - Global replacement flag. All occurrences are replaced if ~g~ provided.

** awk
- *NOTE* use '' instead of "" !
#+begin_src shell
pactl get-sink-volume 0 | head -n1 | awk '{print $5}' | sed "s/%//g"

echo 'this is a test' | awk '{print $3}'
#+end_src


** tmux

*** config
#+begin_src sh
unbind C-b
set-option -g prefix M-Space
bind-key M-Space send-prefix

set -g default-terminal screen-256color
set-option -ga terminal-overrides ",*256col*:Tc"
#+end_src

*** options
+ start =tmux new -s *SESSION_NAME*=
+ attach =tmux a -t *SESSION_NAME*=
+ detach =send-prefix d=
+ create window =send-prefix c=
+ show sessions =tmux list-sessions=

** jobs
*** list background process
+ =jobs=

** environment

** condition
*** 中括号
**** 单中括号 [ ]
a. =[ ]= 两个符号左右都要有空格分隔
b. 内部操作符与操作变量之间要有空格：如 [ "a" = "b" ]
c. 字符串比较中，> < 需要写成> \< 进行转义
d. =[ ]= 中字符串或者${}变量尽量使用""双引号扩住，以避免值未定义引用而出错
e. =[ ]= 中可以使用 –a –o 进行逻辑运算
f. =[ ]= 是 shell 内置命令
**** 双中括号
a. =[[ ]]= 两个符号左右都要有空格分隔
b. =[[ ]]= 内部操作符与操作变量之间要有空格：如 =[[ "a" = "b" ]]=
c. =[[ ]]= 字符串比较中，可以直接使用 > < 无需转义
d. =[[ ]]= 中字符串或者${}变量尽量使用"" 双引号扩住，如未使用""会进行模式和元字符匹配
e. =[[ ]]= 内部可以使用 && || 进行逻辑运算
f. =[[ ]]= 是 bash keyword
***** example
****** equal
#+begin_src shell
if [[ $randm == 1 ]] ; then
    echo $random == 1
fi
#+end_src
****** not equal
#+begin_src shell
if [[ $randm != 1 ]] ; then
    echo $random != 1
fi
#+end_src

*** exist

**** environment variable
- check if a environment variable is set
#+begin_src bash
if [[ -z ${env} ]]; then
fi
#+end_src

**** command
- check if a command exists
#+begin_src bash
if ! command -v python &> /dev/null
fi
#+end_src

*** 文件夹或文件

**** 文件夹
#+begin_src shell
if [ ! -d "folder" ] ; then
fi
#+end_src

**** 文件

#+begin_src shell
if [ ! -f "file" ] ; then
fi
#+end_src


** find files
#+begin_src shell
find ./fuzz-results/crashes -type f -name "id:$1*" -print0 | while read -d $'\0' file
do
    echo $file
done
#+end_src

** xargs
多行转单行输入, 默认分隔符为换行
#+begin_src shell
pacman -Q | grep -i "haskell" | xargs sudo pacman -S --noconfirm
#+end_src

placeholder

#+begin_src shell
xargs -I {} mv dir1/{} dir2/{}
xargs -I '{}' mv dir1/'{}' dir2/'{}'
#+end_src

** stream
| symbol      | meanings         |
| 0           | stdin            |
| 1           | stdout           |
| 2           | stderr           |
| 2>&1        | stderr to stdout |
| 1>          | stdout redirect  |
| 2>          | stderr redirect  |
| 1>/dev/null | write to void    |

** uniq
仅输出 unique 行
#+begin_src shell
uniq -u
#+end_src

** grep & rg
*** grep
*** no ignore
#+begin_src shell
rg --no-ignore
#+end_src

** zip
*** zip

**** compress
- -q 不显示命令
- -r 递归
- -y 不解析 symbol link

#+begin_src shell
zip -qry zip_file.zip file1 file2 dir1
#+end_src

**** decompress
#+begin_src shell
unzip jacoco-0.8.7.zip -d jacoco
#+end_src

**** list
#+begin_src shell
unzip -l zipfile.zip
#+end_src

*** tar
**** compress
#+begin_src
tar -czf sample.tar.gz ./sample
#+end_src

**** decompress
#+begin_src shell
tar -xzvf sample.tar.gz -C ./sample
#+end_src

** git disable zsh status
#+begin_src sh
git config --add oh-my-zsh.hide-status 1
git config --add oh-my-zsh.hide-dirty 1
#+end_src

** zsh keybind

- ~^F~ ctrl-F
- ~^[F~ alt-F

*** list
#+begin_src zsh
bindkey -L
#+end_src

* kernel

** dependencies
#+begin_src shell
sudo apt-get install libncurses-dev gawk flex bison openssl libssl-dev dkms libelf-dev libudev-dev libpci-dev libiberty-dev autoconf llvm
#+end_src

** build
#+begin_src shell
make menu_config
make -j$(nproc)
sudo make modules_install -j$(nproc)
sudo make install -j$(nproc)
#+end_src

** get current kernel config
#+begin_src sh
cat /boot/config

# or

zcat /proc/config.gz > config
#+end_src

** kernel boot command
#+begin_src sh
cat /proc/cmdline
#+end_src

* network
** iw
- 格式
#+begin_src shell
iw [ OPTIONS ] { help [ command ] | OBJECT COMMAND }
OBJECT := { dev | phy | reg }
OPTIONS := { --version | --debug }
#+end_src
- 搜索
#+begin_src shell
iw dev <devname> scan
#+end_src
- 显示设备
#+begin_src shell
iw dev
#+end_src
- 显示设备信息
#+begin_src shell
iw dev <devename> info
#+end_src
- 显示连接信息
#+begin_src shell
iw dev <devname> link
#+end_src
- 连接
#+begin_src shell
iw dev <devname> connect [-w] <SSID> [<freq in MHz>] [<bssid>] [key 0:abcde d:1:6162636465] [mfp:req/opt/no]
# Join the network with the given SSID (and frequency, BSSID).
#+end_src

** iwd
*** connect
#+begin_src sh
[iwd]# help
[iwd]# device list
[iwd]# station wlan0 scan
[iwd]# station wlan0 get-networks
[iwd]# station wlan0 connect SSID
[iwd]# station wlan0 show
#+end_src

** Network Manager
- 扫描
#+begin_src shell
nmcli device wifi rescan
#+end_src
- 显示
#+begin_src shell
nmcli device wifi list
#+end_src
- 连接
#+begin_src shell
nmcli device wifi connect <SSID> password <password> [hidden yes]
#+end_src


- connection up/down
#+begin_src shell
nmcli connection up/down <connection>
#+end_src

** ubcsecure
#+begin_src shell
nmcli connection add type wifi con-name "ubcsecure" ifname wlan0 ssid "ubcsecure" -- wifi-sec.key-mgmt wpa-eap 802-1x.eap peap 802-1x.phase2-auth mschapv2 802-1x.identity "yayuwang" 802-1x.anonymous-identity "yayuwang" 802-1x.password "password..."
#+end_src


*** iwd configuration
#+begin_src sh
# /var/lib/iwd/ubcsecure.8021x
[Security]
EAP-Method=PEAP
EAP-Identity=username
EAP-PEAP-Phase2-Method=MSCHAPV2
EAP-PEAP-Phase2-Identity=username
EAP-PEAP-Phase2-Password=password

[Settings]
AutoConnect=true
#+end_src

** ss
- ss is the alternative of netstat
#+begin_src shell
ss -tunpl
#+end_src


* High DPI

** Netease Music
#+begin_src conf
#/opt/netease/netease-cloud-music/netease-cloud-music.bash

export QT_SCALE_FACTOR=2
#+end_src

** ghidra
#+begin_src conf
#/opt/ghidra/support/launch.properties

VMARGS_LINUX=-Dsun.java2d.uiScale=2
#+end_src


** Wechat
#+begin_src conf
#/opt/apps/com.qq.weixin.deepin/files/run.sh
export DEEPIN_WINE_SCALE=2.00
#+end_src


* user management

** add user

#+BEGIN_SRC bash
useradd miao
#+END_SRC

*** create home directory
#+begin_src bash
useradd -m miao
#+end_src


*** add sudo access
#+BEGIN_SRC bash
usermod -aG sudo username
#+END_SRC

** groups

*** list groups
#+begin_src sh
groups $USER
#+end_src

*** add user to a group
#+begin_src shell
usermod -aG group user
#+end_src

** sudo

*** list user's privilege
#+begin_src shell
sudo -l
#+end_src

** show login sessions
#+begin_src shell
w
#+end_src

** show login history
#+begin_src shell
last
#+end_src

#+begin_src shell
lastlog
#+end_src


** set password
#+begin_src bash
passwd miao
#+end_src


** ssh pem

- 生成密钥
#+BEGIN_SRC bash
sudo su miao
cd ~/.ssh
ssh-keygen -t ed25519 -C "miao@mail.com"
#+END_SRC

- 修改权限
#+BEGIN_SRC bash
cp ed25519.pub authorized_keys
chmod 600 authorized_keys
chmod 700 ~/.ssh
#+END_SRC

- 拷贝密钥
#+BEGIN_SRC bash
scp root@10.105.250.92:/home/miao/.ssh/ed25519 ~/.ssh/
#+END_SRC

** remove user
#+begin_src bash
userdel -r username
#+end_src

** change shell
#+begin_src shell
chsh -s /usr/bin/zsh
#+end_src


* ssh

** sshd

#+begin_src conf
Match User yayu
      X11Forwarding yes
      X11UseLocalhost no
      AllowTcpForwarding yes
#+end_src

** ssh client
#+begin_src conf
 Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/yayu
#+end_src

** execute command
#+begin_src shell
ssh leapx16 -t 'cd /home/yayu/ && pwd'
#+end_src

* Xorg

** names

*** xprop

*** wmctrl
#+begin_src sh
wmctrl -x -l
#+end_src

** touchpad

*** change scroll speed
#+begin_src conf
Section "InputClass"
        Identifier "Enable natural scrolling by default"
        MatchIsTouchpad "on"
        MatchDevicePath "/dev/input/event*"
        MatchDriver "libinput"
        Option "Natural Scrolling" "on"
        Option "ScrollPixelDistance" "25"
EndSection
#+end_src

* Archlinux

** pacman

- refresh pgp =sudo pacman-key --refresh-keys=

*** auto remove
+ sudo pacman -R $(pacman -Qdtq)

*** install deb
#+begin_src shell
debtap *.deb
sudo pacman -U *.zst
#+end_src

*** install image
#+begin_src shell
chmod +x ./app.AppImage
./app.AppImage
#+end_src

** makepkg

*** install
- ~-s, --syncdeps~ install dependencies
- ~-i, --install~ install package

#+begin_src sh
makepkg -si
#+end_src

** yay
| option               | description                                                 |
| -S                   | install                                                     |
| -Ss                  | search the package                                          |
| --mflags --skipinteg | skip validaty check                                         |
| -R                   | remove                                                      |
| -Rs                  | Remove dependencies not required by other packages          |
| -Sc                  | remove all cache                                            |
| --overwrite <glob>   | Bypass file conflict checks and overwrite conflicting files |
| --editmenu           | edit pkgbuild before install                                |
| -Ql                  | show package contents                                       |
| -Qo                  | show file owner                                             |
| -Qi                  | show package info                                           |


* Bluetooth
** install
#+begin_src sh
yay -S pipewire pipewire-pulse pipewire-alsa pipewire-audio wireplumber bluez bluez-utils
#+end_src


** pair
#+begin_example
$ sudo systemctl start bluetooth.service
$ bluetoothctl

# power on
# default-agent
# scan on
[NEW] Device 00:12:34:56:78:90 Air Pods
# pair 00:12:34:56:78:90
# connect 00:12:34:56:78:90
#+end_example

** airpods
#+begin_src conf
# /etc/bluetooth/main.conf
ControllerMode = bredr
#+end_src

* module

** show loaded
#+begin_src sh
lsmod
#+end_src

** install/load
#+begin_src shell
modprobe <module>
#+end_src


** remove
#+begin_src shell
modprobe -r <module>
rmmod <module>
#+end_src

* nouveau

** blacklist
#+begin_src conf
# /etc/modprobe.d/nvidia.conf
blacklist nouveau
#+end_src

** Remove kms from the HOOKS array in /etc/mkinitcpio.conf

** regenerate the initramfs

#+begin_src shell
mkinitcpio -p linux
#+end_src

* keyboard layout

** xmodmap
*** caps_lock -> control
#+begin_src conf
# ~/.Xmodmap
clear lock
clear control
keycode 66 = Control_L
add control = Control_L Control_R
#+end_src
*** setxkbmap -option caps:ctrl_modifier
#+begin_src shell
setxkbmap -option caps:ctrl_modifier
#+end_src
* gpg
** migrate

*** list keys
#+begin_src sh
gpg --list-secret-keys --keyid-format LONG

sec   rsa4096/[your key] 2018-03-30 [SC]
      ABCDEFGHIJKLMNOPQRSTUVWXYZ
uid                 [ unknown] wyy (KEY NAME) <user@domain>
ssb   rsa4096/ABCDEFGHIJKL 2018-03-30 [E]
#+end_src

*** export
#+begin_src
gpg --export -a [your key] > gpg-pub.asc
gpg --export-secret-keys -a [your key] > gpg-sc.asc
#+end_src

*** import
#+begin_src sh
gpg --import gpg-pub.asc
gpg --import gpg-sc.asc
gpg --edit-key [your key]
trust [your key]
#+end_src

*** restart gpg-agent
#+begin_src sh
sudo gpgconf --kill gpg-agent
#+end_src

*** restart
* disk
** format usb

#+begin_src sh
sudo fdisk /dev/sda

m # menu
p # print partition
d # delete partition
g # GPT partition table
n # new partition

sudo mkfs.ext4 -L Kingston -m 1 -b 4096 /dev/sda1
sudo mkfs.fat -F 32 -n Kingston /dev/sda1
sudo mkfs.exfat -n Kingston /dev/sda1

sudo dd if=/dev/zero of=/dev/sda1 bs=1M status=progress
#+end_src

** install ISO
- ~dd if=/archlinux-x86_64.iso of=/dev/sda bs=4M status=progress && sync~

*** windows iso
- ventoy

* swap file
#+begin_src sh
sudo fallocate -l 1G /swap
sudo dd if=/dev/zero of=/swap bs=1024 count=1048576
sudo chmod 600 /swap
sudo mkswap /swap
sudo swapon /swap
sudo swapon --show
genfstab / > /etc/fstab
#+end_src

* misc
** disable tab bell
#+begin_src sh
set bell-style none
#+end_src

** open terminal here
#+begin_src sh
gsettings set org.cinnamon.desktop.default-applications.terminal exec kitty
#+end_src

* font

** utf-8 & locale

*** check
#+begin_src shell
locale
#+end_src

*** set
#+begin_src shell
# /etc/locale.conf

LANG=en_US.UTF-8
#+end_src

or
#+begin_src shell
localectl set-locale LANG=en_US.UTF-8
#+end_src

** show fonts
#+begin_src sh
fc-list
#+end_src

* systemd daemon

** user
#+begin_src conf
# ~/.config/systemd/user/xremap.service
[Unit]
Description=Xremap Daemon

[Service]
ExecStart=xremap /home/yayu/.config/xremap/config.yml

[Install]
WantedBy=default.target
#+end_src

** system
#+begin_src conf
# /etc/systemd/systemd/kmonad.service
[Unit]
Description=Kmonad Daemon

[Service]
ExecStart=/home/yayu/.local/bin/kmonad /home/yayu/.config/kmonad/kmonad.kbd

[Install]
WantedBy=default.target
#+end_src

** journalctl

*** kernel log
#+begin_src sh
journalctl -k
#+end_src

*** per service log
#+begin_src sh
journalctl -u httpd
#+end_src

*** recent first / reverse
#+begin_src sh
journalctl -r
#+end_src


* CRLF

** CR - *Carriage Return*
- \r, 0x0D in hexadecimal, 13 in decimal
- moves the cursor to the beginning of the line without advancing to the next line
** LF - *Line Feed*
- \n, 0x0A in hexadecimal, 10 in decimal
- moves the cursor down to the next line without returning to the beginning of the line

* Trouble Shooting
** Read-only file system
1. check permission ~mount | grep ' / '~
2. remount ~mount -o remount,rw /~

