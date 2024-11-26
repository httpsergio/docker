### Docker setup to use qBittorrent over VPN

Containers used:
  - [Gluetun](https://github.com/qdm12/gluetun)
  - [qBittorrent](https://hub.docker.com/r/linuxserver/qbittorrent)
    
VPN Provider: [protonvpn](https://protonvpn.com/) (you can use other providers. See: https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers)
Protocol: Wireguard (it works with OpenVPN too, check Gluetun documentation)

**If you are here, you know why you want to protect your p2p activity.**

###Steps

- First, you need to get a wireguard profile from your ProtonVPN account: https://protonvpn.com/support/wireguard-configurations
(you should select NAT-PMP and VPN Accelerator only, from the options)

- Download the configuration file and open it in a file editor.

### Container
I used a unique docker compose file to deploy this stack.

````
services:
  gluetun:
    image: qmcgaw/gluetun
    container_name: vpn-gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    ports:
      - 9050:9050 #port to access the qbittorrent WebGui
    environment:
      - TZ=Europe/Lisbon
      - VPN_TYPE=wireguard
      - VPN_SERVICE_PROVIDER=custom
      - WIREGUARD_PRIVATE_KEY=your_private_wireguard_key
      - WIREGUARD_ENDPOINT_IP=wireguard_proton_ip
      - WIREGUARD_ENDPOINT_PORT=wireguard_proton_port
      - WIREGUARD_PUBLIC_KEY=your_public_wireguard_key
      - WIREGUARD_ADDRESSES=x.x.x.x/32
      - VPN_DNS_ADDRESS=x.x.x.x
      - VPN_PORT_FORWARDING=on
      - VPN_PORT_FORWARDING_PROVIDER=protonvpn
      - UPDATER_PERIOD=24h 
    volumes:
      - ./gluetun:/gluetun
      - ./temp-vpn-port:/tmp/gluetun/forwarded_port:rw
    restart: no
    healthcheck:
      test: ["CMD", "ip", "a", "show", "tun0"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s


  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: vpn-qbittorrent
    network_mode: "container:vpn-gluetun"  # Sharing network with Gluetun
    depends_on:
      gluetun:
        condition: service_healthy  # Wait until Gluetun is healthy
    environment:
      - TZ=Europe/Lisbon
      - WEBUI_PORT=9050
      - UMASK=022
      - PUID=1000
      - PGID=1000
    volumes:
      - ./qbittorrent:/config
      - ./downloads:/downloads
      - ./temp-vpn-port:/tmp/gluetun/forwarded_port:ro
      - ./entrypoint.sh:/entrypoint.sh  # Mount custom entrypoint script
    entrypoint: ["/bin/bash", "/entrypoint.sh"]  
    restart: no
````
### Explanation

When the stack is depolyed, first the gluetun is created and started and only after that, the qbittorrent container will be created (see the docker compose, line 40).

**Why?**

The gluetun will start to connect to the wireguard profile from your vpn provider and a random port will be assigned to your gluetun. 
This port is random and could be other in the next container/Host restart. Since we need to inform qbittorrent what is the used port so it can connect correctly, we need to first get the port and then pass it to the qbittorrent container. 
This method will allow us to have the stack running withou having to edit the docker compose file with the new port for qbittorrent.

The random port is written by gluetun container to the internal path `/tmp/gluetun/forwarded_port` and that file is mounted to the docker host in the file tmp-vpn-port. Check docker compose file line 26: `- ./temp-vpn-port:/tmp/gluetun/forwarded_port:rw`

Then the temp-vpn-port is mounted to the qbittorrent docker container. Check docker compose file line 52: ` - ./temp-vpn-port:/tmp/gluetun/forwarded_port:ro`

Created a script to pass the tmp-vpn-port to the qbittorrent container variable `TORRENTING_PORT`

Script name: entrypoint.sh
```
#!/bin/bash

# Read the port from the forwarded_port file created by gluetun container
export QB_PORT_FROM_GLUETUN=$(cat /tmp/gluetun/forwarded_port)

# Echo for debugging (optional)
echo "Setting TORRENTING_PORT to: $QB_PORT_FROM_GLUETUN"

# Set the environment variable and start the application
export TORRENTING_PORT=$QB_PORT_FROM_GLUETUN

# Execute the original entrypoint command for qbittorrent
exec /init
````
This script should be located in the same path as the docker-compose.yml file and have execute permissions `chmod +x /path/to/file/entrypoint.sh`

### Note

If you restart your docker host or stop the containers, you should always start the gluetun container first so it writes the random port to the file and then start the qbittorrent container.



