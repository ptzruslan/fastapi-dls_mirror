# FastAPI-DLS

Minimal Delegated License Service (DLS).

> [!warning] Branch support
> FastAPI-DLS Version 1.x supports up to **`17.x`** releases. \
> FastAPI-DLS Version 2.x is backwards compatible to `17.x` and supports **`18.x`** releases in combination
> with [gridd-unlock-patcher](https://git.collinwebdesigns.de/oscar.krause/gridd-unlock-patcher).
> Other combinations of FastAPI-DLS and Driver-Branches may work but are not tested.

> [!note] Compatibility
> Compatibility tested with official NLS 2.0.1, 2.1.0, 3.1.0, 3.3.1, 3.4.0. For Driver compatibility
> see [compatibility matrix](#vgpu-software-compatibility-matrix).

This service can be used without internet connection.
Only the clients need a connection to this service on configured port.

**Official Links**

* https://git.collinwebdesigns.de/oscar.krause/fastapi-dls (Private Git)
* https://gitea.publichub.eu/oscar.krause/fastapi-dls (Public Git)
* https://hub.docker.com/r/collinwebdesigns/fastapi-dls (Docker-Hub `collinwebdesigns/fastapi-dls:latest`)

*All other repositories are forks! (which  is no bad - just for information and bug reports)*

[Releases & Release Notes](https://git.collinwebdesigns.de/oscar.krause/fastapi-dls/-/releases)

**Further Reading**

* [NVIDIA vGPU Guide](https://gitlab.com/polloloco/vgpu-proxmox) - This document serves as a guide to install NVIDIA vGPU host drivers on the latest Proxmox VE version
* [vgpu_unlock](https://github.com/DualCoder/vgpu_unlock) - Unlock vGPU functionality for consumer-grade Nvidia GPUs.
* [vGPU_Unlock Wiki](https://docs.google.com/document/d/1pzrWJ9h-zANCtyqRgS7Vzla0Y8Ea2-5z2HEi4X75d2Q) - Guide for `vgpu_unlock`
* [Proxmox 8 vGPU in VMs and LXC Containers](https://medium.com/@dionisievldulrincz/proxmox-8-vgpu-in-vms-and-lxc-containers-4146400207a3) - Install *Merged Drivers* for using in Proxmox VMs and LXCs
* [Proxmox All-In-One Installer Script](https://wvthoog.nl/proxmox-vgpu-v3/) - Also known as `proxmox-installer.sh`

---

[TOC]

# Setup (Service)

**System requirements**

- 256mb ram
- 4gb hdd
- *maybe IPv6 must be disabled*

Tested with Ubuntu 22.10 (EOL!) (from Proxmox templates), actually its consuming 100mb ram and 750mb hdd.

**Prepare your system**

- Make sure your timezone is set correct on you fastapi-dls server and your client

This guide does not show how to install vGPU host drivers! Look at the official documentation packed with the driver
releases.

## Docker

Docker-Images are available here for Intel (x86), AMD (amd64) and ARM (arm64):

- [Docker-Hub](https://hub.docker.com/repository/docker/collinwebdesigns/fastapi-dls): `collinwebdesigns/fastapi-dls:latest`
- [GitLab-Registry](https://git.collinwebdesigns.de/oscar.krause/fastapi-dls/container_registry): `registry.git.collinwebdesigns.de/oscar.krause/fastapi-dls:latest`

The images include database drivers for `postgres`, `mariadb` and `sqlite`.

**Run this on the Docker-Host**

```shell
WORKING_DIR=/opt/docker/fastapi-dls/cert
mkdir -p $WORKING_DIR
cd $WORKING_DIR
# create ssl certificate for integrated webserver (uvicorn) - because clients rely on ssl
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout  $WORKING_DIR/webserver.key -out $WORKING_DIR/webserver.crt
```

**Start container**

To test if everything is set up properly you can start container as following:

```shell
docker volume create dls-db
docker run -e DLS_URL=`hostname -i` -e DLS_PORT=443 -p 443:443 -v $WORKING_DIR:/app/cert -v dls-db:/app/database collinwebdesigns/fastapi-dls:latest
```

**Docker-Compose / Deploy stack**

See [`examples`](examples) directory for more advanced examples (with reverse proxy usage).

> Adjust `REQUIRED` variables as needed

```yaml
version: '3.9'

x-dls-variables: &dls-variables
  TZ: Europe/Berlin # REQUIRED, set your timezone correctly on fastapi-dls AND YOUR CLIENTS !!!
  DLS_URL: localhost # REQUIRED, change to your ip or hostname
  DLS_PORT: 443
  LEASE_EXPIRE_DAYS: 90  # 90 days is maximum
  DATABASE: sqlite:////app/database/db.sqlite
  DEBUG: false

services:
  dls:
    image: collinwebdesigns/fastapi-dls:latest
    restart: always
    environment:
      <<: *dls-variables
    ports:
      - "443:443"
    volumes:
      - /opt/docker/fastapi-dls/cert:/app/cert
      - dls-db:/app/database
    logging:  # optional, for those who do not need logs
      driver: "json-file"
      options:
        max-file: 5
        max-size: 10m
            
volumes:
  dls-db:
```

## Debian / Ubuntu / macOS (manual method using `git clone` and python virtual environment)

Tested on `Debian 11 (bullseye)`, `Debian 12 (bookworm)` and `macOS Ventura (13.6)`, Ubuntu may also work.
**Please note that setup on macOS differs from Debian based systems.**

**Make sure you are logged in as root.**

**Install requirements**

```shell
apt-get update && apt-get install git python3-venv python3-pip
```

**Install FastAPI-DLS**

```shell
WORKING_DIR=/opt/fastapi-dls
mkdir -p $WORKING_DIR
cd $WORKING_DIR
git clone https://git.collinwebdesigns.de/oscar.krause/fastapi-dls .
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
deactivate
chown -R www-data:www-data $WORKING_DIR
```

**Create keypair and webserver certificate**

```shell
WORKING_DIR=/opt/fastapi-dls/app/cert
mkdir -p $WORKING_DIR
cd $WORKING_DIR
# create ssl certificate for integrated webserver (uvicorn) - because clients rely on ssl
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout  $WORKING_DIR/webserver.key -out $WORKING_DIR/webserver.crt
chown -R www-data:www-data $WORKING_DIR
```

**Test Service**

This is only to test whether the service starts successfully.

```shell
cd /opt/fastapi-dls/app
sudo -u www-data /opt/fastapi-dls/venv/bin/uvicorn main:app --app-dir=/opt/fastapi-dls/app
# or
su - www-data -c "/opt/fastapi-dls/venv/bin/uvicorn main:app --app-dir=/opt/fastapi-dls/app"
```

**Create config file**

> Adjust `DLS_URL` as needed (accessing from LAN won't work with 127.0.0.1)

```shell
mkdir /etc/fastapi-dls
cat <<EOF >/etc/fastapi-dls/env
DLS_URL=127.0.0.1
DLS_PORT=443
LEASE_EXPIRE_DAYS=90
DATABASE=sqlite:////opt/fastapi-dls/app/db.sqlite

EOF
```

**Create service**

```shell
cat <<EOF >/etc/systemd/system/fastapi-dls.service
[Unit]
Description=Service for fastapi-dls
After=network.target

[Service]
User=www-data
Group=www-data
AmbientCapabilities=CAP_NET_BIND_SERVICE
WorkingDirectory=/opt/fastapi-dls/app
EnvironmentFile=/etc/fastapi-dls/env
ExecStart=/opt/fastapi-dls/venv/bin/uvicorn main:app \\
  --env-file /etc/fastapi-dls/env \\
  --host \$DLS_URL --port \$DLS_PORT \\
  --app-dir /opt/fastapi-dls/app \\
  --ssl-keyfile /opt/fastapi-dls/app/cert/webserver.key \\
  --ssl-certfile /opt/fastapi-dls/app/cert/webserver.crt \\
  --proxy-headers
Restart=always
KillSignal=SIGQUIT
Type=simple
NotifyAccess=all

[Install]
WantedBy=multi-user.target

EOF
```

Now you have to run `systemctl daemon-reload`. After that you can start service
with `systemctl start fastapi-dls.service` and enable autostart with `systemctl enable fastapi-dls.service`.

## openSUSE Leap (manual method using `git clone` and python virtual environment)

Tested on `openSUSE Leap 15.4`, openSUSE Tumbleweed may also work.

**Install requirements**

```shell
zypper in -y python310 python3-virtualenv python3-pip
```

**Install FastAPI-DLS**

```shell
BASE_DIR=/opt/fastapi-dls
SERVICE_USER=dls
mkdir -p ${BASE_DIR}
cd ${BASE_DIR}
git clone https://git.collinwebdesigns.de/oscar.krause/fastapi-dls .
python3.10 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
deactivate
useradd -r ${SERVICE_USER} -M -d /opt/fastapi-dls
chown -R ${SERVICE_USER} ${BASE_DIR}
```

**Create keypair and webserver certificate**

```shell
CERT_DIR=${BASE_DIR}/app/cert
SERVICE_USER=dls
mkdir ${CERT_DIR}
cd ${CERT_DIR}
# create ssl certificate for integrated webserver (uvicorn) - because clients rely on ssl
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout  ${CERT_DIR}/webserver.key -out ${CERT_DIR}/webserver.crt
chown -R ${SERVICE_USER} ${CERT_DIR}
```

**Test Service**

This is only to test whether the service starts successfully.

```shell
BASE_DIR=/opt/fastapi-dls
SERVICE_USER=dls
cd ${BASE_DIR}
sudo -u ${SERVICE_USER} ${BASE_DIR}/venv/bin/uvicorn main:app --app-dir=${BASE_DIR}/app
# or
su - ${SERVICE_USER} -c "${BASE_DIR}/venv/bin/uvicorn main:app --app-dir=${BASE_DIR}/app"
```

**Create config file**

> Adjust `DLS_URL` as needed (accessing from LAN won't work with 127.0.0.1)

```shell
BASE_DIR=/opt/fastapi-dls
cat <<EOF >/etc/fastapi-dls/env
DLS_URL=127.0.0.1
DLS_PORT=443
LEASE_EXPIRE_DAYS=90
DATABASE=sqlite:///${BASE_DIR}/app/db.sqlite

EOF
```

**Create service**

```shell
BASE_DIR=/opt/fastapi-dls
SERVICE_USER=dls
cat <<EOF >/etc/systemd/system/fastapi-dls.service
[Unit]
Description=Service for fastapi-dls vGPU licensing service
After=network.target

[Service]
User=${SERVICE_USER}
AmbientCapabilities=CAP_NET_BIND_SERVICE
WorkingDirectory=${BASE_DIR}/app
EnvironmentFile=/etc/fastapi-dls/env
ExecStart=${BASE_DIR}/venv/bin/uvicorn main:app \\
  --env-file /etc/fastapi-dls/env \\
  --host \$DLS_URL --port \$DLS_PORT \\
  --app-dir ${BASE_DIR}/app \\
  --ssl-keyfile ${BASE_DIR}/app/cert/webserver.key \\
  --ssl-certfile ${BASE_DIR}/app/cert/webserver.crt \\
  --proxy-headers
Restart=always
KillSignal=SIGQUIT
Type=simple
NotifyAccess=all

[Install]
WantedBy=multi-user.target

EOF
```

Now you have to run `systemctl daemon-reload`. After that you can start service
with `systemctl start fastapi-dls.service` and enable autostart with `systemctl enable fastapi-dls.service`.

## Debian / Ubuntu (using `dpkg` / `apt`)

Packages are available here:

- [GitLab-Registry](https://git.collinwebdesigns.de/oscar.krause/fastapi-dls/-/packages)

Successful tested with (**LTS Version**):

- **Debian 12 (Bookworm)** (EOL: June 06, 2026)
- *Ubuntu 22.10 (Kinetic Kudu)* (EOL: July 20, 2023)
- *Ubuntu 23.04 (Lunar Lobster)* (EOL: January 2024)
- *Ubuntu 23.10 (Mantic Minotaur)* (EOL: July 2024)
- **Ubuntu 24.04 (Noble Numbat)** (EOL: Apr 2029)

Not working with:

- Debian 11 (Bullseye) and lower (missing `python-jose` dependency)
- Debian 13 (Trixie) (missing `python-jose` dependency)
- Ubuntu 22.04 (Jammy Jellyfish) (not supported as for 15.01.2023 due to [fastapi - uvicorn version missmatch](https://bugs.launchpad.net/ubuntu/+source/fastapi/+bug/1970557))
- Ubuntu 24.10 (Oracular Oriole) (missing `python-jose` dependency)

**Run this on your server instance**

First go to [GitLab-Registry](https://git.collinwebdesigns.de/oscar.krause/fastapi-dls/-/packages) and select your
version. Then you have to copy the download link of the `fastapi-dls_X.Y.Z_amd64.deb` asset.

```shell
apt-get update
FILENAME=/opt/fastapi-dls.deb
wget -O $FILENAME <download-url>
dpkg -i $FILENAME
apt-get install -f --fix-missing
```

Start with `systemctl start fastapi-dls.service` and enable autostart with `systemctl enable fastapi-dls.service`.
Now you have to edit `/etc/fastapi-dls/env` as needed.

## ArchLinux (using `pacman`)

**Shout out to `samicrusader` who created build file for ArchLinux!**

Packages are available here:

- [GitLab-Registry](https://git.collinwebdesigns.de/oscar.krause/fastapi-dls/-/packages)

```shell
pacman -Sy
FILENAME=/opt/fastapi-dls.pkg.tar.zst

curl -o $FILENAME <download-url>
# or
wget -O $FILENAME <download-url>

pacman -U --noconfirm fastapi-dls.pkg.tar.zst
```

Start with `systemctl start fastapi-dls.service` and enable autostart with `systemctl enable fastapi-dls.service`.
Now you have to edit `/etc/default/fastapi-dls` as needed.

## unRAID

1. Download [this xml file](.UNRAID/FastAPI-DLS.xml)
2. Put it in /boot/config/plugins/dockerMan/templates-user/
3. Go to Docker page, scroll down to `Add Container`, click on Template list and choose `FastAPI-DLS`
4. Open terminal/ssh, follow the instructions in overview description
5. Setup your container `IP`, `Port`, `DLS_URL` and `DLS_PORT`
6. Apply and let it boot up

*Unraid users must also make sure they have Host access to custom networks enabled if unraid is the vgpu guest*.

Continue [here](#unraid-guest) for docker guest setup.

## NixOS

Tanks to [@mrzenc](https://github.com/mrzenc) for [fastapi-dls-nixos](https://github.com/mrzenc/fastapi-dls-nixos).

> [!note] Native NixOS-Package
> There is a [pull request](https://github.com/NixOS/nixpkgs/pull/358647) which adds fastapi-dls into nixpkgs.

## Let's Encrypt Certificate (optional)

If you're using installation via docker, you can use `traefik`. Please refer to their documentation.

Note that port 80 must be accessible, and you have to install `socat` if you're using `standalone` mode.

```shell
acme.sh --issue -d example.com \
  --cert-file /etc/fastapi-dls/webserver.donotuse.crt \
  --key-file /etc/fastapi-dls/webserver.key \
  --fullchain-file /etc/fastapi-dls/webserver.crt \
  --reloadcmd "systemctl restart fastapi-dls.service"
```

After first success you have to replace `--issue` with `--renew`.

# Configuration

| Variable               | Default                                | Usage                                                                                                |
|------------------------|----------------------------------------|------------------------------------------------------------------------------------------------------|
| `DEBUG`                | `false`                                | Toggles `fastapi` debug mode                                                                         |
| `DLS_URL`              | `localhost`                            | Used in client-token to tell guest driver where dls instance is reachable                            |
| `DLS_PORT`             | `443`                                  | Used in client-token to tell guest driver where dls instance is reachable                            |
| `CERT_PATH`            | `None`                                 | Path to a Directory where generated Certificates are stored. Defaults to `/<app-dir>/cert`.          |
| `TOKEN_EXPIRE_DAYS`    | `1`                                    | Client auth-token validity (used for authenticate client against api, **not `.tok` file!**)          |
| `LEASE_EXPIRE_DAYS`    | `90`                                   | Lease time in days                                                                                   |
| `LEASE_RENEWAL_PERIOD` | `0.15`                                 | The percentage of the lease period that must elapse before a licensed client can renew a license \*1 |
| `DATABASE`             | `sqlite:///db.sqlite`                  | See [official SQLAlchemy docs](https://docs.sqlalchemy.org/en/14/core/engines.html)                  |
| `CORS_ORIGINS`         | `https://{DLS_URL}`                    | Sets `Access-Control-Allow-Origin` header (comma separated string) \*2                               |
| `SITE_KEY_XID`         | `00000000-0000-0000-0000-000000000000` | Site identification uuid                                                                             |
| `INSTANCE_REF`         | `10000000-0000-0000-0000-000000000001` | Instance identification uuid                                                                         |
| `ALLOTMENT_REF`        | `20000000-0000-0000-0000-000000000001` | Allotment identification uuid                                                                        |

\*1 For example, if the lease period is one day and the renewal period is 20%, the client attempts to renew its license
every 4.8 hours. If network connectivity is lost, the loss of connectivity is detected during license renewal and the
client has 19.2 hours in which to re-establish connectivity before its license expires.

\*2 Always use `https`, since guest-drivers only support secure connections!

# Setup (Client)

**The token file has to be copied! It's not enough to C&P file contents, because there can be special characters.**

This guide does not show how to install vGPU guest drivers! Look at the official documentation packed with the driver
releases.

## Linux

Download *client-token* and place it into `/etc/nvidia/ClientConfigToken`:

```shell
curl --insecure -L -X GET https://<dls-hostname-or-ip>/-/client-token -o /etc/nvidia/ClientConfigToken/client_configuration_token_$(date '+%d-%m-%Y-%H-%M-%S').tok
# or
wget --no-check-certificate -O /etc/nvidia/ClientConfigToken/client_configuration_token_$(date '+%d-%m-%Y-%H-%M-%S').tok https://<dls-hostname-or-ip>/-/client-token
```

Restart `nvidia-gridd` service:

```shell
service nvidia-gridd restart
```

Check licensing status:

```shell
nvidia-smi -q | grep "License"
```

Output should be something like:

```text
vGPU Software Licensed Product
    License Status                    : Licensed (Expiry: YYYY-M-DD hh:mm:ss GMT)
```

Done. For more information check [troubleshoot section](#troubleshoot).

## Windows

**Power-Shell** (run as administrator!)

Download *client-token* and place it into `C:\Program Files\NVIDIA Corporation\vGPU Licensing\ClientConfigToken`:

```shell
curl.exe --insecure -L -X GET https://<dls-hostname-or-ip>/-/client-token -o "C:\Program Files\NVIDIA Corporation\vGPU Licensing\ClientConfigToken\client_configuration_token_$($(Get-Date).tostring('dd-MM-yy-hh-mm-ss')).tok"
```

Restart `NvContainerLocalSystem` service:

```Shell
Restart-Service NVDisplay.ContainerLocalSystem
```

Check licensing status:

```shell
& 'nvidia-smi' -q  | Select-String "License"
```

Output should be something like:

```text
vGPU Software Licensed Product
    License Status                    : Licensed (Expiry: YYYY-M-DD hh:mm:ss GMT)
```

Done. For more information check [troubleshoot section](#troubleshoot).

## unRAID Guest

1. Make sure you create a folder in a linux filesystem (BTRFS/XFS/EXT4...), I recommend `/mnt/user/system/nvidia` (this is where docker and libvirt preferences are saved, so it's a good place to have that)
2. Edit the script to put your `DLS_IP`, `DLS_PORT` and `TOKEN_PATH`, properly 
3. Install `User Scripts` plugin from *Community Apps* (the Apps page, or google User Scripts Unraid if you're not using CA)
4. Go to `Settings > Users Scripts > Add New Script`
5. Give it a name  (the name must not contain spaces preferably)
6. Click on the *gear icon* to the left of the script name then edit script
7. Paste the script and save
8. Set schedule to `At First Array Start Only`
9. Click on Apply 

# API Endpoints

<details>
  <summary>show</summary>

**`GET /`**

Redirect to `/-/readme`.

**`GET /-/health`**

Status endpoint, used for *healthcheck*.

**`GET /-/config`**

Shows current runtime environment variables and their values.

**`GET /-/config/root-certificate`**

Returns the Root-Certificate Certificate which is used. This is required for patching `nvidia-gridd` on 18.x releases.

**`GET /-/readme`**

HTML rendered README.md.

**`GET /-/manage`**

Shows a very basic UI to delete origins or leases.

**`GET /-/origins?leases=false`**

List registered origins.

| Query Parameter | Default | Usage                                |
|-----------------|---------|--------------------------------------|
| `leases`        | `false` | Include referenced leases per origin |

**`DELETE /-/origins`**

Deletes all origins and their leases.

**`GET /-/leases?origin=false`**

List current leases.

| Query Parameter | Default | Usage                               |
|-----------------|---------|-------------------------------------|
| `origin`        | `false` | Include referenced origin per lease |

**`DELETE /-/lease/{lease_ref}`**

Deletes an lease.

**`GET /-/client-token`**

Generate client token, (see [installation](#installation)).

**Others**

There are many other internal api endpoints for handling authentication and lease process.
</details>

# Troubleshoot / Debug

**Please make sure that fastapi-dls and your guests are on the same timezone!**

Maybe you have to disable IPv6 on the machine you are running FastAPI-DLS.

## Docker

Logs are available with `docker logs <container>`. To get the correct container-id use `docker container ls` or `docker ps`.

## Linux

Logs are available with `journalctl -u nvidia-gridd -f`.

## Windows

Logs are available in `C:\Users\Public\Documents\Nvidia\LoggingLog.NVDisplay.Container.exe.log`.

# Known Issues

## Generic

### `Failed to acquire license from <ip> (Info: <license> - Error: The allowed time to process response has expired)`

- Did your timezone settings are correct on fastapi-dls **and your guest**?
- Did you download the client-token more than an hour ago?

Please download a new client-token. The guest have to register within an hour after client-token was created.

### `jose.exceptions.JWTError: Signature verification failed.`

- Did you recreate any certificate or keypair?

Then you have to download a **new** client-token on each of your guests.

## Linux

### Invalid HTTP request

This error message: `uvicorn.error:Invalid HTTP request received.` can be ignored.

- Ref. https://github.com/encode/uvicorn/issues/441

<details>
  <summary>Log example</summary>

```
WARNING:uvicorn.error:Invalid HTTP request received.
Traceback (most recent call last):
  File "/usr/lib/python3/dist-packages/uvicorn/protocols/http/h11_impl.py", line 129, in handle_events
    event = self.conn.next_event()
  File "/usr/lib/python3/dist-packages/h11/_connection.py", line 485, in next_event
    exc._reraise_as_remote_protocol_error()
  File "/usr/lib/python3/dist-packages/h11/_util.py", line 77, in _reraise_as_remote_protocol_error
    raise self
  File "/usr/lib/python3/dist-packages/h11/_connection.py", line 467, in next_event
    event = self._extract_next_receive_event()
  File "/usr/lib/python3/dist-packages/h11/_connection.py", line 409, in _extract_next_receive_event
    event = self._reader(self._receive_buffer)
  File "/usr/lib/python3/dist-packages/h11/_readers.py", line 84, in maybe_read_from_IDLE_client
    raise LocalProtocolError("no request line received")
h11._util.RemoteProtocolError: no request line received
```

</details>

## Windows

### Required cipher on Windows Guests (e.g. managed by domain controller with GPO)

It is required to enable `SHA1` (`TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA_P521`)
in [windows cipher suite](https://learn.microsoft.com/en-us/windows-server/security/tls/manage-tls).

### Multiple Display Container LS Instances

On Windows on some machines there are running two or more instances of `NVIDIA Display Container LS`. This causes a
problem on licensing flow. As you can see in the logs below, there are two lines with `NLS initialized`, each prefixed
with `<1>` and `<2>`. So it is possible, that *daemon 1* fetches a valid license through dls-service, and *daemon 2*
only
gets a valid local license.

<details>
  <summary>Log example</summary>

**Display-Container-LS**

```
Tue Dec 20 17:25:11 2022:<1>:NLS initialized
Tue Dec 20 17:25:12 2022:<2>:NLS initialized
Tue Dec 20 17:25:16 2022:<1>:Valid GRID license not found. GPU features and performance will be restricted. To enable full functionality please configure licensing details.
Tue Dec 20 17:25:17 2022:<1>:License acquired successfully. (Info: 192.168.178.110, NVIDIA RTX Virtual Workstation; Expiry: 2022-12-21 16:25:16 GMT)
Tue Dec 20 17:25:17 2022:<2>:Valid GRID license not found. GPU features and performance will be restricted. To enable full functionality please configure licensing details.
Tue Dec 20 17:25:38 2022:<2>:License acquired successfully from local trusted store. (Info: 192.168.178.110, NVIDIA RTX Virtual Workstation; Expiry: 2022-12-21 16:25:16 GMT)
```

**fastapi-dls**

``` 
> [  origin  ]: 41720000-FA43-4000-9472-0000E8660000: {'candidate_origin_ref': '41720000-FA43-4000-9472-0000E8660000', 'environment': {'fingerprint': {'mac_address_list': ['5E:F0:79:E6:DE:E1']}, 'hostname': 'PC-Windows', 'ip_address_list': ['2003:a:142e:c800::1cc', 'fdfe:7fcd:e30f:40f5:ad5c:e67b:49a6:cfb3', 'fdfe:7fcd:e30f:40f5:6409:db1c:442b:f90b', 'fe80::a32e:f736:8988:fe45', '192.168.178.110'], 'guest_driver_version': '527.41', 'os_platform': 'Windows 10 Pro', 'os_version': '10.0.19045', 'host_driver_version': '525.60.12', 'gpu_id_list': ['1E3010DE-133210DE'], 'client_platform_id': '00000000-0000-0000-0000-000000000113', 'hv_platform': 'Unknown', 'cpu_sockets': 1, 'physical_cores': 8}, 'registration_pending': False, 'update_pending': False}
> [  origin  ]: 41720000-FA43-4000-9472-0000E8660000: {'candidate_origin_ref': '41720000-FA43-4000-9472-0000E8660000', 'environment': {'fingerprint': {'mac_address_list': ['5E:F0:79:E6:DE:E1']}, 'hostname': 'PC-Windows', 'ip_address_list': ['2003:a:142e:c800::1cc', 'fdfe:7fcd:e30f:40f5:ad5c:e67b:49a6:cfb3', 'fdfe:7fcd:e30f:40f5:6409:db1c:442b:f90b', 'fe80::a32e:f736:8988:fe45', '192.168.178.110'], 'guest_driver_version': '527.41', 'os_platform': 'Windows 10 Pro', 'os_version': '10.0.19045', 'host_driver_version': '525.60.12', 'gpu_id_list': ['1E3010DE-133210DE'], 'client_platform_id': '00000000-0000-0000-0000-000000000113', 'hv_platform': 'Unknown', 'cpu_sockets': 1, 'physical_cores': 8}, 'registration_pending': False, 'update_pending': False}
> [   code   ]: 41720000-FA43-4000-9472-0000E8660000: {'code_challenge': 'bTwcOn17SD5mtwmFdKDgufnceGXeGYcnFfMHqmjtReo', 'origin_ref': '41720000-FA43-4000-9472-0000E8660000'}
> [   code   ]: 41720000-FA43-4000-9472-0000E8660000: {'code_challenge': 'FCVDfgKmgr+lyvSpOxr4fZnDZv8VrNtNEAZPUuLAr7A', 'origin_ref': '41720000-FA43-4000-9472-0000E8660000'}
> [   auth   ]: 41720000-FA43-4000-9472-0000E8660000 (bTwcOn17SD5mtwmFdKDgufnceGXeGYcnFfMHqmjtReo): {'auth_code': 'eyJhbGciOiJSUzI1NiIsImtpZCI6IjAwMDAwMDAwLTAwMDAtMDAwMC0wMDAwLTAwMDAwMDAwMDAwMCIsInR5cCI6IkpXVCJ9.eyJpYXQiOjE2NzE1NTcwMzMsImV4cCI6MTY3MTU1NzkzMywiY2hhbGxlbmdlIjoiYlR3Y09uMTdTRDVtdHdtRmRLRGd1Zm5jZUdYZUdZY25GZk1IcW1qdFJlbyIsIm9yaWdpbl9yZWYiOiJiVHdjT24xN1NENW10d21GZEtEZ3VmbmNlR1hlR1ljbkZmTUhxbWp0UmVvIiwia2V5X3JlZiI6IjAwMDAwMDAwLTAwMDAtMDAwMC0wMDAwLTAwMDAwMDAwMDAwMCIsImtpZCI6IjAwMDAwMDAwLTAwMDAtMDAwMC0wMDAwLTAwMDAwMDAwMDAwMCJ9.m5M4h9HRYWkItHEdYGApJVM7TgBH0qyDXCxPkaG2-Km5SviRMk0_3er5Myjq3rYGlr88JBviA07Pc3cr7fV-tDAXaSGalxLNfFtVRcnzqbtgnkodep1PHRUXYkiQgfaJ36m02zZucu4qMyYfQTpZ_-x67eycFKyN9T9cRJ4PYFe5W_6_zjzz6D0qeLACDhXt4ns980URttKfn2vACE8gPP5-EC-7lSY1g1mAWJKB_X9OlYRFE2mkCxnde6z5I2qmCXE_awimkigjo5LYvDcjCz60QDsOD2Ojgz4Y9xgjPbKnup4c2orKTWLUfT8_o4toKbaSfuLzPtD-41b3E8NqHQ', 'code_verifier': 'NCkAAB0+AACEHAAAIAAAAEoWAACAGAAArGwAAOkkAABfTgAAK0oAADFiAAANXAAAHzwAAKg4AAC/GwAAkxsAAEJHAABiDwAAaC8AAFMYAAAOLAAAFUkAAEheAAALOwAAHmwAAIJtAABpKwAArmsAAGM8AABnVwAA5FkAAP8mAAA'}
> [   auth   ]: 41720000-FA43-4000-9472-0000E8660000 (FCVDfgKmgr+lyvSpOxr4fZnDZv8VrNtNEAZPUuLAr7A): {'auth_code': 'eyJhbGciOiJSUzI1NiIsImtpZCI6IjAwMDAwMDAwLTAwMDAtMDAwMC0wMDAwLTAwMDAwMDAwMDAwMCIsInR5cCI6IkpXVCJ9.eyJpYXQiOjE2NzE1NTcwMzQsImV4cCI6MTY3MTU1NzkzNCwiY2hhbGxlbmdlIjoiRkNWRGZnS21ncitseXZTcE94cjRmWm5EWnY4VnJOdE5FQVpQVXVMQXI3QSIsIm9yaWdpbl9yZWYiOiJGQ1ZEZmdLbWdyK2x5dlNwT3hyNGZabkRadjhWck50TkVBWlBVdUxBcjdBIiwia2V5X3JlZiI6IjAwMDAwMDAwLTAwMDAtMDAwMC0wMDAwLTAwMDAwMDAwMDAwMCIsImtpZCI6IjAwMDAwMDAwLTAwMDAtMDAwMC0wMDAwLTAwMDAwMDAwMDAwMCJ9.it_UKCHLLd25g19zqryZ6_ePrkHljXJ3uX-hNdu-pcmnYD9ODOVl2u5bRxOrP6S2EUO4WLZIuvLOhbFBUHfZfXFRmmCv4NDJoZx36Qn6zszePK9Bngej40Qf8Wu3JGXMVrwfC6WNW6WFeUT-s9jos5e1glFk_E3ZhOYQjXljWOcfcNvZ-PVJFBi5OzyQqLuL43GQH_PSF66N2gq0OyKgxTvg2q6SzGD3YAxsbjy2mD0YOUv8pW8Dr_9L4hmnNHg2DdM_lCwmy4qIBaDkAQDq8VCw1-4RcXROiLlYwhvHRalsXnmREPXaOUiUrr8rrCX8jgc7Fcd1uhY5jnouWbwEAg', 'code_verifier': 'tFAAAKQSAAAqOQAAhykAANJxAAA9PQAAyFwAALNsAAB/VQAA4GQAAB5fAAA2JgAApWIAAKMeAAB3YwAAggQAAPsEAAAuAgAAblIAABR/AAAfAgAAenoAAKZ3AABUTQAA5CQAANkTAAC8JwAAvUQAAO0yAAA3awAAegIAAD1iAAA'}
> [  leases  ]: 41720000-FA43-4000-9472-0000E8660000 (bTwcOn17SD5mtwmFdKDgufnceGXeGYcnFfMHqmjtReo): found 0 active leases
> [  leases  ]: 41720000-FA43-4000-9472-0000E8660000 (FCVDfgKmgr+lyvSpOxr4fZnDZv8VrNtNEAZPUuLAr7A): found 0 active leases
> [  create  ]: 41720000-FA43-4000-9472-0000E8660000 (bTwcOn17SD5mtwmFdKDgufnceGXeGYcnFfMHqmjtReo): create leases for scope_ref_list ['1e9335d0-049d-48b2-b719-e551c859f9f9']
```

in comparison to linux

**nvidia-grid.service**

```
Dec 20 17:53:32 ubuntu-grid-server nvidia-gridd[10354]: vGPU Software package (0)
Dec 20 17:53:32 ubuntu-grid-server nvidia-gridd[10354]: Ignore service provider and node-locked licensing
Dec 20 17:53:32 ubuntu-grid-server nvidia-gridd[10354]: NLS initialized
Dec 20 17:53:32 ubuntu-grid-server nvidia-gridd[10354]: Acquiring license. (Info: 192.168.178.110; NVIDIA RTX Virtual Workstation)
Dec 20 17:53:34 ubuntu-grid-server nvidia-gridd[10354]: License acquired successfully. (Info: 192.168.178.110, NVIDIA RTX Virtual Workstation; Expiry: 2022-12-21 16:53:33 GMT)
```

**fastapi-dls**

```
> [  origin  ]: B210CF72-FEC7-4440-9499-1156D1ACD13A: {'candidate_origin_ref': 'B210CF72-FEC7-4440-9499-1156D1ACD13A', 'environment': {'fingerprint': {'mac_address_list': ['d6:30:d8:de:46:a7']}, 'hostname': 'ubuntu-grid-server', 'ip_address_list': ['192.168.178.114', 'fdfe:7fcd:e30f:40f5:d430:d8ff:fede:46a7', '2003:a:142e:c800::642', 'fe80::d430:d8ff:fede:46a7%ens18'], 'guest_driver_version': '525.60.13', 'os_platform': 'Ubuntu 20.04', 'os_version': '20.04.5 LTS (Focal Fossa)', 'host_driver_version': '525.60.12', 'gpu_id_list': ['1E3010DE-133210DE'], 'client_platform_id': '00000000-0000-0000-0000-000000000105', 'hv_platform': 'LINUX_KVM', 'cpu_sockets': 1, 'physical_cores': 16}, 'registration_pending': False, 'update_pending': False}
> [   code   ]: B210CF72-FEC7-4440-9499-1156D1ACD13A: {'code_challenge': 'hYSKI4kpZcWqPatM5Sc9RSCuzMeyz2piTmrRQKnnHro', 'origin_ref': 'B210CF72-FEC7-4440-9499-1156D1ACD13A'}
> [   auth   ]: B210CF72-FEC7-4440-9499-1156D1ACD13A (hYSKI4kpZcWqPatM5Sc9RSCuzMeyz2piTmrRQKnnHro): {'auth_code': 'eyJhbGciOiJSUzI1NiIsImtpZCI6IjAwMDAwMDAwLTAwMDAtMDAwMC0wMDAwLTAwMDAwMDAwMDAwMCIsInR5cCI6IkpXVCJ9.eyJpYXQiOjE2NzE1NTUyMTIsImV4cCI6MTY3MTU1NjExMiwiY2hhbGxlbmdlIjoiaFlTS0k0a3BaY1dxUGF0TTVTYzlSU0N1ek1leXoycGlUbXJSUUtubkhybyIsIm9yaWdpbl9yZWYiOiJoWVNLSTRrcFpjV3FQYXRNNVNjOVJTQ3V6TWV5ejJwaVRtclJRS25uSHJvIiwia2V5X3JlZiI6IjAwMDAwMDAwLTAwMDAtMDAwMC0wMDAwLTAwMDAwMDAwMDAwMCIsImtpZCI6IjAwMDAwMDAwLTAwMDAtMDAwMC0wMDAwLTAwMDAwMDAwMDAwMCJ9.G5GvGEBNMUga25EeaJeAbDk9yZuLBLyj5e0OzVfIjS70UOvDb-SvLSEhBv9vZ_rxjTtaWGQGK0iK8VnLce8KfqsxZzael6B5WqfwyQiok3WWIaQarrZZXKihWhgF49zYAIZx_0js1iSjoF9-vNSj8zan7j-miOCOssfPzGgfJqvWNnhR6_2YkCQgJssHMjGT1QxaJBZDVOuvY0ND7r6jxlS_Xze1nWtau1mtC6bu2hM8cxbYUtM-XOC8welCZ8ZOCKkutmVix0weV3TVNfR5vuBUz1QS6B9YC8R-eVVBhN2hl4j7kGZLmZ4TpyLViYEUVZsqGBayVIPeN2BhtqTO9g', 'code_verifier': 'IDiWUb62sjsNYuU/YtZ5YJdvvxE70gR9vEPOQo9+lh/DjMt1c6egVQRyXB0FAaASNB4/ME8YQjGQ1xUOS7ZwI4tjHDBbUXFBvt2DVu8jOlkDmZsNeI2IfQx5HRkz1nRIUlpqUC/m01gAQRYAuR6dbUyrkW8bq9B9cOLSbWzjJ0E'}
> [  leases  ]: B210CF72-FEC7-4440-9499-1156D1ACD13A (hYSKI4kpZcWqPatM5Sc9RSCuzMeyz2piTmrRQKnnHro): found 0 active leases
> [  create  ]: B210CF72-FEC7-4440-9499-1156D1ACD13A (hYSKI4kpZcWqPatM5Sc9RSCuzMeyz2piTmrRQKnnHro): create leases for scope_ref_list ['f27e8e79-a662-4e35-a728-7ea14341f0cb']
```

</details>

### Error on releasing leases on shutdown (can be ignored and/or fixed with reverse proxy)

The driver wants to release current leases on shutting down windows. This endpoint needs to be a http endpoint.
The error message can safely be ignored (since we have no license limitation :P) and looks like this:

<details>
  <summary>Log example</summary>

```
<1>:NLS initialized
<1>:License acquired successfully. (Info: 192.168.178.110, NVIDIA RTX Virtual Workstation; Expiry: 2023-3-30 23:0:22 GMT)
<0>:Failed to return license to 192.168.178.110 (Error: Generic network communication failure)
<0>:End Logging
```

#### log with nginx as reverse proxy (see [docker-compose-http-and-https.yml](examples/docker-compose-http-and-https.yml))

```
<1>:NLS initialized
<2>:NLS initialized
<1>:Valid GRID license not found. GPU features and performance will be fully degraded. To enable full functionality please configure licensing details.
<1>:License acquired successfully. (Info: 192.168.178.33, NVIDIA RTX Virtual Workstation; Expiry: 2023-1-4 16:48:20 GMT)
<2>:Valid GRID license not found. GPU features and performance will be fully degraded. To enable full functionality please configure licensing details.
<2>:License acquired successfully from local trusted store. (Info: 192.168.178.33, NVIDIA RTX Virtual Workstation; Expiry: 2023-1-4 16:48:20 GMT)
<2>:End Logging
<1>:End Logging
<0>:License returned successfully. (Info: 192.168.178.33)
<0>:End Logging
```

</details>

# vGPU Software Compatibility Matrix

<details>
  <summary>Show Table</summary>

Successfully tested with this package versions.

| FastAPI-DLS Version | vGPU Suftware | Driver Branch | Linux vGPU Manager | Linux Driver | Windows Driver |  Release Date |      EOL Date |
|---------------------|:-------------:|:-------------:|--------------------|--------------|----------------|--------------:|--------------:|
| `2.x`               |    `18.1`     |   **R570**    | `570.133.08`       | `570.133.07` | `572.83`       |    April 2025 |    March 2026 |
|                     |    `18.0`     |   **R570**    | `570.124.03`       | `570.124.06` | `572.60`       |    March 2025 |    March 2026 |
| `1.x` & `2.x`       |    `17.6`     |   **R550**    | `550.163.02`       | `550.63.01`  | `553.74`       |    April 2025 |     June 2025 |
|                     |    `17.5`     |               | `550.144.02`       | `550.144.03` | `553.62`       |  January 2025 |               |
|                     |    `17.4`     |               | `550.127.06`       | `550.127.05` | `553.24`       |  October 2024 |               |
|                     |    `17.3`     |               | `550.90.05`        | `550.90.07`  | `552.74`       |     July 2024 |               |
|                     |    `17.2`     |               | `550.90.05`        | `550.90.07`  | `552.55`       |     June 2024 |               |
|                     |    `17.1`     |               | `550.54.16`        | `550.54.15`  | `551.78`       |    March 2024 |               |
|                     |    `17.0`     |   **R550**    | `550.54.10`        | `550.54.14`  | `551.61`       | February 2024 |               |
| `1.x`               |    `16.10`    |   **R535**    | `535.247.02`       | `535.247.01` | `539.28`       |    April 2025 |     July 2026 |
| `1.x`               |    `15.4`     |   **R525**    | `525.147.01`       | `525.147.05` | `529.19`       |     June 2023 | December 2023 |
| `1.x`               |    `14.4`     |   **R510**    | `510.108.03`       | `510.108.03` | `514.08`       | December 2022 | February 2023 |

</details>

- https://docs.nvidia.com/grid/index.html
- https://docs.nvidia.com/grid/gpus-supported-by-vgpu.html

*To get the latest drivers, visit Nvidia or search in Discord-Channel `GPU Unlocking` (Server-ID: `829786927829745685`)
on channel `licensing`

# Credits

Thanks to vGPU community and all who uses this project and report bugs.

Special thanks to:

- `samicrusader` who created build file for **ArchLinux**
- `cyrus` who wrote the section for **openSUSE**
- `midi` who wrote the section for **unRAID**
- `polloloco` who wrote the *[NVIDIA vGPU Guide](https://gitlab.com/polloloco/vgpu-proxmox)*
- `DualCoder` who creates the `vgpu_unlock` functionality [vgpu_unlock](https://github.com/DualCoder/vgpu_unlock)
- `Krutav Shah` who wrote the [vGPU_Unlock Wiki](https://docs.google.com/document/d/1pzrWJ9h-zANCtyqRgS7Vzla0Y8Ea2-5z2HEi4X75d2Q/)
- `Wim van 't Hoog` for the [Proxmox All-In-One Installer Script](https://wvthoog.nl/proxmox-vgpu-v3/)
- `mrzenc` who wrote [fastapi-dls-nixos](https://github.com/mrzenc/fastapi-dls-nixos)
- `electricsheep49` who wrote [gridd-unlock-patcher](https://git.collinwebdesigns.de/oscar.krause/gridd-unlock-patcher)

And thanks to all people who contributed to all these libraries!
