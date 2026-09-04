# CUPS Print Server - Multi-Architecture Docker Image 🖨️🐳

![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/CaTeIM/cups/build.yml?branch=master&style=for-the-badge)
![Docker Hub Pulls](https://img.shields.io/docker/pulls/cateim/cups?style=for-the-badge)
![Docker Image Size](https://img.shields.io/docker/image-size/cateim/cups/latest?style=for-the-badge)

_[🇧🇷 Leia em Português](https://github.com/CaTeIM/cups/blob/master/README.pt-br.md)_

This is a multi-architecture Docker image of **[CUPS (Common Unix Printing System)](https://github.com/OpenPrinting/cups)**, built upon the latest **Ubuntu (Rolling)** and **Debian 13 (Trixie)** bases. The goal is to provide a print server with the latest CUPS versions, ready to use and easy to deploy in containerized environments.

## 📚 Source Code

This is an open-source project. The `Dockerfile`, startup script, and GitHub Actions build workflow are all available in the project repository.

➡️ **[GitHub Repository: CaTeIM/cups](https://github.com/CaTeIM/cups)**

## 🐳 Available Tags

This repository builds two image "tracks", Ubuntu and Debian. Tags come in two
classes, and the difference matters in production.

**Moving tags** always point at the newest published build of their track:

| Tag                                    | Distro base        | Points to                         |
| :------------------------------------- | :----------------- | :-------------------------------- |
| `latest`, `ubuntu`                     | Ubuntu (Rolling)   | newest Ubuntu build               |
| `debian`                               | Debian 13 (Trixie) | newest Debian build               |
| `[version]-ubuntu`, `[version]-debian` | either             | newest build of that CUPS version |

**Immutable tags** are cut once and never rewritten:

| Tag format                            | Example                     |
| :------------------------------------ | :-------------------------- |
| `[version]-[distro]-[YYYYMMDD].[run]` | `2.4.16-ubuntu-20260903.31` |

_💡 **Pin an immutable tag in production.** A moving tag is convenient, but it
changes under you; an immutable one is the only way to know exactly what is
running and to roll back to it. Browse the
[Tags tab](https://hub.docker.com/r/cateim/cups/tags) for the current list._

## ✨ Why use this image?

- ✅ **Always Up-to-Date**: Rebuilt from scratch every Sunday against the live
  Ubuntu Rolling and Debian 13 archives. A new image is published whenever any
  installed package changes version, not only when CUPS itself is updated, so
  OS security patches actually reach you. When a rebuild finds nothing new,
  nothing is published and the moving tags stay put.

- ✅ **Tells You What Changed**: Every published build records which packages
  moved, from which version to which, and the CVEs the maintainers cite in the
  package changelogs. It is written to the image itself as OCI annotations
  (`docker buildx imagetools inspect cateim/cups:latest --raw | jq .annotations`)
  and kept permanently in [`.github/builds/`](https://github.com/CaTeIM/cups/tree/master/.github/builds).

- ✅ **Multi-Distro**: Choose between an Ubuntu (`latest`, `2.x.x-ubuntu`) or Debian (`2.x.x-debian`) base, depending on your preference.

- 🔒 **Secure**: The build process includes applying all available security updates (`apt-get upgrade`).

- 🖨️ **Ready to Use**: Ships 19 print drivers (Brother, Epson, Canon, Samsung,
    Oki, Dymo, Lexmark, HP and more) plus `hplip` and the `openprinting-ppds` and
    `foomatic-db` PPD catalogues, so most printers are plug and play. Each driver
    is listed explicitly in the Dockerfile, so one disappearing from the distro
    breaks the build instead of quietly vanishing from the image.

- 🚀 **Multi-Architecture**: Built to run natively on `linux/amd64` (PCs, Intel/AMD Servers) and `linux/arm64` (Raspberry Pi, Orange Pi 5, etc.).

- 🔧 **Smart Configuration**: Features a startup script that sets up an admin user and prepares CUPS for remote access on the first run.

## ⚙️ How to Use (Example with `docker-compose.yml`)

The recommended way to use this image is with Portainer Stacks or `docker-compose`. Create a `docker-compose.yml` file with the following content.

> **Note:** For USB printers to work properly (detect out-of-paper, reconnection, etc.), mapping the `/run/udev` and `/run/dbus` volumes is **essential**.

```yaml
services:
  cups:
    # 'latest' tracks Ubuntu. In production, prefer an immutable tag:
    # cateim/cups:2.4.16-ubuntu-20260903.31
    image: cateim/cups:latest
    container_name: cups
    # Unrestricted access to host devices. Works everywhere, but is far more
    # than printing needs: see "Running without privileged" below.
    privileged: true
    restart: unless-stopped
    environment:
      # Password for the 'admin' user on the web interface. Required.
      - ADMIN_PASSWORD=your_strong_password
      - TZ=America/Sao_Paulo
    volumes:
      # --- Configuration and data ---
      - /srv/cups/config:/etc/cups
      - /srv/cups/logs:/var/log/cups
      - /srv/cups/spool:/var/spool/cups
      # --- Hardware and system (CRITICAL FOR USB) ---
      # Physical access to the USB ports. Keep it as a bind mount: it is what
      # makes hotplug work, because nodes created after the container started
      # appear through it.
      - /dev/bus/usb:/dev/bus/usb
      # Talks to the host system services (fixes ColorManager/DBus errors)
      - /run/dbus:/run/dbus:ro
      # Lets CUPS see hardware events (paper reload, cover open)
      - /run/udev:/run/udev:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "631:631"
    # 'bridge' with the port published is the portable choice. 'host' makes
    # network discovery (AirPrint/Bonjour) easier, at the cost of isolation.
    network_mode: bridge
    hostname: cups
    # Gives cupsd time to write job.cache and printers.conf before SIGKILL.
    stop_grace_period: 30s
    # CUPS rotates its own error_log; this only caps Docker's json-file, which
    # would otherwise grow without limit in a crash loop.
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

### ⚠️ Upgrading from an image built before 2026-09

One breaking change, and it only affects you if you used it: **the `sudo`
package and the `admin ALL=(ALL) NOPASSWD:ALL` rule were removed.** With both in
place, anyone who learned the web UI password at :631 became root inside the
container, and under `privileged: true` that is root on the host.

- `docker exec cups <command>` runs as root and is **unaffected**, which is how
  the documentation here has always told you to do it.
- `docker exec -u admin cups sudo <command>` no longer works. Drop the `-u admin`
  and the `sudo`.
- Web UI login is **unaffected**: it is authorised by membership in `lpadmin`,
  which the `admin` user still has.

Everything else in that release is additive: 18 printer drivers that the image
had always claimed to ship but never actually installed, and OCI provenance
metadata. Despite the extra drivers the image got about 3% smaller, because
build headers that nothing ever compiled against were dropped.

### 🔑 Administration

- To access the web interface, use the address: `https://<YOUR_SERVER_IP>:631`
- To access the **Administration** area, use the login `admin` and the password you defined in the `ADMIN_PASSWORD` variable.

## 🔒 Running without `privileged`

The compose example above uses `privileged: true` because it works everywhere,
but it is far more access than printing needs. Of the seven things `privileged`
grants, USB printing consumes exactly one: unrestricted device cgroup access.
The other six are attack surface.

To drop it, replace `privileged: true` with:

```yaml
    device_cgroup_rules:
      - 'c 189:* rmw'          # 189 is the usbfs major (include/linux/usb.h)
    security_opt:
      - no-new-privileges:true
    cap_drop: [ALL]
    cap_add: [SETGID, SETUID, KILL, CHOWN, DAC_OVERRIDE, FOWNER, NET_BIND_SERVICE]
```

Keep the `/dev/bus/usb` bind mount. It is what makes hotplug work: nodes created
after the container started show up through it, while a `devices:` entry is a
snapshot taken at creation time and breaks the first time the printer is power
cycled.

Why this is enough: udev gives printer nodes `root:lp` mode `0664`, and CUPS
backends run as `lp`, so the backend already opens the device read-write through
the group bits. Verify yours with:

```sh
stat -c '%n owner=%U group=%G mode=%a' /dev/bus/usb/BBB/DDD
```

If your printer does not get `group=lp` (some multifunction devices announce
themselves as storage rather than as the printer class), add a udev rule on the
**host** granting `GROUP="lp"` for your `idVendor`/`idProduct`, rather than going
back to `privileged`.

**Two traps worth knowing.** `device_cgroup_rules` is silently ignored while
`privileged: true` is set, so the two cannot be combined "just to be safe": the
rule becomes dead letter and the test proves nothing. And after switching, test
by **power cycling the printer** and printing again, not just by printing once.

## 🐛 Troubleshooting (USB and "Host-Based" Printers)

If you use USB printers that rely on host-loaded firmware (like HP LaserJet P1102, P1005, 1020 series, etc.), you might notice that **turning the printer off and on** causes CUPS to stop responding or leaves jobs as "Held".

This happens because the Container loses the USB device reference when the electrical connection drops.

### Definitive Solution (Host UDEV Rule)

For the Container to automatically reconnect whenever the printer is restarted or the cable is reconnected, create a `udev` rule on your host system (not in the container).

1.  Find your printer's ID with the `lsusb` command (e.g., `03f0:002a`).
2.  Create the `/etc/udev/rules.d/99-fix-cups-usb.rules` file with the content below (replacing with your ID):

```bash
# Automatically restarts the CUPS container upon detecting the printer connection
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="YOUR_VENDOR_ID", ATTR{idProduct}=="YOUR_PRODUCT_ID", RUN+="/usr/bin/docker restart cups"
```

3.  Reload the rules: `udevadm control --reload-rules && udevadm trigger`

### Operational Tip (Out of Paper)

If you run out of paper and the job is held, avoid turning off the printer.

1.  Load the paper.
2.  **Open and close the toner/cartridge cover.**
3.  The mapped `/run/udev` volume in the container will detect the event and CUPS will automatically resume printing.

---

_This project is not officially affiliated with OpenPrinting. All credit for CUPS goes to its respective developers._
