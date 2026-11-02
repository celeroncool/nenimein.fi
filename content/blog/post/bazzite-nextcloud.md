---
title: "Bazzite Nextcloud logouts"
date: 2026-11-02T20:07:32+02:00
draft: false
tags: [linux, lang-english]
---

# Bazzite, Nextcloud and KDE Wallet logouts

I have been running Bazzite as my main machine for over a year now.

I also host my own Nextcloud instance for myself & friends (Cloud is just someone else's computer...).

The most annoying thing would happen every time I rebooted my machine: Nextcloud would log out, and I would need to login every time with password and 2fa. Very annoying!

I was chasing the issue for months, I would make session times longer, tune php.ini files and finally got it to atleast recognice my account but it still seemingly logged out every time.

**Client:** Nextcloud Desktop 35.0.0daily, Flatpak (`com.nextcloud.desktopclient.nextcloud`)

## The fix!

After digging through journal logs and saw this multiple times:
```
marras 02 19:35:03 bazz.nenimein.fi /usr/libexec/xdg-desktop-portal[3968]: Backend call failed: Could not activate remote peer 'org.freedesktop.impl.portal.desktop.kwallet': unit failed
marras 02 19:36:23 bazz.nenimein.fi systemd[3278]: Started dbus-:1.3-org.freedesktop.impl.portal.desktop.kwallet@25.service.
marras 02 19:36:23 bazz.nenimein.fi systemd[3278]: dbus-:1.3-org.freedesktop.impl.portal.desktop.kwallet@25.service: Main process exited, code=exited, status=255/EXCEPTION
marras 02 19:36:23 bazz.nenimein.fi systemd[3278]: dbus-:1.3-org.freedesktop.impl.portal.desktop.kwallet@25.service: Failed with result 'exit-code'.
marras 02 19:36:23 bazz.nenimein.fi systemd[3278]: Started dbus-:1.3-org.freedesktop.impl.portal.desktop.kwallet@26.service.
marras 02 19:36:23 bazz.nenimein.fi systemd[3278]: dbus-:1.3-org.freedesktop.impl.portal.desktop.kwallet@26.service: Main process exited, code=exited, status=255/EXCEPTION
marras 02 19:36:23 bazz.nenimein.fi systemd[3278]: dbus-:1.3-org.freedesktop.impl.portal.desktop.kwallet@26.service: Failed with result 'exit-code'.
marras 02 19:36:23 bazz.nenimein.fi /usr/libexec/xdg-desktop-portal[3968]: Backend call failed: Could not activate remote peer 'org.freedesktop.impl.portal.desktop.kwallet': unit failed
```

The fix turned out to be very simple:

Navigate to "System Settings" -> "KDE Wallet" -> Check the box "Enable the KDE wallet subsystem".

Nowhere in flatpack instructions, or reinstalling the app, or logging in, was there ever a popup that hey, by the way, if you want to NOT GET LOGGED OUT EVERY TIME, please enable KDE Wallet...