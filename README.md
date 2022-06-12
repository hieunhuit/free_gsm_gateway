# Free GSM Gateway
# Install Asterisk + FreePBX + Chan_dongle
#### Asterisk FreePBX installation have refered from https://vietcalls.com/cai-dat-freepbx-15-va-asterisk-16-tren-debian-10/ - Kien Le
> VMWare ESXI 7.0: Passthrough usb via pci option </br>
> OS:debian 10</br>
> RAM: 4096MB</br>
> CPU: 8vCPU</br>
> Disk: 10GB</br>

> Is that actually work?</br>
> I have installed 3 Sim card and it works fine, sound quality is very good, clear but it's quite a bit of delay (~200 => 300ms)</br>
> If I use android wifi softphone, it will become worst (>500ms)</br>

```sudo apt-get install sudo -y \n
export PATH="$PATH:/sbin:/usr/sbin:usr/local/sbin" \n
adduser <user_name> sudo \n
usermod -aG sudo <user_name> \n
id <user_name> \n
su root \n
nano /etc/sudoer \n
```
> insert to the end of file <user_name> ALL=(ALL:ALL) ALL

# Adjust datetime
```
ln -sf /usr/share/zoneinfo/Asia/Ho_Chi_Minh /etc/localtime
sudo apt update && sudo apt -y upgrade sudo reboot
```

# Dependencies installation
```
sudo apt install git curl wget libnewt-dev libssl-dev libncurses5-dev subversion libsqlite3-dev build-essential libjansson-dev libxml2-dev uuid-dev
```
# Install Asterisk
```
cd /usr/src/
sudo curl -O http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-16-current.tar.gz
sudo tar xvf asterisk-16-current.tar.gz
cd asterisk-16*/
```
### download the mp3 decoder library into the source tree
```
sudo contrib/scripts/get_mp3_source.sh
```
### Ensure all dependencies are resolved
```
sudo contrib/scripts/install_prereq install
```
> ITU-T => 84

# Build and install asterisk
```
sudo ./configure
sudo make menuselect
```
- Add-ons: chan_ooh323, format_mp3
- Core Sound Packages: CORE-SOUNDS-EN-*
- Music On Hold File Packages: MOH-OPSOUND-*
- Extra Sound Packages: EXTRA-SOUNDS-EN-*
- Applications: app_macro
```
sudo make
sudo make install
sudo make progdocs
sudo make samples
sudo make config
sudo ldconfig
```

# Create asterisk user
```
sudo groupadd asterisk
sudo useradd -r -d /var/lib/asterisk -g asterisk asterisk
sudo usermod -aG audio,dialout asterisk
sudo chown -R asterisk.asterisk /etc/asterisk
sudo chown -R asterisk.asterisk /var/{lib,log,spool}/asterisk
sudo chown -R asterisk.asterisk /usr/lib/asterisk
```

# Set default user and group for asterisk
```
sudo nano /etc/default/asterisk
```
> AST_USER="asterisk"
> AST_GROUP="asterisk"
```
sudo nano /etc/asterisk/asterisk.conf
```
> runuser = asterisk ; The user to run as.
> rungroup = asterisk ; The group to run as.

