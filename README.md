# Raspberry Pi Network Scanner

Based on the documentation found [here](https://pimylifeup.com/raspberry-pi-network-scanner/), but updated to current versions (at the time of writting this document). Also, security settings and nginx config come from [here](https://www.kismetwireless.net/docs/readme/webserver/).

Node-Red config info was obtained from official docs [here](https://nodered.org/docs/user-guide/runtime/securing-node-red) and [this config](https://discourse.nodered.org/t/node-red-server-with-nginx-reverse-proxy-howto-guide/27397) for nginx

## 1. Setup of the raspberry pi

1. Install raspbian
   1. this doc is based on buster 32 bit version
   2. using raspberry-pi imager app
   3. expand fs, update raspi-config, configure locales, hostname, wifi country info...
   4. in my case I changed hostname to *rastreadorpi*, this will later be important when accessing the web interface
2. `sudo apt update && sudo apt upgrade -y`

### 1.a Extremely annoying wlan0 and wlan1 swapping? read here
It seems that on some occasions (always) the RBPI OS is a bitch and does not remember which interface is which, so it swaps them (wlan0 and wlan1), rendering the configuration useless, as it can land on the physical wireless interface of the rbpi, which is not monitor-capable :) 

To avoid this, seems some fiddling with `/lib/udev/rules.d/` is required, but could not find specific file to do the deed, so that is that. I instead decided to take the "official route": albeit a little annoying, configure (via `raspi-config`) predictable interface names. **THAT** worked.

### 1.b Want to completely disable onboard wifi and/or bluetoot?

This will have the added benefit of not having to mess with the static naming of the interface!, and it will prevent system issues if you want to set up an external USB bluetooth interface (it will fail on reboot or whenever you plug the bluetooth dongle before starting the pi).

Then you have to edit `sudo vim /boot/config.txt` to add this at the end:

```conf
# disabling onboard bluetooth
dtoverlay=disable-bt
# disabling onboard wifi
dtoverlay=disable-wifi
```

## 2. Setup wifi dongle and configure as monitor interface

**THIS STEP IS NOW OPTIONAL** as Kismet now enables monitor mode on its own... if your installation doesn't manage to set the interface into monitor mode, then follow this steps:

1. Check the external wifi dongle plugged, can do monitoring: `iw dev`, check the name of the new interface, then inspect with `iw phy phy<number> info` and search for a line containing **\* monitor**
2. `sudo vim /etc/network/interfaces.d/<interface>` this content:
```
allow-hotplug <interface>
iface <interface> inet manual
    pre-up iw phy phy<nubmer> interface add mon1 type monitor
    pre-up iw dev <interface> del
    pre-up ifconfig mon1 up
```
3. Restart the raspberry `sudo shutdown -r now`
4. Verify new mon1 interface is created `ifconfig`

## 3. Installing Kismet

1. Add Kismet repo:
```bash
# add gpg key and repo
wget -O - https://www.Kismetwireless.net/repos/Kismet-release.gpg.key | sudo apt-key add -
echo 'deb https://www.Kismetwireless.net/repos/apt/release/buster buster main' | sudo tee /etc/apt/sources.list.d/Kismet.list

# Update apt
sudo apt udpate
```
1. Install kissmet
   -  During install answer **YES** to the question wether using suid helpers.
```bash
sudo apt install Kismet -y
```
2. Add user pi to Kismet group: `sudo usermod -aG Kismet pi`
3. Log out and in again: `logout`...
   - Optionally check wether the new group has taken effect: `groups`

## 4. Configuring and starting up the Kismet monitoring tool

1. Configure site: `sudo vim /etc/Kismet/Kismet.conf`
2. Find the right place to put the last two lines:
```bash
# See the README for more information how to define sources; sources take the
# form of:
# source=interface:options
#
# For example to capture from a Wi-Fi interface in Linux you could specify:
# source=wlan0
#
# or to specify a custom name,
# source=wlan0:name=ath9k
#
# Sources may be defined in the config file or on the command line via the 
# '-c' option.  Sources may also be defined live via the WebUI.
#
# Kismet does not pre-define any sources, permanent sources can be added here
# or in Kismet_site.conf
source=mon1
source=hci0
```
3. Start Kismet: `Kismet`

### 4.a Additional config

1. Set log prefix: to avoid cluttering users folder
2. Setting time limits for logs (so they don't fill the disk)
3. Not even using the file for logging (it will be wiped out uppon restart)

Set this in kismet_logging.conf

```conf
# Configure a prefix for the log files
log_prefix=/tmp

# Set ephemereal logs to avoid infinite growht
kis_log_alert_timeout=86400
kis_log_device_timeout=86400
kis_log_message_timeout=86400
kis_log_packet_timeout=86400
kis_log_snapshot_timeout=86400

# Wipe out logs upon restart
kis_log_ephemeral_dangerous=true
```




## 5. Set up user and pass trhough Kismet interface
1. Connect to Kismet's web interface through pi's local ip  and port 2501, in my case it is: http://rastreadorpi.local:2501
2. Set user and password
3. Optionally go to hamburger icon->config and configure to you heart's contempt


## 6. Configure Kismet to start on boot

1. Edit the kismet service file `sudo systemctl edit kismet`
2. Introduce the following:
```bash
[Service]
WorkingDirectory=/home/pi
User=pi
Group=kismet
```
3. Enable the service: `sudo systemctl enable kismet`


## 7. Installing node-red

1. Install node-red
2. Configure as a service so that it starts on boot

All of this with the following lines:

```bash
bash <(curl -sL https://raw.githubusercontent.com/node-red/linux-installers/master/deb/update-nodejs-and-nodered)
sudo systemctl enable nodered.service
```

## 8. Securing and centralising everything

1. Change user's password, perhaps config login via public/private keys
2. Install nginx (to proxy like madness)
3. Generate certificates for SSL (HTTPS)
4. Configure Nginx to proxy both kismet and nodered, and use certificates
5. Configuring both kismet and nodered to listen only on 127.0.0.1

```bash
# Installing nginx
sudo apt install nginx -y
# Creating certificate:
openssl req -nodes -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 1000 -subj "/C=ES/ST=MU/L=Murcia/O=OdinS/OU=IoT/CN=rastreadorpi.local"
# Moving certificates to nginx folder
sudo mkdir /etc/nginx/ssl
sudo mv cert.pem key.pem /etc/nginx/ssl
```


edit the `sudo vim /etc/nginx/sites-available/rastreadorpi.local.conf` to this:

```nginx
#proxy for node-red @ port :1880
server {
   listen 443 ssl;
   listen [::]:443 ssl;

   server_name rastreadorpi.local;

   ssl_certificate     /etc/nginx/ssl/cert.pem;
   ssl_certificate_key /etc/nginx/ssl/key.pem;
   ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
   ssl_ciphers         HIGH:!aNULL:!MD5;

   location = /robots.txt {
      add_header  Content-Type  text/plain;
      return 200 "User-agent: *\nDisallow: /\n";
   }

   location / {
         proxy_pass http://127.0.0.1:1880;

      #Defines the HTTP protocol version for proxying
      #by default it it set to 1.0.
      #For Websockets and keepalive connections you need to use the version 1.1    
      proxy_http_version  1.1;

      #Sets conditions under which the response will not be taken from a cache.
      proxy_cache_bypass  $http_upgrade;

      #These header fields are required if your application is using Websockets
      proxy_set_header Upgrade $http_upgrade;

      #These header fields are required if your application is using Websockets    
      proxy_set_header Connection "upgrade";

      #The $host variable in the following order of precedence contains:
      #hostname from the request line, or hostname from the Host request header field
      #or the server name matching a request.    
      proxy_set_header Host $host;

      #Forwards the real visitor remote IP address to the proxied server
      proxy_set_header X-Real-IP $remote_addr;

      #A list containing the IP addresses of every server the client has been proxied through    
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

      #When used inside an HTTPS server block, each HTTP response from the proxied server is rewritten to HTTPS.    
      proxy_set_header X-Forwarded-Proto $scheme;

      #Defines the original host requested by the client.    
      proxy_set_header X-Forwarded-Host $host;

      #Defines the original port requested by the client.    
      proxy_set_header X-Forwarded-Port $server_port;

   }

   location /kismet/ {
      proxy_pass http://localhost:2501;
      proxy_set_header Host $http_host;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Scheme $scheme;
      proxy_set_header X-Proxy-Dir /kismet;
      proxy_http_version 1.1;
      client_max_body_size 0;
   }
}
```

then link the newly created file so that it becomes available: `sudo ln -s /etc/nginx/sites-available/rastreadorpi.local.conf /etc/nginx/sites-enabled/`, then delete the default `sudo rm /etc/nginx/sites-enabled/default`


Configure Kismet to listen only to _localhost_, on the sub-route _kismet_, un-commenting this two settings:

```conf
httpd_uri_prefix=/kismet
httpd_bind_address=127.0.0.1
```

Then restart kismet :) `sudo service kismet restart`

Configure Node-red to listen only to _localhost_, and set admin pwd: first edit `vim /user/pi/.node-red/settings.js` and un-comment this lines:

```js
// this makes it listen only on localhost
uiHost: "127.0.0.1",

// ...
// this enables user authentication
adminAuth: {
        type: "credentials",
        users: [{
            username: "admin",
            password: "$2a$08$0tTwZ4zQu4rXTOBWdmI6sumBzmhb6IM1Nt.qCGcw0v7.LBE4XofM6",
            permissions: "*"
        }]
    },
```
then replace the **password** field with whatever password hash you want. You can use this to get hashed passwords: `node-red admin hash-pw`.

Finally `sudo service nodered restart` away.

## Backup everything for fuck sake

`sudo dd if=/dev/rdisk6 of=rastreadorpi.dmg bs=1m`

check the `rdisk<number>` with whatever it should be using `sudo diskutil list` or something alike.
