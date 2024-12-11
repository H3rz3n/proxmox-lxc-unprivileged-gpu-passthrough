# How pass a GPU to an unprivileged Proxmox LXC container

## Prepare the container .conf file
From the Proxmox Host, add the following lines to the container .conf file, who is located in `/etc/pve/lxc/ct_number_here.conf` :

```
unprivileged: 1
lxc.apparmor.profile: unconfined
lxc.cap.drop: 
lxc.cgroup.devices.allow: c 226:* rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
```

## Choose what card you want to pass
From the Proxmox Host use the command :
```
ls -l /dev/dri
```
to list all your available graphic cards. You will have an output similar to this :

```
drwxr-xr-x 2 root   root         80 Dec 11 17:10 by-path
crw-rw---- 1 root   root 226,   0 Dec 11 17:10 card0
crw-rw---- 1 root   root 226, 128 Dec 11 17:10 renderD128
```

## Prepare the UDEV rules on the host
From the Proxmox Host create a new UDEV rule in `/etc/udev/rules.d/` using :
```
nano /etc/udev/rules.d/99-gpu-passthrough.rules
```
And then put inside it the following lines :
```
KERNEL=="card0", SUBSYSTEM=="drm", MODE="0660", OWNER="100000", GROUP="100000"
KERNEL=="renderD128", SUBSYSTEM=="drm", MODE="0660", OWNER="100000", GROUP="100000"
```
Remember to substitute the numbers in `card0` and `renderD128` with the one present on your system.

After saving the file you need to execute :
```
udevadm control --reload-rules
udevadm trigger
```
To properly load and enable the new UDEV rule now and on startup.

## Fix container permission on every container startup
The last step is to fix the gpu permission from inside our container. The most simple way is to create a media group called `media_group`, add to it every user who need to use the GPU and creater a simple script to launch on startup to apply those permissions on every container startup.

### Create the media_group group and add the users to it
To create the media_group group you need to use :
```
groupadd media_group
```
To add an user to the media_group you need to use :
```
usermode -aG media_group user_username
```
Two the most common cases are :
```
usermode -aG media_group root
usermode -aG media_group jellyfin
```

### Create the script to fix permission on startup
Choose a directory of your choice and use the command `nano gpu_permission_fix.sh` to create a new script. Inside it put the following lines :
```
#!/bin/bash
chown root:media_group /dev/dri/renderD128
chmod 660 /dev/dri/renderD128
```
Remember to substitute the numbers in `renderD128` with the one present on your system.

Now you need to create a systemd service to make the script start on startup. Let's use the command :
```
nano /etc/systemd/system/gpu_permission_fix.service
``` 
To create the file and put inside it the following lines :
```
[Unit]
Description=Run GPU permission fix at startup
After=network.target

[Service]
ExecStart=/path/to/your/script/file/gpu_permission_fix.sh
Restart=no

[Install]
WantedBy=multi-user.targe
```
`
Remember to substitute `/path/to/your/script/file/` with the path you choosed for your script file. You can check it with `pwd` from inside the folder where is located your script file.

Now you just need to start this service and enable it on startup with `systemctl enable gpu_permission_fix.service` and `systemctl start gpu_permission_fix.service`.

## Check if the permission are correctly applied
Now if you use `ls -l /dev/dri` you should see an output similar to this :
```
drwxr-xr-x 2 nobody nogroup           80 Dec 11 16:10 by-path
crw-rw---- 1 root   root        226,   0 Dec 11 16:10 card0
crw-rw---- 1 root   media_group 226, 128 Dec 11 16:10 renderD128
```
