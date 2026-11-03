# Minecraft on OCI Free Tier with Rootless Podman

Companion configuration for [Minecraft Server on OCI Free Tier with Rootless Podman](https://blog.dieschmitterlinge.de/cloud/minecraft-server-oracle-oci-free-tier-podman/) by Martin Schmitter.

One VM, one container, persistent world data and a player allowlist. The container combines a Java server with Geyser/Floodgate for Bedrock clients. This repository supplies the Podman configuration around the existing upstream image; it does not build or redistribute Minecraft or the container image.

## Files

- [mc.container](mc.container): the Quadlet definition from the article, unchanged apart from a final newline.
- [server.properties.example](server.properties.example): the two allowlist settings to merge into the server's generated configuration.
- [.gitignore](.gitignore): excludes common local runtime data and credentials.

The article describes Ubuntu 24.04 on an OCI Ampere A1 ARM64 VM with Podman 4.9.3 and cgroup v2. These files have been checked against the published configuration and the [Podman 4.9.3 Quadlet documentation](https://docs.podman.io/en/v4.9.3/markdown/podman-systemd.unit.5.html);

## Before starting

Use a non-root Linux account for all Podman and user-service commands. Install the packages on the Ubuntu host:

```bash
sudo apt-get update
sudo apt-get install -y podman uidmap slirp4netns fuse-overlayfs dbus-user-session netavark aardvark-dns
```

Check the rootless environment:

```bash
podman --version
podman info --format '{{.Host.CgroupsVersion}}'
grep "^$(whoami):" /etc/subuid /etc/subgid
```

Expect cgroup v2 and subordinate UID/GID mappings for the service account. Enable lingering if the user service should start at boot and survive logout:

```bash
sudo loginctl enable-linger "$(whoami)"
```

Provision and mount persistent storage separately. The article uses an attached volume at /mnt/oraclevdb2. Confirm it is mounted before starting the service; creating an empty directory does not mount a disk. Do not copy a disk partitioning or fstab layout without adapting it to your host.

## Install the Quadlet

Clone this repository, change into its directory, and review mc.container before installing it:

```bash
git clone https://github.com/mschmitter-dba/minecraft-oci-rootless-podman.git
cd minecraft-oci-rootless-podman
```

Adjust these values for your host:

| Setting | Article value | What to check |
| --- | --- | --- |
| Volume | /mnt/oraclevdb2/podmanvolumes/minecraft_1:/minecraft:Z | The host directory must be on the intended mounted disk and writable by the rootless account. The :Z label is for SELinux. |
| MaxMemory | 4096 | The image's Java heap setting, not a container memory limit. Leave RAM for the OS, native allocations and other processes; reduce this for smaller VMs. |
| TZ | Europe/Berlin | Your time zone. |
| ScheduleRestart | 3:30 | The upstream image's scheduled restart setting. |
| Image / Pull | :latest / newer | Mutable image: service starts may pull updates. Back up before updates; pin a tested digest if reproducibility matters. |

After preparing the writable data directory, install the file and create the network as the same non-root account:

```bash
mkdir -p ~/.config/containers/systemd
cp mc.container ~/.config/containers/systemd/mc.container

podman network create minecraft_default

systemctl --user daemon-reload
systemctl --user start mc.service
```

If minecraft_default already exists, inspect it with podman network inspect minecraft_default instead of creating it again. The fully qualified docker.io image name does not need an unqualified registry search configuration.

Quadlet generates mc.service from mc.container. Its [Install] section supplies boot activation; do not try to enable the generated service with systemctl enable. Keep the persistent disk available before the user service starts, including after reboot. The supplied definition reproduces the article and does not add a dependency on your particular disk mount.

Review the upstream project's first-start requirements and the applicable Minecraft EULA yourself before running the server. This repository does not accept terms on your behalf.

## Allow only known players

Keep public ingress closed until the server configuration is ready. Once the server has generated its files, stop mc.service and merge the settings from server.properties.example into the server.properties under the mounted Minecraft data directory. Do not replace the full generated file with the example. Restart the service afterwards.

Attach to the console:

```bash
podman attach mc1
```

In the Minecraft console, add each known player and verify the list:

```text
whitelist add PLAYERNAME
whitelist list
```

Verify the exact server-side name for Bedrock players; Floodgate may add a prefix. Allowlist membership does not require operator rights. Detach with Ctrl+P, then Ctrl+Q to leave the container running.

## OCI and host networking

The Quadlet publishes the article's ports:

| Purpose | Protocol | Host/container port |
| --- | --- | --- |
| Java Edition | TCP | 25565 |
| Bedrock / Geyser | UDP | 19132 |
| Bedrock / Geyser | TCP | 19132 |

Match the required ports in the OCI Security List or NSG and the host firewall. Restrict source ranges where practical. A public subnet also needs the appropriate Internet Gateway route and public endpoint. Publishing container ports does not configure OCI ingress. SSH access is separate.

## Operations

```bash
systemctl --user status mc.service
journalctl --user -u mc.service -f
podman logs -f mc1
podman exec -it mc1 bash
```

After editing the installed Quadlet:

```bash
systemctl --user daemon-reload
systemctl --user restart mc.service
```

For a consistent backup, stop the service, copy the entire persistent Minecraft directory to storage outside this VM, then start the service again. Test restoration before relying on the backup. Never commit worlds, player lists, Floodgate keys or credentials to this public repository.

## Sources and scope

- [Full walkthrough and design notes on the blog](https://blog.dieschmitterlinge.de/cloud/minecraft-server-oracle-oci-free-tier-podman/)
- [Upstream container project: TheRemote/Legendary-Java-Minecraft-Geyser-Floodgate](https://github.com/TheRemote/Legendary-Java-Minecraft-Geyser-Floodgate)
- [Container image on Docker Hub](https://hub.docker.com/r/05jchambers/legendary-minecraft-geyser-floodgate)
- [Podman 4.9.3 Quadlet reference](https://docs.podman.io/en/v4.9.3/markdown/podman-systemd.unit.5.html)
- [OCI Always Free resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)

OCI Free Tier capacity and eligibility depend on the account and region. The article's VM size is not a promise of the resources available to a new account. This is a small homelab example, not an automatic provisioning tool or a large public-server deployment.
