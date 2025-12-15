# _MEDIA-STACK_
A guide and supporting files to make deploying a reliable media stack easy and repeatable. 
---
__Contributing__

Issues/PRs are welcome, but do not think of this repo as "maintained", it's just here for the benefit of those that find it.

---

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
export PUID=$(id -u)
export PGID=$(id -g)

echo $SMB_PATH $FOLDER_FOR_MEDIA cifs credentials=/etc/smb_credential_media,uid=$PUID$,gid=$PGID,iocharset=utf8 0 0 | sudo tee -a /etc/fstab
```

_Ensure the uid and gid fields are not `0`, as this would indicate you ran the command with the `root` user. You must run the command with `arrs` user._

## Create application and media folders:

You should not need to modify anything in this script block. You may modify the categories and app types, particularly to add categories and apps. _If you change `FOLDER_FOR_INCOMPLETE`, you must change `FOLDER_FOR_INCOMPLETE` in `media-stack.env`._ See note in following section regarding `FOLDER_FOR_CONFIGS` before changing it.

```sh
export FOLDER_FOR_CONFIGS=/opt/media-stack
export FOLDER_FOR_ARCHIVE=$FOLDER_FOR_MEDIA/apps/docker
export FOLDER_FOR_INCOMPLETE=/incomplete

