---
title: "Nextcloud VM cloud backups with restic and Backblaze"
date: 2024-01-23T16:58:43+02:00
draft: false
tags: [nextcloud, linux, lang-english, backblaze]
---

# UPDATE 20241206

Happy Independence Day for all Finns <3

I have created a V1.0 script for backing up your Nextcloud instance using Restic.
It has been mainlined to "not supported" scripts folder and you can find it [here](https://github.com/nextcloud/vm/blob/main/not-supported/restic-cloud-backup-wizard.sh)

Currently script has support for:

- AWS S3 Bucket
- Azure Blob Storage
- Backblaze B2

Tested working with Azure and Backblaze, as I have access to those two environments.

Script has logging, error handling and hopefully clear instuctions how to set it up.
Also has options to only backup the config + DB or config + DB + /mnt/ncdata.

## Original post follows:

Currently there are no instructions on how to backup Nextcloud VM to a offsite location without mountpoints.
Mounting Backblaze B2 as a mount inside linux is supported as a S3 compatible fuse, but that gets really clunky, and has no support for snapshotted backups.

I have opened a [issue](https://github.com/nextcloud/vm/issues/2583) to the project, so this situation might get better soon :)

{{< toc >}}

## Current setup

I use 2.5G NIC, directly connected to my NAS from my ESXi host. In ESXi, I have a separate vSwitch which has a vmNIC to the nextcloud VM.
Inside nextcloud, the vmNIC has a static IP and subnet without gateway, so that only 192.168.40.0/24 traffic goes to this NIC.

My NAS is Xigmanas, running on HPE Proliant Microserver Gen7 (N54L) and has a Intel I-225V 2.5G NIC added to it.
In Xigmanas, NIC has IP 192.168.40.40, and presented NFS mounts of my ZFS pool to that IP.

Nextcloud VM has following line in /etc/fstab
```
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
192.168.40.40:/mnt/DATAPOOL/nextcloud /mnt/ncdata nfs defaults  0 1
```
This automounts /mnt/ncdata directly to NFS share from the NAS.

## Scripts and configs

### Install restic from restic
Current apt restic program is very old, "restic 0.12.1 compiled with go1.18.1 on linux/amd64".
So we need to first hold the apt version and use restic own .deb packages for install.
```
apt-mark hold restic
wget https://github.com/restic/restic/releases/download/v0.16.2/restic_0.16.2_linux_amd64.bz2
bzip2 -d restic_0.16.2_linux_amd64.bz2
cp restic_0.16.2_linux_amd64 /usr/bin/restic
chmod +x /usr/bin/restic
```

Now restic version shows "restic 0.16.2 compiled with go1.21.3 on linux/amd64"

You can now use restic self-update to update binary to latest release.

### Main backup script
First is the actual backup script which is responsible for backing up files and pruning old snapshots from the restic repository.

- BACKUP_PATHS are the locations we want to include inside the repository.
- RETENTION_DAYS, WEEKS, MONTHS and YEARS define how long we keep different snapshots.
- EXCLUDE_FILE is file used to exclude certain folders from repo.
- BACKUP_TAG is added to all snapshots, so you know what took the snapshots inside repo.
- source /etc/restic-env is used to save all restic repo settings.

/etc/restic-backup.sh

```
#!/usr/bin/env bash
# This script is intended to be run by a systemd timer

# Exit on failure or pipefail
set -e -o pipefail

#Set this to any location you like
BACKUP_PATHS="/mnt/ncdata /mnt/NCBACKUP"

EXCLUDE_FILE="/etc/restic-excludes.txt"

BACKUP_TAG="systemd.timer"

# How many backups to keep.
RETENTION_DAYS=30
RETENTION_WEEKS=4
RETENTION_MONTHS=1
RETENTION_YEARS=0

source /etc/restic-env

# Remove locks in case other stale processes kept them in
restic unlock &
wait $!

#Do the backup

restic backup \
  --verbose \
  --one-file-system \
  --tag $BACKUP_TAG \
  $BACKUP_PATHS \
  --exclude-file $EXCLUDE_FILE &

wait $!

# Remove old Backups

restic forget \
  --verbose \
  --tag $BACKUP_TAG \
  --prune \
  --keep-daily $RETENTION_DAYS \
  --keep-weekly $RETENTION_WEEKS \
  --keep-monthly $RETENTION_MONTHS \
  --keep-yearly $RETENTION_YEARS &
wait $!

# Check if everything is fine
restic check &
wait $!

echo "Backup done!"
```

### Restic settings
All settings related to repo go to this file.
ID's can be found from backblaze.com -> Account -> Application Keys

ALWAYS create a new key and give it write permissions to the bucket you want to save the backups.

- B2_ACCOUNT_ID is "keyID" in Backblaze.
- B2_ACCOUNT_KEY is shown you only one time during the creation of the key. Save this in you password manager so you can recreate the config, or create a new one per client you connect to the repository. 
- RESTIC_REPOSITORY is the Restic repo we must create before running the script for the first time.
- RESTIC_PASSWORD_FILE is the password we use to encrypt the repository data. SAVE THIS, because if you do not have this, you cannot open the repository ever again!

/etc/restic-env
```
export B2_ACCOUNT_ID="xxxxxxxxxxxxxxxxxxxxxx"
export B2_ACCOUNT_KEY="xxxxxxxxxxxxxxxxxxxxxxxx"
export RESTIC_REPOSITORY="b2:your-amazing-b2-nextcloud-repo"
export RESTIC_PASSWORD_FILE=/etc/restic-password
```

### Restic ignore files
I would highly recommend you ignore the preview folder from backups.
This reduces the time restic has to compare snapshots, as these files change very often and are not needed if you need to do a restore.

/etc/restic-excludes.txt
```
/mnt/ncdata/*/preview/
```
This will match all folders under /mnt/ncdata which are named preview.
You can also use the full folder name, eg /mnt/ncdata/appdata_ocupxw9n9gyb/preview but I think this is unique per server.
If you have a user name preview, well, this will also exclude all files from that user. But don't do that :)

### Repository password
Just type the repository password inside this file.

### Securing it all!
Do not forget this.

```
chown root:root /etc/restic-env
chown root:root /etc/restic-password
chmod 600 /etc/restic-env
chmod 600 /etc/restic-password
```

### Creating restic repository
We need to initialize the restic repo to connect to backblaze.

1. Elevate to root inside nextcloud VM, `sudo su root`.
2. Source all settings with `source /etc/restic-env`.
3. Create the restic repo with `restic -r b2:your-amazing-b2-nextcloud-repo init`

Now you have a working repository to store snapshots.
You can check that it was created successfully from backblaze.com -> B2 Cloud Storage -> Browse files -> your-amazing-b2-nextcloud-repo -> you should see "config" file and data, index, keys, snapshots folders.

## First backup
We run the first backup manually, as it might take a long time and we can see if we have any errors etc.

First we source the backup config again and then run the actual backup.

- "-r" defines the repo name.
- backup is the restic action.
- After backup are the directories we want to backup.
- Then we also need to add our exclude file so we do not backup previews.
- Add --dry-run if you want to test your excludes without copying all the data.
- And we run it all inside screen so we dont need to watch the console for 10 hours :). 
```
source /etc/restic-env
screen restic -r b2:your-amazing-b2-nextcloud-repo backup /mnt/ncdata /mnt/NCBACKUP --exclude-file "/etc/restic-excludes.txt" --tag "systemd.timer"
```
Now you should see following:
```
repository 7dfb0247 opened successfully, password is correct
no parent snapshot found, will read all files

Files:       85163 new,     0 changed,     0 unmodified
Dirs:        69932 new,     0 changed,     0 unmodified
Added to the repo: 1.062 KiB

processed 85163 files, 47.856 GiB in 22:05
snapshot 3ca17de6 saved
```

## Automation
Now that we have a working backup with a snapshot, we need to automate the whole deal so we dont have to SSH to our box to run the backups.

Create a systemd service to `/etc/systemd/system/backup.service`
```
[Unit]
Description=Backup with restic
[Service]
Type=simple
Nice=10
ExecStart=/etc/restic-backup.sh
#$HOME must be set for restic to find /root/.cache/restic/
Environment="HOME=/root"
```
Then create a timer for the backup to `/etc/systemd/system/backup.timer`
```
[Unit]
Description=Backup on schedule

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```
Now lets make systemd aware of the new service and then enable it:
```
systemctl daemon-reload
systemctl enable --now backup.timer
```
To check the next run time, run `systemctl list-timers`.

To run the backup now, run `systemctl start backup.service`.

Check logs with `journalctl -f -u backup.service`.

## TODO

Add some way to inform if something is not right or backups fail...