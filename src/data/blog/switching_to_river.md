---
title: Switching to river on Fedora Silverblue
author: Wilf Silver
pubDatetime: 2026-03-16T20:30:00Z
slug: river_on_silverblue
featured: true
draft: false
tags:
  - river
  - fedora
  - systemd
  - desktop
description: Why is this so difficult
---

Things to talk about

- Setting up the systemd commands + desktop
- Screen zoom in with `kanshi`
- Using the `systemd-xdg-autostart-generator`
- Deja Dup with, editing `.config/xdg-desktop-portal/portals.conf`
  - Found by `dbus-monitor --session`
  - Test with:
    ```sh
    dbus-send --session --dest=org.freedesktop.portal.Desktop --type=method_call --print-reply \
      /org/freedesktop/portal/desktop \
      org.freedesktop.DBus.Properties.Get \
      string:"org.freedesktop.portal.Secret" \
      string:"version"
    ```
  - Automated backups: Enable in gnome and then it works as long as you have it autostarting