sudo -E mkdir -p $FOLDER_FOR_INCOMPLETE/{torrent,usenet}
sudo -E mkdir -p $FOLDER_FOR_ARCHIVE
sudo -E mkdir -p $FOLDER_FOR_CONFIGS/{authelia/assets,bazarr,ddns-updater,heimdall,homarr/{configs,data,icons},homepage,jellyfin,jellyseerr,lidarr,mylar,portainer,prowlarr,qbittorrent,radarr,sabnzbd,sonarr,tdarr/{server,configs,logs},tdarr_transcode_cache}
sudo -E mkdir -p $FOLDER_FOR_MEDIA/{anime,audio,book,comic,movie,music,photos,tv}
sudo -E mkdir -p $FOLDER_FOR_MEDIA/usenet/{anime,audio,book,comics,complete,console,movies,music,prowlarr,software,tv}
sudo -E mkdir -p $FOLDER_FOR_MEDIA/torrents/{anime,audio,books,comics,complete,console,movies,music,prowlarr,software,tv}
sudo -E mkdir -p $FOLDER_FOR_MEDIA/watch
sudo -E mkdir -p $FOLDER_FOR_MEDIA/filebot/{input,output}
sudo -E chmod -R 775 $FOLDER_FOR_MEDIA $FOLDER_FOR_CONFIGS $FOLDER_FOR_INCOMPLETE
sudo -E chown -R $PUID:$PGID $FOLDER_FOR_MEDIA $FOLDER_FOR_CONFIGS $FOLDER_FOR_INCOMPLETE
```


### Create rsync cron job to move configs to NAS

The applications will corrupt their sqlite databases used for configuration if `FOLDER_FOR_CONFIGS` is non-local. To avoid this, we run with a local data folder, but backup the local config shares to the NAS on a regular schedule.  (Sundays at 3 am, maintain ownership/permissions, zip up the folder). _If you change `FOLDER_FOR_CONFIGS`, you must change `FOLDER_FOR_CONFIGS` in `media-stack.env`._

```sh
echo "0 3 * * 0 root rsync -az $FOLDER_FOR_CONFIGS $FOLDER_FOR_ARCHIVE" | sudo tee -a /etc/cron.d/rsync_media-stack_to_nas
```

Of cource, you can manually create a backup by running the `rsync` command, which you should do following deployment, configuration of the `media-stack`.

*** ___If you have to redploy the VM, you will need to manually copy `$FOLDER_FOR_ARCHIVE` back to `$FOLDER_FOR_CONFIGS`___ ***

# Configure LXC for Jellyfin and Seerr
lxc-create --name seerr --template=oci -- --url docker://ghcr.io/seerr-team/seerr:develop

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
1. Environment variables: 
   1. ___UPDATE___ media-stack-sensitive.env
   1. __Take Care not to share the file__ if updating anything in Secrets section.
   1. upload media-stack.env
1. `Deploy the stack`
1. Once deployed, the initial admin password for each app from the container logs.

### Configure [Homarr](http://arrs.magic:7575)
This will make navigation to each of the apps so much easier.
1. Create user and turn off all their analytics and telemetry.
1. Create a board, give it a name
1. Enter edit mode (pencil icon in top right)
1. Add each app
    1. Click __+__ → Add an app → Open app creation
    1. Name (see table)
    1. Icon - will be automatically populated based on app name.
    1. Url (see table)
    1. Create and use

|Name|Url|
|-|-|
|Bazarr|http://arrs.magic:6767|
|ByParr|http://arrs.magic:8191|
|Jackett|http://arrs.magic:9117|
|Lidarr|http://arrs.magic:8686|
|Mylar|http://arrs.magic:8090|
|Portainer|http://arrs.magic:9443|
|Pinchflat|http://arrs.magic:8945|
|Prowlarr|http://arrs.magic:9696|
|Radarr|http://arrs.magic:7878|
|Readerr|http://arrs.magic:8787|
|SABnzbd|http://arrs.magic:8100|
|Sonarr|http://arrs.magic:8989|
|qBittorent|http://arrs.magic:8200|

1. Save changes (clicking pencil icon in top right again)

### Configure [Jackett](http://arrs.magic:9117)
Index helper for other servers.


1. Note the api key in the top right, this will be used in following steps
1. Admin password
1. FlareSolverr API Url: http://arrs.magic:8191 # This is for CAPTCHA bypass.
1. Add indexer: bring your own private/semiprivate, or try a public one, for example 1337x.
  Adding the indexer will Test connectivity, including CAPTCHA bypass.
1. See instructions provided in Jackett UI for how to add indexers to other apps -- this guide uses Prowlarr instead of Sonarr/Radarr, but instructions are the same.

### Configure [qBittorrent](http://arrs.magic:8200)
Torrent download client.

1. Open Portainer shell in qbittorrent container
   1. `mkdir /incomplete && chmod 777 /incomplete`
   1. Create Categories
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
  3. Configure Jackett plugin with Portainer shell in qbittorrent container.
     1. api_key= (from Jackett web ui), url= "http://arrs.magic:9117"
API key also available at `cat /config/jackett/Jackett/ServerConfig.json | jq .APIKey`
```sh
vi /config/qBittorrent/nova3/engines/jackett.json
```
_Remember to `Save` before leaving options._
1. Options → Behavior: Check "Show external IP in status bar:
1. Options → Downloads
   1. Default Torrent Management Mode: Automatic
   1. Default Save Path: Paths: /data/torrents/complete
   1. Keep incomplete torrents in: true, /incomplete
   1. Monitored Folder: /data/watch, Default save location
1. Options → WebUI → Authentication: Bypass for 172.18.0.0/16, 192.168.6.0/28

__Update__ the bypass network to your `media-stack_default` (observed in Portainer → Networks) and VPN VLAN, respectively.

4. Options/BitTorrent:Seeding check when ratio reaches 1 then Stop torrent
1. Open the Search Tab (top right button) → Search plugins...(bottom right) → Install new plugin: Path=https://raw.githubusercontent.com/qbittorrent/search-plugins/master/nova3/engines/jackett.py



### Configure [Sabnzbd](http://arrs.magic:8100)
Usenet download client.

Be sure to hit `Save Changes`
1. General → Security
   1. username:password
   1. External internal access = Full API
   1. API Key - note this, it will be given to other apps for configuration.
1. Open Portainer shell on sabnzbd container,
   1. `mkdir /incomplete && chmod 777 /incomplete`
   1. Modify /config/sabnzbd.ini
```ini
host_whitelist = localhost, dockerhost, arrs.magic
local_ranges = 172.18.0.0/16, 192.168.6.0/28
```
__Update__ `local_ranges` to match your `media-stack_default` and VPN VLAN, respectively.

3. Folders
   1. Temporary Folder = /incomplete
   1. Completed Folder = /data/usenet/complete
   1. Watched Folder = /data/watch
   1. Save Changes
1. Server:
   1. Add your usenet __Providers__ - these are typically paid for (<$20/yr)
1. Categories
   1. For each of the categories (i.e. anime, movies, etc.), create a row with `Category={CAT_NAME}` and `folderPath=/data/usenet/{CAT_NAME}`
1. RSS - add if you have RSS feeds.


1. Configure a usenet server: bring your own provider 
1. Configure Categories- for each category in anime,audio,books,comics,console,movies,music,prowlarr,software,tv
   1. Category: {category}
   1. FolderPath: /data/usenet/{category}
   1. Save

### Configure [Mylar](http://arrs.magic:8090)
1. Settings → Web Interface → API: Optional (recommended) create a free account at [ComicVine](https://comicvine.gamespot.com/api/) and grab an API Key to add to this page.
1. Settings → Web Interface → Tick "Enable API" → Generate Mylar API Key
1. Settings → Web Interface → Comic Location → Comic Location Path: `/data/comic`
1. Settings → Web Interface → Permissions
   1. Enforce Permisions: yes
   1. Directory CHMOD: 0775
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
1. Settings → Search Providers → Tick "Use Newznab" → Tick "Torrents" → Tick "Enable Torznab"
1. Settings → Quality & Post Processing → Post Processing
   1. Enable Post-Processing: true, move
   1. Enable Folder Monitoring: true, `/data/watch`, 5 mins
1. Save Changes


### Configure [Prowlarr](http://arrs.magic:9696)
Index helper for other servers.


Prowlarr and Jackett do much of the same thing. They are both included because why not. If you have a preference, you can remove the service from the stack.

1. Settings → Indexers (Indexer Proxies) → Add
   1. Name: FlareSolverr (aka ByParr), Tags: flaresolverr, Host: http://arrs.magic:8191/
   1. Hit test to see if ByParr is able to crack the CAPTCHA.
   1. Add the Indexers from Jackett following instructions on [Jacket web ui](http://arrs.magic:9117), add the `flaresolverr` tag.
   1. Add additional indexers
1. Settings → Apps → Add Apps, per following table

|Name | Sync Level | Prowlarr Server | App Server | API Key (location in App) | Sync Categories | Other | 
|---|---|---|---|---|---|--|
|Lidar | Full Sync | http://arrs.magic:9696 | http://arrs.magic:8686 | Settings → General → Security | Books/Comics (only one check) |  |
|Mylar | Full Sync | http://arrs.magic:9696 | http://arrs.magic:8090 | Settings → Web Interface →  API | Audio (except Audio/Vidio) | Sync Reject Blocklisted Torrent |
|Radarr | Full Sync | http://arrs.magic:9696 | http://arrs.magic:7878 | Settings → General → Security | All Movies | Sync Reject Blocklisted Torrent |
|Readarr | Full Sync | http://arrs.magic:9696 | http://arrs.magic:8787 | Settings → General → Security | Audio/Audiobook, Books (except Books/Comics) | Sync Reject Blocklisted Torrent |
|Sonarr | Full Sync | http://arrs.magic:9696 | http://arrs.magic:8989 | Settings → General → Security | All TV (except Anime) [Anime: TV/Anime] | Snyc Anime Standard, Sync Reject Blocklisted Torrent |

4. Settings → Apps → Test All Apps
1. Settings → Apps → Sync App Indexers


## Configure Download client on the ARRs servers
### Add download clients
For some reason, the download clients _are not_ synced from prowlarr to the arr servers...

1. [Prowlarr](http://arrs.magic:9696) → Settings → Download Clients
   1. qBittorrent- host: arrs.magic, port: 8200, username/password (per qBittorrent Options/WebUI/Authentication), add mapped categories per: [this guide](https://mediastack.guide/config/prowlarr/#add-torrent-downloader) (note anime and comics are the standouts)
   1. Sabnzbd - host: arrs.magic, port: 8100, api key & username/password (per Sabnzbd General → Security), add mapped categories as above.


1. [Lidarr](http://arrs.magic:8686), [Radarr](http://arrs.magic:7878), [Readarr](http://arrs.magic:8787), [Sonarr](http://arrs.magic:8989)
   1. Copy steps for adding download clients to Prowlarr. Pay attention to the default Category each app provides, they should be: music, movies, books, tv; respectively.

1. Mylar was configured previously.

## Other Servers Configuration

### Configure [Radarr](http://arrs.magic:7878)
Movie library management.

1. Settings → Metadata → Kodi (XBMC): Enable, Movie Metadata, Movie Images, Save
1. Settings → Metadata → Roksbox: Enable, Movie Metadata, Movie Images, Save
1. Settings → Metadata → WDTV: Enable, Movie Metadata, Movie Images, Save
1. Settings → Media Management: See [Common Settings for Apps](#common-settings-for-apps) and [Application File Naming](#application-file-naming)
1. Settings → Media Management → Add Root Folder: `/data/movie`

Time to import existing movies...

### Configure [Sonarr](https://arrs.magic:8989)
Television series library management.

1. Settings → Metadata → Kodi (XBMC): Enable, Series Metadata, Series Images, Season Images, Episode Images
1. Settings → Metadata → Roksbox: Enable, Save
1. Settings → Metadata → WDTV: Enable, Save
1. Settings → Media Management: See [Common Settings for Apps](#common-settings-for-apps) and [Application File Naming](#application-file-naming)
1. Settings → Media Management → Add Root Folder: `/data/tv`

  Time to import existing series...

### Configure [Lidarr](http://arrs.magic:8686)
Music library management.

1. Settings → Media Management: See [Common Settings for Apps](#common-settings-for-apps) and [Application File Naming](#application-file-naming)
1. Settings → Media Management → Add Root Folder
   1. Name: library
   1. Path: `/data/music`
   1. Monitor: None
   1. Monitor New Albums: No New Albums
   1. Quality Profile: Any
   1. Metadata Profile: Standard
   1. Save

  Time to import existing tunes...   

### Configure [Readarr](http://arrs.magic:8787)
Book and Audiobook library management.

1. Settings → Media Management: See [Common Settings for Apps](#common-settings-for-apps) and [Application File Naming](#application-file-naming)
1. Settings → Media Management → Add Root Folder
   1. Name: library
   1. Path: `/books`
   1. Monitor: All Books
   1. Monitor New Books: All Books
   1. Quality Profile: eBook
   1. Metadata Profile: Standard
   1. Default Readarr Tags:
   1. Use Calibre Content Server: false

  Time to import existing books...

### Configure [Bazarr](http://arrs.magic:6767)
Subtitles for Movies and TV.
1. Settings → Sonarr: Enabled
   1. Address: `arrs.magic`
   1. Port: `8989`
   1. API Key: from [here](http://arrs.magic:8989/settings/general)
   1. Test - version will be displayed if passing
   1. Save - required to enable further options
   1. Download Only Monitored: true
   1. Path Mappings:
      1. Sonarr: `/data/`
      1. Bazarr `/data/`
   1. Save
1. Settings → Radarr: Enabled
   1. Address: `arrs.magic`
   1. Port: `7878`
   1. API Key: from [here](http://arrs.magic:7878/settings/general)
   1. Test - version will be displayed if passing
   1. Save - required to enable further options
   1. Download Only Monitored: true
   1. Path Mappings:
      1. Radarr: `/data/`
      1. Bazarr `/data/`
   1. Save


### Configure [Pinchflat](http://arrs.media:8945)
1. \+ New Media Profile
   1. Use a Preset:  Media Center
   1. Name: Media Center
   1. Output path template:
   1. Subtitle Options → Download Subtitles: true
   1. Thumbnail Options → Download: true, Embed: true
   1. Metadata Options → Download: true, Embed: true
   1. Release Format Options → Shorts: Exclude, Livestreams: Exclude
   1. Quality Options → 1080p
   1. Media Center Options → Download NFO: true, Download Series: true
   1. SponsorBlock Options → Sponsor: true, Outro/Credits: true, Interaction Reminder: true, Self Promotion: true
   1. Save Media profile
1. Populate `/config/extras/cookies.txt
   1. **TODO**
