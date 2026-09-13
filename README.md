![Base](logo.webp)

# Base

Base is a rollup built on Ethereum.

## Learn More

- Visit the [docs](https://docs.base.org) for information on how to:
    - [Run a node](https://docs.base.org/base-chain/node-operators/run-a-base-node)

## Install Binaries

Use [`baseup`](baseup/README.md) to install the GitHub release binaries mainly used for downloading snapshots, check link above for more info:

```bash
curl -fsSL https://raw.githubusercontent.com/base/base/main/baseup/install | bash
```

# Deploying Base Reth Node Guide

## Minimum Requirements
- Modern Multicore CPU 8+ cores (recommended 16+ cores)
- 32GB RAM (64 - 128GB Recommended)
- Storage: 4+ TB NVMe SSD drive locally attached (RAID0) for pruned node (2 * current chain size + snapshot size + 20% buffer) (to accommodate future growth)
- Docker and Docker Compose
- Ideally Ubuntu 24.04 LTS x64

**NOTE**: DigitalOcean is not suitable since it has network attached storage blocks

# Host Machine Setup
- update pkgs
```bash
sudo apt upgrade
```
- install docker and docker compose (check docker docs)

- ideally configure ssh to prevent root and password login (only through ssh pubkey login):
```bash
sudo nano /etc/ssh/sshd_config
```
and then:
```txt
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
```

- update firewall:
```bash
# set firewall
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow OpenSSH
sudo ufw allow 80
sudo ufw allow 443
sudo ufw allow 18545 # rpc http
sudo ufw allow 18546 # rpc websocket
sudo ufw allow 9222 # p2p for sync
sudo ufw allow 30303 # p2p for sync
sudo ufw allow 13100 # loki rpc metrics
sudo ufw allow 19090 # prometheus node metrics
```

- add system swap:
```bash
sudo fallocate -l 64G /path/to/swapfile
sudo chmod 600 /path/to/swapfile
sudo mkswap /path/to/swapfile
sudo swapon /path/to/swapfile

# add to fstab so it is persisted through reboot
echo '/path/to/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# optionally configure swappiness, more value means more favorable towards swap, less mean more favorable towards RAM
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# check that swap is added
free -h
```

- instal nginx, this allows us the set configurations for rpc such as api keys, rate limiting, etc:
```bash
sudo apt install -y nginx
```

- configure nginx default config:
```sh
sudo nano /etc/nginx/nginx.conf
```

then enable `multi_accept on;` and `worker_connections 2048;` on events block and enable or add the following lines to `http {}` block under `Basic Settings`:
```txt
map_hash_bucket_size 128;
map_hash_max_size 4096;
```

example:
```txt
events {
	worker_connections 2048;
	multi_accept on;
}

http {

	##
	# Basic Settings
	##
	map_hash_bucket_size 128;
	map_hash_max_size 4096;
	sendfile on;
	tcp_nopush on;
	types_hash_max_size 2048;
	# server_tokens off;

	# server_names_hash_bucket_size 64;
	# server_name_in_redirect off;

    .
    .
    .
}
```

configure `rotate` field in `logrotate.d` for nginx log rotation.
```sh
sudo nano /etc/logrotate.d/nginx
```

example:
```txt
/var/log/nginx/*.log {
	daily
	missingok
	rotate 14 # this is set to 14 days log rotation
	compress
	delaycompress
	notifempty
	create 0640 www-data adm
	sharedscripts
	prerotate
		if [ -d /etc/logrotate.d/httpd-prerotate ]; then \
			run-parts /etc/logrotate.d/httpd-prerotate; \
		fi \
	endscript
	postrotate
		invoke-rc.d nginx rotate >/dev/null 2>&1
	endscript
}
```

## Repo Setup
Start by cloning this repo and go to the repo directory:
```sh
git clone https://github.com/rainlanguage/base.node.git /path/to/repo
cd /path/to/repo
```

- generate api key (as many as u wish) with running:
```sh
./gen-apikey.sh add <name> "<comment>"
```
example:
```sh
./gen-apikey.sh add myapp "this is my app's key" # it will output a key like: "myapp-5ae25e03b774cc1c031e994edf09b10d03052d9a5621"
```

delete api keys with:
```sh
./gen-apikey.sh list # first list the numbered keys:
# 1) user1-ace5ccb821873eaf7227c379a0407dd0f6eb9fb247e1 1; # no comment
# 2) myapp-5ae25e03b774cc1c031e994edf09b10d03052d9a5621; # this is my app's key
# 3) user2-ab947bfd4c1ed927466cc401918219ef48addca50ae3 1; # user 2 comment

./gen-apikeys.sh del 2 # delete the key with its number in the list
```

- enable and link the nginx rpc config (running from node repo dir):
```bash
sudo ln -s $(pwd)/nginx/rpc.conf /etc/nginx/sites-enabled/
sudo ln -s $(pwd)/nginx/api_keys.map /etc/nginx/
```

- obtain a TLS certificate for the public HTTPS endpoint. `rpc.conf` serves RPC
  over HTTPS on `:443` (so the api key is not sent in cleartext), and that server
  block references a Let's Encrypt cert, so `nginx -t` below will fail until the
  cert exists. Point a public DNS A record (`base-rpc.rainlang.xyz` by default in
  `rpc.conf`) at the node's public IP, then issue the cert via the default `:80`
  site and install a renewal hook that reloads nginx:
```sh
sudo apt install -y certbot
sudo certbot certonly --webroot -w /var/www/html -d base-rpc.rainlang.xyz \
  --agree-tos -m devops@rainlang.xyz --no-eff-email
echo -e '#!/bin/sh\nsystemctl reload nginx' \
  | sudo tee /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```
  If you use a different domain, update `server_name` and the two `ssl_certificate`
  paths in the `:443` block of `nginx/rpc.conf` to match.

and test the config:
```sh
sudo nginx -t
sudo nginx -s reload
```

## Starting the Node
- configure the mainnet env varibales in `.env.mainnet`, those are L1, L1 Beacon RPC and JWT auth 32bytes length secrete:
```env
# [REQUIRED] L1 CONFIGURATION
# ---------------------------
# Replace these values with your L1 (Ethereum) node endpoints
BASE_NODE_L1_ETH_RPC=<your-preferred-l1-rpc>
BASE_NODE_L1_BEACON=<your-preferred-l1-beacon>

# ENGINE CONFIGURATION
# --------------------
BASE_NODE_L2_ENGINE_AUTH_RAW=<your-jwt-32-bytes-hex-secrete>
```

- configure the general `.env` (specify snapshot download and extraction directories if you are starting the node from a snapshot):
```env
# Tag of image to pull, defaults to latest if unspecified
NODE_TAG=

# path to dir that keeps all EL node data
HOST_DATA_DIR=./reth-data

# RSS config for forwarding Base Chain Status to telegram
RSS_BOT_URL="https://base-l2.statuspage.io/history.rss"
RSS_BOT_TG_BOT_TOKEN=
RSS_BOT_TG_CHANNEL_ID=
# polling interval for feed check in hour, default 15 mins
RSS_BOT_POLL_INTERVAL=0.25
```

- start the node with:
```sh
sudo docker compose up -d
```

if you have already downloaded snapshot, place it at your configured directory that you specified in `.env`.

- start rss bot to subscribe to Base status for updates, outages, etc:
```sh
sudo docker compose -f docker-compose-rss.yml up -d
```

## Monitor Sync Process
- priodically check container logs:
```bash
sudo docker logs --since 100s execution
sudo docker logs --since 100s consensus
```

check pipeline stages progress:
```bash
sudo docker logs execution | grep -i "pipeline_stages"
sudo docker logs consensus | grep -i "pipeline_stages"
```

and check for errors in them:

```bash
sudo docker logs execution | grep -i "error"
sudo docker logs consensus | grep -i "error"
```

- monitor VM (if available) panel for system usage, sometime sudden drops in RAM usage may indicate that one of the container has unexpectedly restarted, you can confirm that with:
```bash
# check the `CREATED` time vs `STATUS` time of each container
sudo docker ps
```

or you can check sysem for `oom` errors:
```bash
sudo dmesg | grep -i kill
```

potential disk bottleneck can be monitored via:
```sh
iostat -xz 1
```

- Sync process may take some time (days), so be patiant but keep an eye on it and watch for possible errors and/or container restarts.

- You can check if the L2 blocks are increasing by:
```sh
echo $(($(curl -X POST http://127.0.0.1:8545 \
-H "Content-Type: application/json" \
--data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":["latest"],"id":1}' | jq -r .result )))
```

- You can check L1 sync with (this is the mai sync command in base docs, but it can get left behind during sync process, so plz be patient):
```bash
echo Latest synced block behind by: $((($(date +%s)-$( \
curl -d '{"id":0,"jsonrpc":"2.0","method":"optimism_syncStatus"}' \
-H "Content-Type: application/json" http://localhost:7545 | \
jq -r .result.unsafe_l2.timestamp))/60)) minutes
```

# Using The Node RPC
- HTTP URL
`http://node-host-public-ip:18545/<your-api-key>`

- WS URL
`ws://node-host-public-ip:18546/<your-api-key>`

## Setting up Grafana Monitor Dashboard
The node metrics exposed on ports specified in `docker-compose.yml` can be configured to be with grafana dahsboard templates on `./grafana`, for more info on how to set it up, you can visit Grafana website.
