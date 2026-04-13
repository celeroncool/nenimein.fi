---
title: "Migrating mumble servers"
date: 2022-02-03T19:44:10+02:00
draft: false
tags: [projects, lang-english, linux, server]
---

How to migrate mumble server to from old linux server to new linux server.

This is mostly a reminder for myself, on how to do it in the future if I ever need to migrate servers again. But, it might also help you if you hit a few "gotchas" on the way as I did.

## Prep

First, set up the new server. My new server is running Ubuntu 20.04 LTS, so I get the best mix of latest packages and stability.
Most of the commands are run as #root.

### Install mumble

First, we need to install mumble server, or "murmur" to the new server. Start by running:

```# apt get update```

```# apt get upgrade```

```# apt get install mumble-server```

```# apt get install certbot``` <- Optional, for "green" connection bar in mumble.

Then we need to stop the newly installed mumble-server service to make it ready to accept new configs.

```# systemctl stop mumble-server```

Make sure you also open up port TCP 80 for the certificate renewals and your mumble-server TCP/UDP port.

### Copy DB file

On the old server, stop mumble server and copy the DB file to the new server via filezilla or SCP.
Here I'm using SQLite as the database. You can find your database by checking the config file ```/etc/mumble-server.ini```.

```# systemctl stop mumble-server```

```# cp /var/lib/mumble-server/dbname.sqlite /home/user/dbname.sqlite```

Copy the file to the new server, and move it to ```/var/lib/mumble-server/dbname.sqlite```.
Then set the correct permissions to the DB file. Otherwise you will have errors regading unaccessible DB file :).
You can find the correct user in ```/etc/mumble-server.ini``` -> ```uname=username```.

```# chown mumble-server:mumble-server /var/lib/mumble-server/murmur.sqlite```

### Compare new config file to old config file

Copy all the necessary settings from old config file to new config file. I do not recommend copying the whole file to the new server, as there might be new config options on new installs.

Most important config options are:
- ```database=/var/lib/mumble-server/dbname.sqlite``` <- Make sure this points to a correct file.
- ```logfile=/var/log/mumble-server/mumble-server.log```
- ```welcometext="Welcome to this server running Murmur. Please enjoy your stay!"```
- ```port=1337```
- ```serverpassword=``` <- If you are using a password.
- ```sslCert=/etc/letsencrypt/live/server.com/fullchain.pem``` <- If you are using SSL.
- ```sslKey=/etc/letsencrypt/live/server.com/privkey.pem``` <- If you are using SSL.

### Set up SSL certificates with letsencrypt (optional)

Here I'm using certbot interactively in standalone mode to get a new cert to the server
Remember to point your mumble DNS name to the new server, otherwise the certificate renew will not go through.

```# certbot certonly -a standalone -d mumble.server.com```

You should see something like this which tells you where the certificates are saved:
"```Congratulations! Your certificate and chain have been saved at:```"

Next you have to configure mumble-server to start as root and drop it privileges after start, so mumble can read the certificate file.

```# vim /etc/default/mumble-server``` and set ```MURMUR_USE_CAPABILITIES=1```

## Start chatting!

Now start the mumble server with command: 

```# systemctl start mumble-server```

Check that mumble-server service status is "active: (running)" with:

```# systemctl status mumble-server```

And finally let's make it so that mumble-server will automatically start after reboots/upgrades:

```# systemctl enable mumble-server```