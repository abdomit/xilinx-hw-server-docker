# Xilinx hw_server & xsdb Docker Image

This Docker image allows you to run Xilinx's `hw_server` as well as the Xilinx debugger `xsdb` in a Docker container.
Using QEmu user-space emulation, these tools can also run on embedded devices like a Raspberry Pi or the Ultra-scale boards themself.

Differences from the original repo:
* Updated Vivado 2023.2 toolchain
* Update README.md with up-to-date instructions

![Setup](./docs/setup.png)

## Tested configuration

- Raspberry Pi 3B+ with 64-bit Raspberry Pi OS (Debian Bookworm)
- FPGA board Digilent CMOD A7

## Usage

### Docker

```bash
docker run \
   --rm \
   --privileged \
   --volume /dev/bus/usb:/dev/bus/usb \
   --publish 3121:3121 \
   --detach \
   ghcr.io/stv0g/hw_server:v2023.2
```

### Docker-compose

Copy the `docker-compose.yml` file from this repo to your target's working directory and run the following command to start the container.

```bash
docker-compose up -d
```

## Running on non x86_64 systems

Install docker:

```bash
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install -y docker-compose
```

Run hw_server:

```bash
# Enable qemu-user emulation support for running amd64 Docker images
docker run --rm --privileged aptman/qus -s -- -p x86_64

# Run the hw_server with docker
docker run --restart unless-stopped --privileged --volume /dev/bus/usb:/dev/bus/usb --publish 3121:3121 --detach ghcr.io/sst/hw_server:2023.2

# - OR -

# Run the hw_server with docker-compose (copy the docker-compose.yml to your working dir first)
docker-compose up -d
```

### Optional: Enable QEmu userspace emulation at system startup

```bash
cat > /etc/systemd/system/qus.service <<EOF
[Unit]
Description=Start docker container for QEMU x86 emulation
Requires=docker.service
After=docker.service

[Service]
Type=forking
TimeoutStartSec=infinity
Restart=no
ExecStart=/usr/bin/docker run --rm --privileged aptman/qus -s -- -p x86_64

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now qus
```

The above steps in conjunction with the docker restart policy will make your `hw_server` container start whenever your system is booted.

If you want to let systemd manage the hw_server you can use also do the following:

```bash
cat > /etc/systemd/system/hw_server.service <<EOF
[Unit]
Description=Starts xilinx hardware server
After=docker-qemu-interpreter.service
Requires=docker.service

[Service]
Type=forking
PIDFile=/run/hw_server.pid
TimeoutStartSec=infinity
Restart=always
ExecStart=/usr/bin/docker run --rm --name hw_server --privileged  --platform linux/amd64 --volume /dev/bus/usb:/dev/bus/usb --publish 3121:3121 --detach ghcr.io/stv0g/hw_server:v2023.2
ExecStartPost=/bin/bash -c '/usr/bin/docker inspect -f '{{.State.Pid}}' hw_server | tee /run/hw_server.pid'
ExecStop=/usr/bin/docker stop hw_server

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now hw_server
```

## Building your own image

It is advised to build the docker image on a native x86_64 system, and to send the newly created image on the target device using docker `save` and `load` commands.

1. Download the _Vivado Lab Solutions_ Linux installer to the current directory.
   - **Do not extract it!**
   - E.g. `Vivado_Lab_Lin_2023.2_1013_2256.tar.gz`
2. Build the image with the [build.sh](build.sh) script:

   ```bash
   ./build.sh
   ```

### Note concerning Accept EULA

Depending on the Vivado version, you have to agree WebTalk (e. g. Version 2020.1) in the [Dockerfile](Dockerfile) or omit it (e. g. Version 2023.2). If this particular line does not match, Vivado installation will fail!

#### For 2021.2 and future versions

```bash
...
--agree XilinxEULA,3rdPartyEULA \
...
```

#### For 2020.1 and probably older versions

```bash
...
--agree XilinxEULA,3rdPartyEULA,WebTalkTerms \
...
```