1. \+ New Source
   1. General Options: channel URL or playlist URL
   1. Custom Name
   1. Downloading Options → Download Media: true, Cookie Behavior: `All Operations`
   1. Save Source
1. Let's Go
1. Config → Settings → Extractor Settings: Restrict Filenames: true, Sleep Interval: 30
1. Setup cookies



# Settings for Apps
The table below contains common settings for Lidarr, Radarr, Readarr, and Sonarr. `Show Advanced` is necessary, and don't forget to save.
| Setting | Value |
|-------------------------------------|-----------------------------|
| Rename Media File: | Yes |
| Replace Illegal Characters: | Yes |
| Colon Replacement: | Replace with Space Dash |
| Create empty media folders: | No |
| Delete empty folders: | Yes |
| Skip Free Space Check: | No |
| Minimum Free Space: | 10000 |
| **Use Hardlinks instead of Copy:** | **Yes** |
| Import Using Script: | Optional |
| Import Extra Files: | Optional |
| Unmonitor Deleted Media: | Optional |
| Propers and Repacks: | Prefer and Upgrade |
| Rescan Media Folder after Refresh: | Always |
| Set Permissions: | Yes |
| Chmod Folder: | 775 |

## Application File Naming
Radarr
- Movie Naming
Standard Movie Format: `{Movie CleanTitle} {(Release Year)} {Edition-{Edition Tags}} - [{Quality Full}, {MediaInfo VideoCodec}, {Mediainfo AudioCodec}]{-Release Group}`
- Movie Folder Format:`{Movie CleanTitle} {(Release Year)} [tmdb-{TmdbId}]`

