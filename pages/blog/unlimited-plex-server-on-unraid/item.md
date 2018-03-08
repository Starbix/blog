---
title: 'Unlimited Plex Server on unRAID'
date: '12-09-2017 17:27'
taxonomy:
    category:
        - blog
    tag:
        - software
        - Plex
        - cloud
        - rclone
        - sonarr
        - radarr
        - unRAID
visible: true
---

This tutorial is not as detailed as https://hoarding.me so I suggest first reading through that tutorial to get a broad view of how it rclone, Sonarr and Radarr work with the cloud.

unRAID makes running a cloud based unlimited Plex server not as easy as it's on Ubuntu. So here's a tutorial for unRAID.


## Required Plugins

- [Community Applications](https://forums.lime-technology.com/topic/38582-plug-in-community-applications)
- CA User Scripts (search in Community Applications)
- rclone or rclone-beta by Waseh (search in Community Applications)
- [unionfs](https://raw.githubusercontent.com/Starbix/unRAID-plugins/master/plugins/unionfs.plg)
- [plexdrive](https://raw.githubusercontent.com/Starbix/unRAID-plugins/master/plugins/plexdrive.plg)

To install the needed plugins, just enter this https://raw.githubusercontent.com/Squidly271/community.applications/master/plugins/community.applications.plg to **Plugins > Install Plugin**.
After refreshing the site you should see a new Tab called **Apps**, under that Tab you just search for **User Scripts** and **rclone** and install both of those plugins.

For our setup we also need unionfs and plexdrive for which I didn't find any plugin, so I wrote my own. Install it like Community Applications with https://raw.githubusercontent.com/Starbix/unRAID-plugins/master/plugins/unionfs.plg and https://raw.githubusercontent.com/Starbix/unRAID-plugins/master/plugins/plexdrive.plg.


To edit your rclone config, SSH into your server and execute `rclone config`.

Add the following remotes:
```
Name                 Type
====                 ====
gdrive               drive
gdrivecrypt          crypt	("remote" = "/mnt/disks/pd/crypt")
uploadcrypt          crypt	("remote" = "gdrive:crypt")
```
Note: Use the same password for the crypt remotes.

Then create the config file of plexdrive using
```
plexdrive mount /mnt/disks/pd -c "/mnt/user/appdata/plexdrive" -o allow_other -v 2
```
then upload mountcheck files used for checking whether everything is mounted properly or not
```
touch mountcheck
rclone copy mountcheck uploadcrypt:Series -vv --no-traverse
rclone copy mountcheck uploadcrypt:Movies -vv --no-traverse
rclone copy mountcheck uploadcrypt: -vv --no-traverse

```
Then go to 

After setting up your rclone shares, go into **Settings > User Scripts**.

Here we'll need to add several scripts. 
(These scripts are mostly stolen from [hoarding.me](https://hoarding.me))

**fuse_mount** - at Startup of the Array
```
#!/bin/bash
if [[ -f "/mnt/user/media/cloud/Series/mountcheck" ]] && [[ -f "/mnt/user/media/cloud/Movies/mountcheck" ]]; then
echo "$(date "+%d.%m.%Y %T") INFO: Check successful, Series and Movies fuse mounted."
exit
else
echo "$(date "+%d.%m.%Y %T") ERROR: Series and Movies not mounted, remount in progress."
# Unmount before remounting
fusermount -uz /mnt/user/media/cloud/Series
fusermount -uz /mnt/user/media/cloud/Filme
unionfs -o cow,allow_other,direct_io,auto_cache,sync_read /mnt/user/media/cloud/Seriestmp=RW:/mnt/disks/crypt/Series=RO /mnt/user/media/cloud/Series
unionfs -o cow,allow_other,direct_io,auto_cache,sync_read /mnt/user/media/cloud/Moviestmp=RW:/mnt/disks/crypt/Movies=RO /mnt/user/media/cloud/Movies
if [[ -f "/mnt/user/media/cloud/Series/mountcheck" ]] && [[ -f "/mnt/user/media/cloud/Movies/mountcheck" ]]; then
echo "$(date "+%d.%m.%Y %T") INFO: Remount of Series and Movies successful."
else
echo "$(date "+%d.%m.%Y %T") CRITICAL: Remount failed."
fi
fi
exit
``` 
**rclone_mount** - at Startup of the Array
```
#!/bin/bash

mkdir -p /mnt/disks/pd
mkdir -p /mnt/disks/crypt

#This section mounts the various cloud storage into the folders that were created above.
if [[ -f "/mnt/disks/crypt/mountcheck" ]]; then
echo "$(date "+%d.%m.%Y %T") INFO: Check successful, crypt mounted."
exit
else
echo "$(date "+%d.%m.%Y %T") ERROR: Drive not mounted, remount in progress."
# Unmount before remounting
fusermount -uz /mnt/disks/crypt
fusermount -uz /mnt/disks/pd
plexdrive mount /mnt/disks/pd -c "/mnt/user/appdata/plexdrive" -o allow_other -v 2 &
rclone mount --max-read-ahead 512M --allow-other --allow-non-empty -v --buffer-size 1G gdrivecrypt: /mnt/disks/crypt &
if [[ -f "/mnt/disks/crypt/mountcheck" ]]; then
echo "$(date "+%d.%m.%Y %T") INFO: Remount successful."
else
echo "$(date "+%d.%m.%Y %T") CRITICAL: Remount failed."
fi
fi
exit
```

**rclone_unmount** - at Stopping of the Array
```
#!/bin/bash
fusermount -uz /mnt/user/media/cloud/Movies
fusermount -uz /mnt/user/media/cloud/Series
fusermount -uz /mnt/disks/crypt
fusermount -uz /mnt/disks/pd
``` 

**rclone_upload** - `*/30 * * * *` or whatever you want
```
#!/bin/bash
uploadfolderMovies="/mnt/user/media/cloud/Moviestmp" 
uploadfolderSeries="/mnt/user/media/cloud/Seriestmp" 
rclone move $uploadfolderSeries uploadcrypt:/Series -vv --no-traverse --transfers=15 --checkers=4 --drive-chunk-size 512M --delete-after
rclone move $uploadfolderMovies uploadcrypt:/Movies -vv --no-traverse --transfers=15 --checkers=4 --drive-chunk-size 512M --delete-after
```


## Setup Radarr, Sonarr and Plex

You need to mount /media with the **RW/Slave** option in both Radarr and Sonarr. Otherwise it won't work.

Point Sonarr to /media/cloud/Series for folder containing shows.
Point Radarr to /media/cloud/Movies for folder containing movies.

Point Plex directly to the rclone mount `/mnt/disks/crypt/Movies` and `/mnt/disks/crypt/Series` using the RO/Slave option.