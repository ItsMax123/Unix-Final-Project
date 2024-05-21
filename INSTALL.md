# Installation Guide

## Getting Raspberry PI OS (Server)
1. On a computer, install Raspberry Pi Imager.
2. Plug in a Micro SD Card.
3. Launch the Imager and select raspberry Pi 5, Micro SD Card and Raspberry Pi OS (Server)
4. In the settings, make sure SSH is turned on.
5. Flash, wait for it to finish and then plug the micro SD Card in the raspberry PI.

## Installing docker on the PI
(Here we will be using `your_username` as the user, replace this with the name of your user)

1. Connect to the Pi using SSH and run the following commands:
* `curl -sSL https://get.docker.com | sh`
* `sudo usermod -aG docker your_username`
* `mkdir /dockers`

Now that we have installed docker, we can install all the pieces of software we need.

## Installing Jellyfin
Run the following commands:
* `mkdir /dockers/jf`
* `cd /dockers/jf`
* `nano docker-composer.yml`

Inside the docker-composer.yml file, paste the following:
```yaml
services:
  jellyfin:
  image: jellyfin/jellyfin:latest
  container_name: jellyfin
  network_mode: 'host'
  volumes:
  - /dockers/jf/config:/config
  - /dockers/jf/cache:/cache
  - /dockers/jf/media:/media
  restart: 'unless-stopped'
  extra_hosts:
  - 'host.docker.internal:host-gateway'
```

* `docker compose up -d`

Now Jellyfin is running on http://{YOUR_LOCAL_IP}:8096/ which you can find using `ip addr`.

## Installing Cloudflare-DDNS (Optional)
If you own a domain, you can use it to connect to the VPN instead of an IP which may change when your router restarts.
To do this, run the following commands:
* `mkdir /dockers/cf`
* `cd /dockers/cf`
* `nano docker-compose.yml`

Inside the docker-composer.yml file, paste the following: **(Replace {YOUR_API_KEY} with the API key which can be found on the cloudflare dashboard and replace {YOUR_DOMAIN_NAME} with your domain name)**
```yaml
services:
  cloudflare-ddns:
    image: oznu/cloudflare-ddns:latest
    restart: always
    environment:
      - API_KEY={YOUR_API_KEY_HERE}
      - ZONE={YOUR_DOMAIN_HERE}
      - SUBDOMAIN=vpn
      - PROXIED=false
```
* `docker compose up -d`

Now, for the next step you can use `vpn.{YOUR_DOMAIN_HERE}` instead of your IP in the docker-composer.yml file.

## Installing WireGuard
Run the following commands:
* `mkdir /dockers/wg`
* `cd /dockers/wg`
* `nano docker-composer.yml`

Inside the docker-composer.yml file, paste the following: **(Replace {YOUR_LOCAL_IP} with your local IP or your domain name if you set up Cloudflare-DDNS!)**
```yaml
services:
  wg-easy:
    environment:
      - LANG=en
      - WG_HOST={YOUR_LOCAL_IP}
      - WG_PORT=321
      - WG_DEFAULT_ADDRESS=10.5.0.x
      - WG_DEFAULT_DNS=1.1.1.1
      - WG_ALLOWED_IPS=0.0.0.0/0
      - WG_PERSISTENT_KEEPALIVE=25
      - UI_TRAFFIC_STATS=true
    image: ghcr.io/wg-easy/wg-easy:latest
    container_name: wg-easy
    volumes:
      - /dockers/wg:/etc/wireguard
    ports:
      - "321:51820/udp"
      - "51821:51821/tcp"
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
```
* `docker compose up -d`

Now that WireGuard is running, you can go to http://{YOUR_LOCAL_IP}:51821 to add accounts.
After adding an account, a QR code is created which can be used to connect to the VPN.
Alternatively, you can download a file which can also be used to connect (This method is easier for computers).
To connect to the VPN, install the Wireguard application on any device(s) and scan the QR code or use the file.

## Installing Samba
If you want to simplify sending media files to your raspberry pi, samba lets you access your raspberry pi files on the network.

* `chmod 777 /dockers/jf/media`
* `sudo apt install samba`
* `sudo nano /etc/samba/smb.conf`

At the very end of the smb.conf file, paste the following:
```yaml
[pi]
  path=/dockers/jf/media
  browsable=yes
  writable=yes
  Create mask=0777
  Directory mask=0777
```
* `sudo smbpasswd -a pi`

Type a new samba password that will be used to access the files over the network.
* `sudo apt install wsdd`
* `sudo service wsdd start`

Then you need to connect to it on another device.
For this example we will be using a Windows machine.
In a command prompt, run the following command:

* `net use Z: \\{YOUR_LOCAL_IP}\pi /user:pi {YOUR_SAMBA_PASSWORD} /persistent:yes`

Then you will see the folder appear on your Windows machine, and you can now access everything inside the media folder.

Now that the setup is complete, you can use WireGuard to connect to your local network.
Connected to your local network, you can use samba to add media files to the PI.
Using Jellyfin, you can watch all of your media on a nice interface. You can configure many different features within the jellyfin dashboard, like adding different users just like how netflix works.