# Start and set config startup asterisk with systemd
```
sudo systemctl restart asterisk
```
### Enable asterisk service to start on system boot
```
sudo systemctl enable asterisk
```
### Test to see if you can connect to Asterisk CLI
```
sudo asterisk -rvvvvv
```
# Install FreePBX
## mysql
```
sudo apt update
sudo apt install mariadb-server mariadb-client
```
### Initial DB setup and set root's password for DB
```
sudo /usr/bin/mysql_secure_installation
```
# Nodejs 10 LTS
```
sudo apt install curl dirmngr apt-transport-https lsb-release ca-certificates
curl -sL https://deb.nodesource.com/setup_10.x | sudo bash
sudo apt update
sudo apt install gcc g++ make
sudo apt install nodejs
```
# Apache2
```
sudo apt install apache2
```
### change Apache user to asterisk and turn on AllowOverride option
```
sudo cp /etc/apache2/apache2.conf /etc/apache2/apache2.conf_orig
sudo sed -i 's/^\(User\|Group\).*/\1 asterisk/' /etc/apache2/apache2.conf
sudo sed -i 's/AllowOverride None/AllowOverride All/' /etc/apache2/apache2.conf
```
### Remove default index.html page
```
sudo rm -f /var/www/html/index.html
```
# Install php and libs
```
sudo apt install wget php php-pear php-cgi php-common php-curl php-mbstring php-gd php-mysql \
php-gettext php-bcmath php-zip php-xml php-imap php-json php-snmp php-fpm libapache2-mod-php

sudo sed -i 's/\(^upload_max_filesize = \).*/\120M/' /etc/php/7.3/apache2/php.ini
sudo sed -i 's/\(^upload_max_filesize = \).*/\120M/' /etc/php/7.3/cli/php.ini
```
# Install FreePBX15
```
sudo apt install wget
cd /usr/src
wget http://mirror.freepbx.org/modules/packages/freepbx/freepbx-15.0-latest.tgz
tar xfz freepbx-15.0-latest.tgz
rm -f freepbx-15.0-latest.tgz
cd freepbx
sudo ./start_asterisk start
sudo ./install -n --dbuser root --dbpass "yourpassword"
```
### Enable Apache Rewrite engine 
```
sudo a2enmod rewrite
sudo systemctl restart apache2
```
# Go on FreePBX Web interface for first's configuration
> Can you do that? :D
# Install chan_dongle
```
cd /usr/src
git clone https://github.com/haha8x/asterisk-chan-dongle-16.git
sudo ./bootstrap
sudo ./configure --with-astversion=<asterisk version>
sudo make
sudo make install
sudo cp etc/dongle.conf /etc/asterisk/dongle.conf
```
# Set permisions for asterisk access to ttyUSB
```
sudo nano /etc/udev/rules.d/30-permisions.rules
```
> KERNEL=="ttyUSB[0-9]*",         MODE="0666",    GROUP="root"
```
sudo nano /et	c/udev/rules.d/92-dongle.rules
```
> ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="12d1", RUN+="/usr/bin/switch_modem %s{idVendor} %s{idProduct}"
```
sudo udevadm control --reload-rules && udevadm trigger
```
> re-check: ls -la /dev/ttyUSB*

# Configure dongle.conf
> context=from-trunk-dongle</br>
> [dongle name]</br>
> imei=</br>
> imsi=</br>
> exten=
```
sudo asterisk -rvvvvv
```
### Load chan_dongle and apply dongle.conf
> module load chan_dongle.so</br>
> dongle reload now


# Configure dialplan for incomming call

### Add into file /etc/asterisk/extensions_custom.conf
> [from-trunk-dongle]</br>
> exten => sms,1,Verbose(Incoming SMS from ${CALLERID(num)} ${BASE64_DECODE(${SMS_BASE64})})</br>
> exten => sms,n,Set(FILE(/var/log/asterisk/sms.txt,,,a)=${STRFTIME(${EPOCH},,%Y-%m-%d %H:%M:%S)} - ${DONGLENAME} - ${CALLERID(num)}: ${BASE64_DECODE(${SMS_BASE64})})</br>
> exten => sms,n,System(echo >> /var/log/asterisk/sms.txt)</br>
> exten => sms,n,Hangup()</br>
> exten => _.,1,Set(CALLERID(name)=${CALLERID(num)})</br>
> exten => _.,n,Goto(from-trunk,${EXTEN},1)</br>

### Adjust file /etc/asterisk/dongle.conf
> [from-trunk-dongle]

### create custom trunk for dongle on freePBX. It call `dongle/<dongle name in dongle.conf>/$OUTNUM$`

