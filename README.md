# TODOs
- does watchtower this need to go into the media-stack to be effective?
- configure apps programatically
- Terraform and ansible?

# Guide Usage
Search for `update` -- apart from the container named `ddns-u***ter`, every instace of 'that word' should jump you to a place you need to modify for your network.

# Requirements
This setup assumes the NAS and VM are on a separate VLAN that only communicates externally via a VPN. If that isn't your setup, _don't use this guide_.

### Router
Configure a VLAN and route to send all traffic from that VLAN out a VPN-bound gateway. If that doesn't make sence, _don't use this guide_.


### NAS
Add a _dedicated_ network interface to the NAS for the VPN VLAN. If your NAS does not have multiple NICs or you don't know how to configure this, _don't use this guide_ - unless you are using a dedicated NAS, in which case it can live only on this VLAN.


### arrs VM
Ensure the IP received is associated with the VPN-connected VLAN - `curl ifconfig.io` validate you are not leaking your personal IP from this VM. If you are having trouble getting the VM to solidly stay on the VPN, _don't use this guide_.

# Initialization
### arrs VM
1. Create `arrs.magic` VM with a static IP from the VPN VLAN range. If you name it something else, you'll need to make modification throughout this guide.
1. Create `arrs` user, add to sudoers group.

### NAS
1. Create a new username and password for `arrs.magic` `arrs` user to access share. If no share (SMB/NFS) exists, create that too.

### Router
1. In your router, configure static IP and local DNS for `arrs.magic` VM. This setup assumes a VM named `arrs.magic`. Ensure `arrs.magic` is configured to only connect to the VPN VLAN.



## Add network shares to VM

You'll want to store your media and some configuration data separately from the media-stack applications and host.


1. Create credential file readable only by root user that contains the `NAS share` username and password corresponding to the `arrs` user (configured previously)

```sh
echo username=SOMEUSER\npassword=SOMEPASSWORD | sudo tee -a /etc/smb_credential_media
chmod 600 /etc/smb_credential_media
```

1. Setup NAS share to mount on startup

  __1. Update `SMB_PATH`.__
  
  `SMB_PATH` is your NAS IP/hostname and share path, configure it for your network.
  
  `FOLDER_FOR_MEDIA` is the location on `arrs.magic` to which the SMB share is mounted. This does not need to change. _If you do change it, you must modify `FOLDER_FOR_MEDIA` in `media-stack.env`, as well._

```sh
export SMB_PATH="//tnas.arrs/media"
export FOLDER_FOR_MEDIA="/mnt/tnas_media"

echo $SMB_PATH $FOLDER_FOR_MEDIA cifs credentials=/etc/smb_credential_media,uid=$(id -u),gid=$(id -g),iocharset=utf8 0 0 | sudo tee -a /etc/fstab
```

_Ensure the uid and gid fields are not `0`, as this would indicate you ran the command with the `root` user. You must run the command with `arrs` user._

## Create application and media folders:

You should not need to modify anything in this script block. You may modify the categories and app types, particularly to add categories and apps. See not in following section regarding `FOLDER_FOR_CONFIGS` before changing it.

```sh
export FOLDER_FOR_CONFIGS=/opt/media-stack
export FOLDER_FOR_ARCHIVE=$FOLDER_FOR_MEDIA/apps/docker

export PUID=$(id -u)
export PGID=$(id -g)


sudo -E mkdir -p $FOLDER_FOR_ARCHIVE
sudo -E mkdir -p $FOLDER_FOR_CONFIGS/{authelia/assets,bazarr,ddns-updater,heimdall,homarr/{configs,data,icons},homepage,jellyfin,jellyseerr,lidarr,mylar,portainer,prowlarr,qbittorrent,radarr,sabnzbd,sonarr,tdarr/{server,configs,logs},tdarr_transcode_cache}
sudo -E mkdir -p $FOLDER_FOR_MEDIA/{anime,audio,book,comic,movie,music,photos,tv}
sudo -E mkdir -p $FOLDER_FOR_MEDIA/usenet/{anime,audio,book,comics,complete,console,incomplete,movies,music,prowlarr,software,tv}
sudo -E mkdir -p $FOLDER_FOR_MEDIA/torrents/{anime,audio,books,comics,complete,console,incomplete,movies,music,prowlarr,software,tv}
sudo -E mkdir -p $FOLDER_FOR_MEDIA/watch
sudo -E mkdir -p $FOLDER_FOR_MEDIA/filebot/{input,output}
sudo -E chmod -R 775 $FOLDER_FOR_MEDIA $FOLDER_FOR_CONFIGS
sudo -E chown -R $PUID:$PGID $FOLDER_FOR_MEDIA $FOLDER_FOR_CONFIGS
```


### Create rsync cron job to move configs to NAS