Sonarr - TV / Anime Naming:
- Standard Episode Format: `{Series CleanTitle} ({Series Year}) - S{season:00}E{episode:00} - {Episode CleanTitle} - [{Quality Full}, {MediaInfo VideoCodec}, {Mediainfo AudioCodec}]{-Release Group}`
- Daily Episode Format: `{Series CleanTitle} ({Series Year}) - {Air-Date} - {Episode CleanTitle} - [{Quality Full}, {MediaInfo VideoCodec}, {Mediainfo AudioCodec}]{-Release Group}`
- Anime Episode Format: `{Series CleanTitle} ({Series Year}) - S{season:00}E{episode:00} - {Episode CleanTitle} - [{Quality Full}, {MediaInfo VideoCodec}, {Mediainfo AudioCodec}]{-Release Group}`
- Series Folder Format: `{Series CleanTitle} ({Series Year}) [tmdb-{TmdbId}]`
- Season Folder Format: `Season {season:00}`
- Specials Folder Format: `Specials`
- Multi Episode Style: `Prefixed Range`

Lidarr - Music Naming:
- Standard Track Format: `{Album CleanTitle} ({Release Year})/{Artist CleanName} - {Album CleanTitle} - {track:000} - {Track CleanTitle} - [{MediaInfo AudioCodec}, {MediaInfo AudioChannels}, {MediaInfo AudioBitRate}, {MediaInfo AudioSampleRate}, {MediaInfo AudioBitsPerSample}]{-Release Group}`
- Multi Disk Track Format: `{Album CleanTitle} ({Release Year})/{Artist CleanName} - {Album CleanTitle} {Medium Format} {medium:00} - {track:000} - {Track CleanTitle} - [{MediaInfo AudioCodec}, {MediaInfo AudioChannels}, {MediaInfo AudioBitRate}, {MediaInfo AudioSampleRate}, {MediaInfo AudioBitsPerSample}]{-Release Group}`
- Artist Folder Format: `{Artist CleanName} (mbid-{Artist MbId})`

