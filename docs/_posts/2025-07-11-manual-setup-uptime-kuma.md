---
layout: post
title: "Setting up uptime kuma and running via systemd"
date: 2025-07-10 23:24:00 -0000
categories: runbooks
---

1. Setup a lxc container running debian 12
2. access lxc shell, install and configure uptime kuma (with dependencies satisfied, using systemd instead of pm2)

```bash
installing npm and node.js via nodesource installer:
sudo apt-get install -y curl
curl -fsSL https://deb.nodesource.com/setup_20.x -o nodesource_setup.sh
sudo -E bash nodesource_setup.sh
sudo apt-get install -y nodejs
node -v
npm -v

installing git:
sudo apt install -y git-all

installing pm2: (if not using pm2 not needed)
npm install pm2 -g

installing uptime kuma and starting via pm2:

groupadd --system uptime
useradd -r --no-create-home -g uptime --shell /usr/sbin/nologin uptime
mkdir -p /opt/uptime-kuma

git clone https://github.com/louislam/uptime-kuma.git /opt/uptime-kuma
cd /opt/uptime-kuma
npm install
npm run build (make sure your container has enough RAM to build it)

chown -R uptime:uptime /opt/uptime-kuma

pm2 based approach (on uptime user):

npm install pm2 -g && pm2 install pm2-logrotate

su - uptime

pm2 start server/server.js --name uptime-kuma

pm2 monit #check status

pm2 save && pm2 startup #add to startup

systemd based approach:

touch /etc/systemd/system/uptime-kuma.service

[Unit]
Description=Uptime-Kuma - A free and open source uptime monitoring solution
Documentation=https://github.com/louislam/uptime-kuma
After=network.target

[Service]
Type=simple
User=uptime
WorkingDirectory=/opt/uptime-kuma
ExecStart=/usr/bin/node server/server.js
Restart=on-failure

[Install]
WantedBy=multi-user.target

systemctl daemon-reload
systemctl enable --now uptime-kuma

curl http://localhost:3001
systemctl status uptime-kuma.service

```
