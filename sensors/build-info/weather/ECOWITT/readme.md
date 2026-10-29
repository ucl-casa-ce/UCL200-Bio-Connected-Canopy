# ECOWITT WITTBOY GW2000

- 94-3C-C6-92-A2-AF
- UCL200-ecowitt
- UCL-Device-IOT 


note: couldn’t connect to UCL_IoT - think this was an issue with 2.4GHz vs 5GHz network - ended up using a direct network cable link from GW to Pi

Set a static IP for eth0 on a dedicated isolated subnet (e.g. 192.168.50.1)

`sudo nmcli con add type ethernet con-name gw2000-direct ifname eth0 ip4 192.168.50.1/24`

Because the GW2000 defaults to Auto-IP/Link-Local (or DHCP) when plugged in without a router, it will either assign itself a 169.254.x.x address or fall back to its internal default IP (192.168.4.1 or 192.168.1.1). To quickly find the GW2000's IP over the direct cable, run nmap or arp-scan on the Pi:

`sudo apt install nmap -y`

Scan the local link

`sudo nmap -sn 192.168.50.0/24`

The only host appearing in that scan (192.168.50.1) is the Pi 5 itself. This happens because the GW2000 does **not** run its own DHCP server—it expects a router to give it an IP address. When connected directly to a cable without a DHCP server, the GW2000 falls back to an **APIPA / Link-Local address in the** 169.254.x.x **range**. Here is the quickest way to give the GW2000 an IP address on eth0 using a lightweight DHCP server on your Pi.

`sudo apt update && sudo apt install dnsmasq -y`

Create a small configuration file specifically for the direct Ethernet link:

`sudo nano /etc/dnsmasq.d/gw2000.conf`

with

```
# Only listen on the Ethernet port (leave wlan0 / UCL_IoT untouched)
interface=eth0
bind-interfaces

# Hand out IPs between 192.168.50.10 and 192.168.50.50
dhcp-range=192.168.50.10,192.168.50.50,255.255.255.0,12h
```

restart service

`sudo systemctl restart dnsmasq`

Unplug the USB power cable from the GW2000, wait 5 seconds, and plug it back in.

Watch the Ethernet port LEDs on the Pi—they should start blinking as the GW2000 boots up and requests an IP address over DHCP.

Run the Nmap scan again:

`sudo nmap -sn 192.168.50.0/24`

ip_address = 192.168.50.42

## The weewx-gw1000 Extension
This uses Ecowitt’s local network API to **pull** data directly from the GW2000 hub over Wi-Fi/Ethernet. It requires zero cloud reliance and doesn't depend on HTTP POST callbacks.

## Setup Steps

**1 Get the Hub's IP:** 
Make sure the GW2000 gateway is connected to your local network (via Wi-Fi or its built-in RJ45 Ethernet port). Find its IP address in your router or via the Ecowitt app.

**2** **Install the Extension:** 
On your Pi, run the weectl extension installer:I had to update to:

`sudo apt install python3-six -y`

then

`sudo weectl extension install https://github.com/weewx-contrib/weewx-gw1000/archive/refs/heads/master.zip`

**3 Configure WeeWX:** 
Reconfigure WeeWX to use the new driver - 

`sudo weectl station reconfigure --driver=user.gw1000`

Check it in weewx.conf

`sudo nano /etc/weewx/weewx.conf`

I also updated topic to the ecowitt (not dvp2) and polling is set to 20

```
[Station]
    station_type = GW1000
[GW1000]
    driver = user.gw1000
    ip_address = 192.168.1.XXX   # Replace with your GW2000 IP address
    poll_interval = 10           # Poll rate in seconds
```

Finally

`sudo systemctl restart weewx`

### Wittboy-Specific Nuances to Keep in Mind

* **Haptic Rain Sensor (WS90):** The Wittboy uses a piezo/haptic sensor for rain detection rather than a tipping bucket. It sends rain rate and accumulation data straight over the Ecowitt protocol, which WeeWX maps automatically.

* **Piezo Calibration:** If you want ultra-precise rain metrics, Ecowitt allows you to calibrate the haptic sensor output directly within the Ecowitt app before the data is passed down to WeeWX.

* **Dual Run Potential:** Because the GW2000 streams via IP, you can have WeeWX pull data locally *and*still allow the gateway to report to Ecowitt.net, WOW, or Weather Underground simultaneously without conflicting.
