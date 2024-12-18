# Lynchbros Media Services

## QNAP Deployment Notes

- Docker requires custom `qnet` network driver to assign static IPs
- `macvlan` is not supported or we just don't know how to make it work
  - Could be containers need permission to network stack
- App Configs should be host mounts to: `/share/CACHEDEV1_DATA/Container/{App_Name}`
- Media is located at: `/share/CACHEDEV2_DATA/Media/media`
- For HW Transcoding support, `PGID` should be `0` (root) and then the following code should be added

  ```bash
  devices:
    - /dev/dri:/dev/dri
  ```

### Example Docker file

```yaml
services:
  plex:
    image: lscr.io/linuxserver/plex:latest
    container_name: Plex
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=0
      - TZ=America/New_York
      - VERSION=latest
    volumes:
      - /share/CACHEDEV1_DATA/Container/Plex:/config
      - /share/CACHEDEV2_DATA/Media/media:/data
    tmpfs: /transcode
    networks:
      host_bridge:
        ipv4_address: 192.168.60.10
    devices:
      - /dev/dri:/dev/dri

networks:
  host_bridge:
    driver: qnet
    driver_opts:
      iface: "eth0"
    ipam:
      driver: qnet
      options:
        iface: "eth0"
      config:
        - subnet: 192.168.60.0/24
          gateway: 192.168.60.1

```

## TrueNas Deployment Notes

- Before deploying any apps, we need to create our Docker networks
  - SSH into the TrueNas server and run `sudo -s`
  - Create a public facing `macvlan` network that matches your actual network configuration. (Note: the name of the network is the last value)

    ```bash
    docker network create --driver macvlan \
      --subnet=192.168.50.0/24 \
      --gateway=192.168.50.1 \
      -o parent=enp2s0 \
      external
    ```

  - Create a private facing `bridge` network for our apps to communicate privately over

    ```bash
    docker network create \
      --driver=bridge \
      --subnet=172.28.0.0/16 \
      --ip-range=172.28.0.0/24 \
      --gateway=172.28.0.1 \
      internal
    ```

- Now we can log into TrueNas click on `Discover Apps` then the ... button and select `Install via YAML`
- App Configs should be host mounts to: `/mnt/zPool2/data/app_configs/{App_Name}`
- Media is located at: `/mnt/zPool2/data/media`
- For HW Transcoding support, `PGID` should be `0` (root) and then the following code should be added

  ```bash
  devices:
    - /dev/dri:/dev/dri
  ```

### Example Docker file

```yaml
networks:
  macvlan_net:
    external: True
services:
  app:
    devices:
      - /dev/dri:/dev/dri
    environment:
      PUID: 0
      PGID: 0
      TZ: America/New_York
      VERSION: latest
    image: linuxserver/plex:latest
    networks:
      macvlan_net:
        ipv4_address: 192.168.50.5
    restart: unless-stopped
    volumes:
      - /mnt/zPool2/data/app_configs/plex:/config
      - /mnt/zPool2/data/media:/data
      - target: /transcode
        tmpfs:
          size: 32G
        type: tmpfs

```
