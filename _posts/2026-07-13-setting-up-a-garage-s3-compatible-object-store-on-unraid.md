---
title: Setting up a Garage S3‑Compatible Object Store on Unraid
date: 2026-07-13 18:48:00
tags:
  - unraid
  - garage
  - s3
  - docker
---
## Frustrations...

In the year of 2026, storage has become an expensive commodity among price increases that keep... well, increasing. I found myself hit with $30 a month in expenses with my Backblaze bucket which, while cheaper than competitors, is a lot more than I'm willing to spend for Minecraft server backups. Luckily, I had two Unraid servers sitting around and plenty of storage. I asked myself if I was willing to take on the risk of data ownership in order to not pay almost $360 a year in storage costs... and I was! 

In this article we walk through the process of turning an **Unraid** server into a lightweight, self‑hosted S3‑compatible object store using **Garage**. The stack runs inside Docker and stores data on fast, dedicated storage, entirely local.

## Prerequisites

- Unraid 6.x with Docker support.
- A writable directory for Garage data (e.g., `/mnt/user/data/garage`).
- `docker` or **Portainer** to deploy a stack.

## Preparation and defining the Storage Directory

```bash
mkdir -p /mnt/user/data/garage
# Create the marker file that Garage expects
sudo touch /mnt/user/data/garage/garage-marker
```
The marker is required for Garage to verify the data volume.

## Docker Stack (YAML)

I created this stack via Portainer:
```yaml
version: "3.8"
services:
  garage:
    image: dxflrs/garage:v2.3.0
    container_name: garage
    restart: always
    network_mode: host
    volumes:
      - /mnt/user/appdata/garage/meta:/var/lib/garage/meta
      - /mnt/user/data/garage:/var/lib/garage/data
      - /mnt/user/appdata/garage/garage.toml:/etc/garage.toml:ro
    environment:
      - RUST_LOG=garage=info
    command: /garage server --single-node
```
*`appdata/meta`* keeps the SQLite DB with keys and bucket definitions. The actual objects live in `/mnt/user/data/garage`.

## Deploying and Verification

```bash
docker stack deploy -c docker-compose.yml garage
# Check status
docker exec -it garage /garage status
```
At this point, I saw a healthy node. since I was doing this for the first time, I had to assign a layout:
```bash
NODE=$(docker exec -it garage /garage status | head -1 | awk '{print $1}')
docker exec -it garage /garage layout assign -z dc1 -c 10G $NODE
docker exec -it garage /garage layout apply --version 1
```
After a few seconds, `garage status` had reported the node as fully connected.

## Creating a Bucket and Key
```bash
docker exec -it garage /garage bucket create contentbox-s3

docker exec -it garage /garage key create contentbox-key
# Note down the secret that appears – it’s shown only once.

docker exec -it garage /garage bucket allow --read --write --owner contentbox-s3 --key contentbox-key
```
Now I had an S3‑compatible endpoint at `http://<unraid-ip>:3900` with region `garage`, bucket `contentbox-s3`, and the key/secret pair above.

## Testing with MinIO Client
```bash
mc alias set garage http://<unraid-ip>:3900 GKabc123... <secret>
mc ls garage/contentbox-s3
```
I replaced `GKabc123...` & `<secret>` with the values from step 4.

## Tying in access via Tailscale

Luckily, I already had Tailscale set up in my environment. Using Tailscale, I can access this vault from any of my endpoints using my server's magic DNS name, contentbox. I thought about hosting the endpoints over https, but I really didn't have a need to do so at the time I set this up. Maybe one day in the future I'll set that up for cloud-based applications, but not this time. 

## Mirroring the setup

In order to achieve some level of redundancy, I mirrored the setup with another Unraid box, giving me two S3 compatible endpoints. The second setup is stored off-site. I can keep the two in a one-way sync offsite. I manually cull the data from the second setup, but this gives me some level of safety against large file deletions. 

## Does Backblaze still have a place? 

Surprise! I am actually continuing to use Backblaze. At this point, it's serving as more of an archival capacity than anything else. I have plans to phase it out once I can get a tertiary box in place, but I'll have to wait for hard drive prices to cool down a bit. Maybe 2028 will be my year! 
