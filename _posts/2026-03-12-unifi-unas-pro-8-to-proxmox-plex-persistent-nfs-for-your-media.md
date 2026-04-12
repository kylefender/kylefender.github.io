---
layout: post
title: "Unifi UNAS Pro 8 to Proxmox & Plex: Persistent NFS for Your Media"
description: "Unlock robust and reliable media storage! This post guides you through setting up a permanent NFS connection from your Unifi UNAS Pro 8 to a Proxmox server, then seamlessly bind mounting that storage into your Plex LXC container for optimal performance and accessibility."
date: 2026-03-12 20:41:05 -0400
categories: [Blog]
tags: [homelab, proxmox, plex, nfs, unifi, self-hosting]
media_subpath: /assets/images/posts/unifi-unas-pro-8-to-proxmox-plex-persistent-nfs-for-your-media/
---

If you're running Plex inside a Proxmox LXC container and your media lives on a Unifi UNAS Pro 8, getting that storage reliably mounted, and keeping it mounted across reboots, takes a few steps. This guide walks through the whole chain: enabling NFS on the UNAS, mounting it on the Proxmox host, making it persistent with `fstab`, and bind mounting it into the Plex LXC container.

## Prerequisites

- Unifi UNAS Pro 8 set up and accessible on your local network
- Proxmox host running and reachable
- A Plex LXC container already created
- Basic familiarity with the Proxmox shell

## (Optional) Step 0: Setup a Separate User

I setup a separate user specifically for Plex. You can setup the NFS connection with your user, but I wanted to keep the access separate.

1. Log into your UniFi OS console and navigate to the **Drive** application.
2. Click the **Admins & Users** in the left-hand navigation bar.
    **NOTE:** Sometimes the linke doesn't navigate to the correct page; the URL should look like: `https://unifi.ui.com/consoles/<YOUR_CONSOLE_ID>/drive/admins/users`
3. At the top, click **Create New**.
4. Fill in the user details.
5. If you already have the Shared Drive created, click the **Shared Drives** link and select the drives that Plex will need access to.

## Step 1: Enable NFS on the UNAS Pro 8

1. Log into your UniFi OS console and navigate to the **Drive** application.
2. Click **All Files** in the left-hand navigation bar.
3. Create the Shared Drive you want to use, if you haven't already.
4. Navigate to **Settings** → **Services**, look for the **File Services** section, then look for **NFS**.
5. Enable the NFS service by checking the box next to **NFS**.
6. Add a new NFS share, by clicking **ADD NFS Connections**.

    ![Add NFS Connections](Add_NFS_Connection.png)
    _Add NFS Connections_

7. Set the **Hostname or IP** to your Proxmox host's IP address (ex., `192.168.1.70`).
8. Set the **NFS Write Mode**; I selected **sync**.
9.  Click the **Add Shared Drives** link.
10. Select the Shared Drive(s) you want to use, and make sure to select the permission you want at the bottom, **Read-Write** or **Read-Only**.
11. Click **Apply**.

![Completed setup enabling NFS service and connections for Unifi Drive](File_Services.png)
_Completed setup enabling NFS service and connections for Unifi Drive_

Take note of the UNAS Pro's IP address (ex. `192.168.1.83`) and the NFS export path, which looks like `/var/nfs/shared/[Shared Drive Name]`.

## Step 2: Prepare the Proxmox Host

My Proxmox setup is fairly simple with a single node, so you might need to do this on every node if you have more nodes.

1. Open a browser and navigate to your Proxmox UI.
2. Select your node, and click **Shell** in the menu.

    ![Proxmox Node Shell](Proxmox_Node_Shell.png)
    _Proxmox Node Shell_

3. Create a local mount point on the Proxmox host.

    ```bash
    mkdir -p /mnt/unas-pro-8-tvshows
    ```
    {: .nolineno }

4. Test that the NFS share is reachable and mounts correctly before committing it to `fstab`:

    ```bash
    mount -t nfs 192.168.1.83:/var/nfs/shared/TVShows /mnt/unas-pro-8-tvshows
    ```
    {: .nolineno }

5. Verify you can see your files:

    ```bash
    ls /mnt/unas-pro-8-tvshows
    ```
    {: .nolineno }

6. If everything looks good, unmount the share. We will add the permanent mount to `fstab`. **NOTE**: the command **umount**, not **unmount**.

    ```bash
    umount /mnt/unas-pro-8-tvshows
    ```
    {: .nolineno }

## Step 3: Make the NFS Mount Persistent with fstab

1. In the same Shell session for the Proxmox host, open `/etc/fstab`.

    ```bash
    nano /etc/fstab
    ```
    {: .nolineno }

2. Add the following line at the bottom, replacing the IP (of the NAS) and paths with yours. On a Windows machine, I can paste into the Shell using `Ctrl+Shift+v`. After the mount is in `fstab`, you will need to save and exit nano. To do that, you will press `Ctrl+x` to start to exit, `y` to save the changes, and `Enter` to finish.

    ```plaintext
    mount 192.168.1.83:/var/nfs/shared/TVShows /mnt/unas-pro-8-tvshows nfs defaults 0 0
    ```
    {: .nolineno }

3. Reload the systemd daemon and re-mount everything in `fstab`.

    ```bash
    systemctl daemon-reload
    mount -a
    ```

4. Confirm the mount is active. You should see a line of text return in the shell when you run the following command.

    ```bash
    df -h | grep unas-pro-8-tvshows
    ```
    {: .nolineno }

## Step 4: Bind Mount the NFS Share into the Plex LXC Container

Now that the Proxmox host has the NFS share reliably mounted, you need to expose it to the Plex container as a bind mount.

1. Look in your left-hand navigation bar for Proxmox to find your Plex container's ID (ex. `101`). It'll be the number in the name of the LXC.

    ![Proxmox Application List](Proxmox_Application_List.png)
    _Proxmox Application List_

2. Select your Plex container, and click **Shell** in the menu.
3. Edit the Plex LXC's config file.

    ```bash
    nano /etc/pve/lxc/101.conf
    ```
    {: .nolineno }

4. Add a `mp` (mount point) line at the bottom of the config. **NOTE:** This will be my second mount, so it is `mp1`, but if this is your first, it will be `mp0`. Also, the `mp` doesn't need to match the `mp#` if you want it to be mounted somewhere else in the container. You can change the container-side path to whatever makes sense for your Plex library setup.

    ```plaintext
    mp1: /mnt/unas-pro-8-tvshows,mp=/mnt/unas-pro-8-tvshows
    ```
    {: .nolineno }

5. Restart the Plex container to apply the change. I usually do this in from the UI.

## Step 5: Add the Library in Plex

1. Navigate and log into your Plex web UI.
2. Click **+** next to your library name (mine says **Plex**). **NOTE:** Two things, one, if you don't see a plus sign, you might need to click `MORE >` to navigate to where you can add a library. Two, the plus button is hidden until you are hovering over the library name.
3. Choose the library type (Movies, TV Shows, etc.) and click **Next**.
4. Click **Browse For Media Folder** and select `unas-pro-8-tvshows` (or whichever path you set as `mp=` in the container config), then click **Add Library**.
5. Plex will scan and start building your library.

## Wrapping Up

With this setup, your UNAS Pro 8's NFS share will survive reboots on both the Proxmox host and the Plex container.

If more containers need access to the same share (ex. Immich), you just add another `mp` line to each container's config pointing at the same `/mnt/unas-pro-8-tvshows` host path. No need to create additional NFS mounts.
