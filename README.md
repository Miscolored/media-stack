Arrs VM setup
[x] Verify VPN
[x] Install qemu agent
[x] Install docker
[x] Create VM template
[x] Deploy new VM
[x] Add network shares to /etc/fstab
[x] Create application and media folders:
```sh
export FOLDER_FOR_MEDIA=/mnt/tnas_media
export FOLDER_FOR_DATA=/opt/media-stack
export FOLDER_FOR_ARCHIVE=/mnt/tnas_media/apps/docker

export PUID=1000
export PGID=1000


sudo -E mkdir -p $FOLDER_FOR_ARCHIVE
sudo -E mkdir -p $FOLDER_FOR_DATA/{authelia/assets,bazarr,ddns-updater,gluetun,heimdall,homarr/{configs,data,icons},homepage,jellyfin,jellyseerr,lidarr,mylar,plex,portainer,prowlarr,qbittorrent,radarr,readarr,sabnzbd,sonarr,swag,tdarr/{server,configs,logs},tdarr_transcode_cache,unpackerr,whisparr}
sudo -E mkdir -p $FOLDER_FOR_MEDIA/media/{anime,audio,books,comics,movies,music,photos,tv,xxx}
sudo -E mkdir -p $FOLDER_FOR_MEDIA/usenet/{anime,audio,books,comics,complete,console,incomplete,movies,music,prowlarr,software,tv,xxx}
sudo -E mkdir -p $FOLDER_FOR_MEDIA/torrents/{anime,audio,books,comics,complete,console,incomplete,movies,music,prowlarr,software,tv,xxx}
sudo -E mkdir -p $FOLDER_FOR_MEDIA/watch
sudo -E mkdir -p $FOLDER_FOR_MEDIA/filebot/{input,output}
sudo -E chmod -R 775 $FOLDER_FOR_MEDIA $FOLDER_FOR_DATA
sudo -E chown -R $PUID:$PGID $FOLDER_FOR_MEDIA $FOLDER_FOR_DATA
```


[x] Create rsync cron job to move configs to nas (compressed, weekly, Sunday, 3 am GMT)
```sh
echo "0 3 * * 0 root rsync -az /opt/media-stack/ /mnt/tnas_media/apps/docker/" >> /etc/cron.d/rsync_media-stack_to_nas
```


[x] Add portainer
```sh
docker run -d \
    --name portainer \
    --restart=always \
    -p 8000:8000 \
    -p 9443:9443 \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v /opt/media-stack/portainer:/data \
    portainer/portainer-ce:latest
```
[x] Set adminiatrator credentials
[x] Add watchtower stack
version: "3.5"
services:
  watchtower:
    image: nickfedor/watchtower:latest
    container_name: watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - WATCHTOWER_CLEANUP=true
      - WATCHTOWER_INCLUDE_RESTARTING=true
      - WATCHTOWER_INCLUDE_STOPPED=true
      - WATCHTOWER_REVIVE_STOPPED=false
      - WATCHTOWER_NO_RESTART=false
      - WATCHTOWER_TIMEOUT=30s
      - WATCHTOWER_SCHEDULE=0 0 4 * * *
      - WATCHTOWER_DEBUG=false
      - TZ=Australia/Brisbane
    network_mode: bridge
    
[x] Create media-stack
[x] Use media-stack.env for environment variables
[x] Use docker-compose.yml for web editor build method

[x] Set credential on each app 

[] Configure qBittorrent
Based on https://mediastack.guide/config/qbittorrent
Options/Downloads: Management Mode Auto, Default Paths: /data/torrents/{complete,incomplete}, monitor /data/watch
Options/WebUI/Authentication username:password
Bypass for both and 172.18.2.0/16 192.168.6.0/28
Options/BitTorrent:Seeding
Create Categories
```sh
cat << EOF > /opt/media/qBittorrent/categories.json
{
    "anime": {
        "save_path": "/data/torrents/anime"
    },
    "audio": {
        "save_path": "/data/torrents/audio"
    },
    "books": {
        "save_path": "/data/torrents/books"
    },
    "comics": {
        "save_path": "/data/torrents/comics"
    },
    "movies": {
        "save_path": "/data/torrents/movies"
    },
    "music": {
        "save_path": "/data/torrents/music"
    },
    "podcasts": {
        "save_path": "/data/torrents/podcasts"
    },
    "prowlarr": {
        "save_path": "/data/torrents/prowlarr"
    },
    "software": {
        "save_path": "/data/torrents/software"
    },
    "tv": {
        "save_path": "/data/torrents/tv"
    }
}
EOF
```

[x] Configure qBittorrent as download client in each app.


[] Setup sabnzbd:8100
Based on https://mediastack.guide/config/sabnzbd/
[x] Update /opt/media-stack/sabnzbd/sabnzbd.ini with following:
```ini
host_whitelist = 4c1e2e8eb1a0, localhost, dockerhost, arrs.magic
local_ranges = 172.18.0.0/24, 192.168.6.0/28, 192.168.0.0/24
```
[x] Configure a usenet server: bring your own provider 
[x] Configure download client in each app

[x] Setup Jackett:9117

[] Setup Prowlarr:9696
[x] Configure Indexer Proxy - Flaresolverr/ByParr

[ ] Setup Dispatcharr https://dispatcharr.github.io/Dispatcharr-Docs/
[] Setup Jellyfin
[] Setup Jellyseer