The applications will corrupt their sqlite databases used for configuration if `FOLDER_FOR_CONFIGS` is non-local. To avoid this, we run with a local data folder, but backup the local config shares to the NAS on a regular schedule.  (Sundays at 3 am, maintain ownership/permissions, zip up the folder). _If you change `FOLDER_FOR_CONFIGS`, you must change `FOLDER_FOR_CONFIGS` in `media-stack.env`._

```sh
echo "0 3 * * 0 root rsync -az $FOLDER_FOR_CONFIGS $FOLDER_FOR_ARCHIVE" | sudo tee -a /etc/cron.d/rsync_media-stack_to_nas
```

Of cource, you can manually create a backup by running the `rsync` command, which you should do following deployment, configuration of the `media-stack`.

# Configure Stacks
1. Install docker
1. Add portainer

Portainer will be used to manage all other containers, so this is the only `docker` command we need to run.
```sh
docker run -d \
    --name portainer \
    --restart=unless-stopped \
    -p 8000:8000 \
    -p 9443:9443 \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v $FOLDER_FOR_CONFIGS/portainer:/data \
    portainer/portainer-ce:latest
```
From this point on, the apps are deployed from the __[Portainer GUI](`https://arrs.magic:9443`)__.

## Add watchtower stack to Portainer.
Create a new Stack in Portainer. Watchtower will keep your other containers up to date by periodically (see WATCHTOWER_SCHEDULE) checking if newer images are available, and redeploying them, if so.

```yaml
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
```

## Create media-stack
Create a new Stack in Portainer where all the capability apps will run.

1. Name: media-stack
1. Build methodSelect Upload: select docker-compose.yml
1. Environment variables: upload media-stack.env
1. `Deploy the stack`
1. Once deployed, the initial admin password for each app from the container logs.

### Configure [Jackett](http://arrs.magic:9117)
1. Note the api key in the top right, this will be used in following steps
1. Admin password
1. FlareSolverr API Url: http://arrs.magic:8191 # This is for CAPTCHA bypass.
1. Add indexer: bring your own private/semiprivate, or try a public one, for example 1337x.
  Adding the indexer will Test connectivity, including CAPTCHA bypass.
1. See instructions provided in Jackett UI for how to add indexers to other apps -- this guide uses Prowlarr instead of Sonarr/Radarr, but instructions are the same.

### Configure [qBittorrent](http://arrs.magic:8200)
Based on https://mediastack.guide/config/qbittorrent

Remember to `Save` before leaving options.

1. Options/Behavior: Check "Show external IP in status bar:
1. Options/Downloads: Management Mode Auto, Default Paths: /data/torrents/{complete,incomplete}, monitor /data/watch
1. Options/WebUI/Authentication
1. Bypass for 172.18.0.0/16, 192.168.6.0/28

__Update__ the bypass network to your `media-stack_default` (observed in Portainer → Networks) and VPN VLAN, respectively.
1. Options/BitTorrent:Seeding check when ratio reaches 1 then Stop torrent
1. Open the Search Tab (top right button) → Search plugins...(bottom right) → Install new plugin: Path=https://raw.githubusercontent.com/qbittorrent/search-plugins/master/nova3/engines/jackett.py
1. Configure Jackett plugin with Portainer shell in qbittorrent container.
   1. api_key= (from Jackett web ui), url= "http://arrs.magic:9117"
API key also available at `cat /config/jackett/Jackett/ServerConfig.json | jq .APIKey`
```sh
vi /config/qBittorrent/nova3/engines/jackett.json
```


