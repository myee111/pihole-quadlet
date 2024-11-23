## Pihole-quadlet
Pihole-quadlet is a collection of Podman configuration files compatible with the Quadlet program. Quadlet enables you to define and run containers as services with Podman. Quadlet is similar to docker-compose or podman-compose.

For more information about the Pi-hole container, [go here](https://hub.docker.com/r/pihole/pihole).

For more information about Quadlet, please read [this](https://www.redhat.com/sysadmin/multi-container-application-podman-quadlet) and [this](https://www.redhat.com/sysadmin/quadlet-podman).

This repo contains configuration files that enable you to quickly set up Pi-hole to run as a rootless container service on Red Hat Enterprise Linux. *You can get 16 free Red Hat Enterprise Linux licenses and access to Red Hat Insights by simply registering for a new account at https://console.redhat.com*. Read more about it [here](https://developers.redhat.com/articles/faqs-no-cost-red-hat-enterprise-linux).

These instructions will probably work in the latest version of Fedora. They will definitely work in RHEL 9.2. 

## Set up

### Install RHEL container tools

```bash
dnf install -y container-tools
```

### Enable unprivileged access to port 53

You need to do this or Pi-hole won't work. Make this change as the root user.

Add this to `/etc/sysctl.conf`:

```bash
net.ipv4.ip_unprivileged_port_start = 53
```

Then run `systctl -p` to enable the change.

### Disable systemd-resolved DNS stub listener

```bash
cat <<EOF > /etc/systemd/resolved.conf.d/no-stub.conf
[Resolve]
DNS=127.0.0.1
DNSStubListener=no
EOF

systemctl restart systemd-resolved
```

--- 

**Perform the remaining actions as a non-root user.**

---
## Make systemd user services persist

```bash
loginctl enable-linger <user>
```

### Clone this repository

Clone this repository.

```
git clone https://github.com/myee111/pihole-quadlet.git
```
### Create directories in your home directory
Create the directory needed for Podman to find your configuration files.

```bash
mkdir -p ~/.config/containers/systemd/
```

### Modify the config files

All 3 files need to modified. Luckily there isn't much to change. Look at the comments inline.

### Copy the files to the  `~/.config/containers/systemd/` directory

```
cp ~/pihole-quadlet/pihole* ~/.config/containers/systemd/
```

### Reload systemd with the new config files

```bash
systemctl --user daemon-reload
```

### Check that there are no syntax errors

```bash
/usr/libexec/podman/quadlet -dryrun -user
```

### Start up Pi-hole

```bash
systemctl --user start pihole.service
```

That's it! You should be up and running. Try logging into your new Pi-hole server.

## Automatically updating Pi-hole

The configuration file has already configured autoupdates for the Pi-hole container.

All you have to do is enable the autoupdate service with this command.

```bash
systemctl --user enable --now podman-auto-update.service
```

By default, this service will run every night.

If you want to reduce the frequency of the autoupdates, run the following and create your own schedule.

```bash
systemctl --user edit podman-auto-update.timer
```

## FAQ

### What is a rootless container?

It's when you run your container as a regular user and not as root. [You can read more about it here](https://developers.redhat.com/blog/2020/09/25/rootless-containers-with-podman-the-basics).

### Why can't I do this with a docker-compose file and docker?
Sure you can. Go right ahead.

### Don't I need to download the container from the docker registry first?

No. Podman will do that for you.

### Why don't I need to run `systemctl --user enable pihole.service`?

The pihole.container file specifies that the service starts upon startup.

### How are persistent volumes handled?

The configuration files `pihole-etc.volume` and `pihole-dnsmasq.volume` tell Podman to create persistent volumes in `/var/home/myee/.local/share/containers/storage/volumes`. 

Specifically, look for `systemd-pihole-dnsmasq` and `systemd-pihole-etc` (unless you changed the name of the volume files).


## Troubleshooting

View pihole log.

```
podman exec -it pihole cat /var/log/pihole-FTL.log
```

View the volume configuration.

```
[myee@protectli1 ~]$ podman volume inspect systemd-pihole-dnsmasq
[
     {
          "Name": "systemd-pihole-dnsmasq",
          "Driver": "local",
          "Mountpoint": "/var/home/myee/.local/share/containers/storage/volumes/systemd-pihole-dnsmasq/_data",
          "CreatedAt": "2023-08-28T15:58:46.941744165-04:00",
          "Labels": {},
          "Scope": "local",
          "Options": {},
          "MountCount": 0
     }
]
```
