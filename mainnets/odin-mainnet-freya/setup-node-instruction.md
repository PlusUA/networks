# NODE SETUP INSTRUCTION

#### 1. Install Basic Packages
```bash:
# update the local package list and install any available upgrades 
sudo apt-get update && sudo apt upgrade -y 
# install toolchain and ensure accurate time synchronization 
sudo apt-get install make build-essential gcc git jq chrony -y
```

#### 2. Install Go
Follow the instructions [here](https://golang.org/doc/install) to install Go.

Alternatively, for Ubuntu LTS, you can do:
```bash:
wget https://golang.org/dl/go1.18.1.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.18.1.linux-amd64.tar.gz
```

Unless you want to configure in a non standard way, then set these in the `.profile` in the user's home (i.e. `~/`) folder.

```bash:
cat <<EOF >> ~/.profile
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export GO111MODULE=on
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
EOF
source ~/.profile
go version
```

Output should be: `go version go1.18.1 linux/amd64`

#### 3. Install cosmovisor from source

Clone git repo:
```bash
git clone https://github.com/cosmos/cosmos-sdk.git
cd cosmos-sdk
git checkout cosmovisor/v1.1.0
make cosmovisor
```

Move cosmovisor to GOPATH:
```bash
cp cosmovisor/cosmovisor ~/go/bin/cosmovisor
```

To confirm that the installation was successful, you can run:

```bash:
cosmovisor version
```
Output should be: `v1.1.0`

#### 4. Create folders for cosmovisor

Cosmovisor folder layout:
```
.odin
└── cosmovisor
    ├── current -> genesis or upgrades/<name>
    ├── genesis
    │   └── bin
    │       └── odind
    └── upgrades
        └── <name>
            ├── bin
            │   └── odind
            └── upgrade-info.json
```

Create folders .odin/cosmovisor/genesis/bin.

```bash
mkdir -p .odin/cosmovisor/genesis/bin
```

#### 5. Build new binary odind

Build binary odind:
```bash
git clone https://github.com/ODIN-PROTOCOL/odin-core.git

cd odin-core
git fetch --tags
git checkout v0.6.0

make all

cp ~/go/bin/odind ~/.odin/cosmovisor/genesis/bin/odind
```

#### 6. Download Snapshot
Go to the `.odin/data folder`
```bash
cd
cd .odin/data
```

NOTE! Your system should have lz4 installed, for debian-based linux systems run the command
```bash
apt-get install lz4
```

Download the snapshot
```bash
SNAP_NAME=$(curl -s https://mercury-nodes.net/odin-snapshot/ | egrep -o ">odin.*tar.lz4" | tail -1 | tr -d ">"); \
wget -O - https://mercury-nodes.net/odin-snapshot/${SNAP_NAME} | lz4 -d | tar xf -
```
#### 7. Setup Unit/Daemon file

```bash
# 1. create daemon file
touch /etc/systemd/system/cosmovisor.service

# 2. run:
cat <<EOF >> /etc/systemd/system/cosmovisor.service
[Unit]
Description=Odin Cosmovisor Daemon
After=network-online.target

[Service]
User=<USER>
ExecStart=/home/<USER>/go/bin/cosmovisor run start
Restart=on-failure
RestartSec=3
LimitNOFILE=infinity

Environment="DAEMON_HOME=/home/<USER>/.odin"
Environment="DAEMON_NAME=odind"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"

[Install]
WantedBy=multi-user.target
EOF

# 3. reload the daemon
systemctl daemon-reload

# 4. enable service - this means the service will start up 
# automatically after a system reboot
systemctl enable cosmovisor.service

# 5. stop old odind service
systemctl stop odin.service

# 6. start daemon
systemctl start cosmovisor.service
```

In order to watch the service run, you can do the following:
```
journalctl -u cosmovisor.service -f
```