1. Create Categories with with Portainer shell in qbittorrent container
```sh
cat << EOF > /config/qBittorrent/categories.json
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
    "console": {
        "save_path": "/data/torrents/console"
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

### Configure [Sabnzbd](http://arrs.magic:8100)
Based on https://mediastack.guide/config/sabnzbd/

Be sure to hit `Save Changes`
1. General → Security
   1. username:password
   1. External internal access = Full API
   1. API Key - note this, it will be given to other apps for configuration.
1. Folders
   1. Temporary Folder = /data/usenet/incomplete
   1. Completed Folder = /data/usenet/complete
   1. Watched Folder = /data/watch
1. Server:
   1. Add your usenet __Providers__ - these are typically paid for (<$20/yr)
1. Categories
   1. For each of the categories (i.e. anime, movies, etc.), create a row with `Category={CAT_NAME}` and `folderPath=/data/usenet/{CAT_NAME}`
1. RSS - add if you have RSS feeds.
1. Modify /config/sabnzbd.ini using Portainer shell on sabnzbd container,
```ini
host_whitelist = localhost, dockerhost, arrs.magic
local_ranges = 172.18.0.0/16, 192.168.6.0/28
```
__Update__ `local_ranges` to match your `media-stack_default` and VPN VLAN, respectively.


1. Configure a usenet server: bring your own provider 
1. Configure Categories- for each category in anime,audio,books,comics,console,movies,music,prowlarr,software,tv
   1. Category: {category}
   1. FolderPath: /data/usenet/{category}
   1. Save


### Configure [Prowlarr](http://arrs.magic:9696)

Prowlarr and Jackett do much of the same thing. They are both included because why not. If you have a preference, you can remove the service from the stack.

1. Settings → Indexers (Indexer Proxies) → Add
   1. Name: FlareSolverr (aka ByParr), Tags: flaresolverr, Host: http://arrs.magic:8191/
   1. Hit test to see if ByParr is able to crack the CAPTCHA.
   1. Add the Indexers from Jackett following instructions on [Jacket web ui](http://arrs.magic:9117), add the `flaresolverr` tag.
   1. Add additional indexers
1. Settings → Download Clients
   1. qBittorrent- host: arrs.magic, port: 8200, username/password (per qBittorrent Options/WebUI/Authentication), add mapped categories per: [this guide](https://mediastack.guide/config/prowlarr/#add-torrent-downloader) (note anime and comics are the standouts)
   1. Sabnzbd - host: arrs.magic, port: 8100, api key & username/password (per Sabnzbd General → Security), add mapped categories as above.
1. Apps → Add "commmon" apps (Radarr, Sonarr, Lidarr, Readarr-aka Bookshelf)
   1. Prowlarr Server: http://arrs.magic:9696
   1. API found at /config/config.xml of corresponding container
   1. Server: all use same host (http://arrs.magic, with respective ports: 7878, 8989, 8686, 8787)
1. Apps → Add Mylar
   1. Open [Mylar](http://arrs.magic:8090/config) then Web Interface → Tick "Enable API" → Generate Mylar API Key
   1. Optionally (recommended) create a free account at [ComicVine](https://comicvine.gamespot.com/api/) and grab an API Key to add to this page.
   1. Save Changes
   1. Search Providers → Tick "Use Newznab" → Tick "Torrents" → Tick "Enable Torznab" → Save Changes
   1. Restart Mylar (top of page)
   1. Add Mylar just like the other apps.
1. Apps → Test All Apps
1. Apps → Sync App Indexers


## Configure ARRs servers
### Add download clients
For some reason, the download clients _are not_ synced from prowlarr to the arr servers...
1. [Lidarr](http://arrs.magic:8686), [Radarr](http://arrs.magic:7878), [Readarr](http://arrs.magic:8787), [Sonarr](http://arrs.magic:8989)
   1. Copy steps for adding download clients to Prowlarr. Pay attention to the default Category each app provides, they should be: music, movies, books, tv; respectively.
1. [Mylar](http://arrs.magic:8090)
   1. Settings → Download settings → Usenet
      1. Sabnzbd selected
      1. Sabnzbd Host: http://arrs.magic:8100
      1. Sabnzbd: user/password/API
      1. Sanbzbd Category: comics
      1. Are Mylar / SABnzbd on separate machines: true
      1. Sabnzbd Download Directory: /data/usenet/comics
      1. Test SABnzbd
   1. Settings → Download settings → Torrents
      1. Use Torrents: true
      1. qBittorrent
      1. qBittorrent Host:Port : http://arrs.magic:8200
      1. username/password
      1. qBittorrent Label: comics
      1. qBittorrent Folder: /data/torrents/comics
      1. Test Connection
   1. Save Changes
   1. Restart (top of page)

## App-specific Configuration

### Configure [Radarr](http://arrs.magic:7878)
1. Settings → Metadata → Kodi (XBMC): Enable, Movie Metadata, Movie Images, Save
1. Settings → Metadata → Roksbox: Enable, Movie Metadata, Movie Images, Save
1. Settings → Metadata → WDTV: Enable, Movie Metadata, Movie Images, Save
1. Settings → Media Management (formats per [this guide](https://jellyfin.org/docs/general/server/media/movies/))
   1. Show Advanced (top)
   1. Rename Movies: true
   1. Replace Illegal Characters: true
   1. Colon Replacement:	Replace with Dash
   1. Standard Movie Format:	`{Movie CleanTitle} {(Release Year)} {imdbid-{ImdbId}} - {edition-{Edition Tags}} {[Custom Formats]}{[Quality Full]}{[MediaInfo 3D]}{[MediaInfo VideoDynamicRangeType]}{[Mediainfo AudioCodec}{ Mediainfo AudioChannels}]{MediaInfo AudioLanguages}[{Mediainfo VideoCodec}]{-Release Group}`
   1. Movie Folder Format:	`{Movie CleanTitle} {(Release Year)} - [imdbid-{ImdbId}]`
   1. Delete empty folders: true
   1. Save Changes (top)

Time to import existing movies...
1. Movies → Import Existing Movies → Start Import: `/data/movie`

### Configure [Sonarr](https://arrs.magic:8989)

[ ] Setup Dispatcharr https://dispatcharr.github.io/Dispatcharr-Docs/
[] Setup Jellyfin
[] Setup Jellyseer