Mylar3 - Comic Naming:
- Folder Format: `$Series ($Year)`
- File Format: `$Series $Annual $Issue ($Year)`

Readarr - ePub Naming:
- Standard Book Format: `{Book CleanTitle} ({Release Year})/{Author CleanName} - {Book CleanTitle}{ - Part (PartNumber:00)}{-Release Group}`
- Author Folder Format: `{Author CleanName}`


# TODOs
- configure apps programatically
- GPU passthrough 
- Terraform and ansible?
- [Bazarr providers](http://arrs.magic:6767/settings/providers)
- [Pinchflat cookies instructions](http://arrs.magic:8945/)
- 
### Configure others?
- ddns-updater
- tailscale/cloudflare?
  - To what do external users need access?
    - Seerr, probably just that
- [Calibre Content Server](https://manual.calibre-ebook.com/server.html) for Readarr? See https://github.com/AdrienPoupa/docker-compose-nas/blob/ee9d034b1ea0624ba1f66ea9da144ce0c98ef349/docker-compose.yml#L406

### Configure [Dispatcharr](https://dispatcharr.github.io/Dispatcharr-Docs/)


# References and citations
- https://github.com/AdrienPoupa/docker-compose-nas
- https://github.com/geekau/mediastack
- https://github.com/geekau/mediastack-guide
