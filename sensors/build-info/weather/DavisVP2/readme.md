# Weather station setup 

Build notes and files are on cenotes ([casa ce private github](https://github.com/ucl-casa-ce/cenotes/tree/master/sensors/weather/DVP2))

# Maintenance helpers

`sudo journalctl -u weewx -f`


# Setup Notes
Using weewx and Davis VP2

192.168.1.31

1. RPi5 64 build
2. setup VNC
3. `ip link show wlan0`  then add to network manager:
```bash
sudo nmcli connection add type wifi con-name "UCL_IoT" ifname wlan0 ssid "UCL_IoT"

sudo nmcli connection modify "UCL_IoT" wifi-sec.key-mgmt wpa-psk wifi-sec.psk "xxxxx"
```
and check it had registered:
```bash
nmcli -s -f 802-11-wireless-security.psk connection show "UCL_IoT"
```
4. Ran: `sudo apt update` and `sudo apt upgrade`

installing weewx
followed instructions here:
[Debian - WeeWX 5.2](https://www.weewx.com/docs/5.2/quickstarts/debian/)

`sudo apt install -y wget gnupg`

`wget -qO - https://weewx.com/keys.html | sudo gpg --dearmor --output /etc/apt/trusted.gpg.d/weewx.gpg`

`echo "deb [arch=all] https://weewx.com/apt/python3 buster main" | sudo tee /etc/apt/sources.list.d/weewx.list`

`sudo apt update`

`sudo apt install weewx`

`scp pi@192.168.1.31:/etc/weewx/weewx.conf ~/Desktop/`

Install webserver to show weather pages locally:

`sudo apt install nginx`

`sudo chown -R weewx:www-data /var/www/html/weewx`

`sudo chmod -R 755 /var/www/html/weewx`

`sudo systemctl enable nginx`

Install MQTT [mqtt](https://github.com/weewx/weewx/wiki/mqtt)

download:

`wget -O weewx-mqtt.zip ~https://github.com/matthewwall/weewx-mqtt/archive/master.zip`

`sudo pip3 install paho-mqtt==1.6.1 --break-system-packages`

`sudo weectl extension install weewx-mqtt.zip`

then edit section in weewx.conf

Belchertown Skin setup
[https://github.com/poblabs/weewx-belchertown](https://github.com/poblabs/weewx-belchertown)

follow instructions in GitHub repo:
`wget https://github.com/poblabs/weewx-belchertown/releases/download/weewx-belchertown-1.3.1/weewx-belchertown-release.1.3.1.tar.gz`

`sudo weectl extension install weewx-belchertown-release.1.3.1.tar.gz`

then configure in weewx.conf

needed to do some tweaking of belchertown:

updates to python meant that web generator was breaking - error AttributeError: module 'locale' has no attribute 'format' happens because you are running **Python 3.12+** (standard on Debian Trixie), which finally removed locale.format after years of it being deprecated. It was replaced by locale.format_string - so had to do find and replace in:

`sudo nano /etc/weewx/bin/user/belchertown.py`

Daily graph error
[https://github.com/poblabs/weewx-belchertown/issues/158](https://github.com/poblabs/weewx-belchertown/issues/158) 

added the graphs.conf file from and then ran following to get reporting interbal from console:
`sudo weectl device --info` - was 30 minutes - i need 5 minutes so:

`sudo systemctl stop weewx`

`sudo weectl device --set-interval=5`

`sudo systemctl start weewx`

`sudo weectl report run`