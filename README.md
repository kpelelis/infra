# Home Infra

This is the my home infrastructure using docker and compose. For the
time being it is just a load balancer/reverse proxy (Traefik) with
a media server.

## Prerequisites

* Docker (currently running on 20.10)
* Some DNS work (see below)
* (Optional for HTTPS) Cloudflare account

## Setup

### Configure environment

Copy the example file

```bash
$ cp .env.example .env
```

1. Set your domain

This will work even if you don't own the actual domain, but you are required to have it
registered if you want HTTPS to work.

2. Set your Cloudflare credentials

HTTPS over DNS can work with other [providers](https://doc.traefik.io/traefik/https/acme/#providers) as well but you need to
configure accordingly. For Cloudflare you will need the email and the Global API key.
If you want to move your domain to cloudflare, all you have to do go to the provider that
you have your domain registered at and point the DNS servers to the one cloudflare uses.

See [more](https://developers.cloudflare.com/registrar/get-started/transfer-domain-to-cloudflare/)

3. Set your paths

For now this setup only works with one volume/path so make sure the paths you choose are
mounted on a disk with enough space

### Configure your DNS

Since we are using a reverse proxy for the routing, the services will be accessible
through hostnames. It is best if you can configure your DNS to point a wildcard hostname
to the machine running the load balancer. Ideally this must be setup in your router but
if you can't do that, you can run a simple DNS server on your machine (like dnsmasq) and
through the settings of your router, make DHCP advertise this machine as the DNS authority
of the network

### Run Traefik

Traefik can be run with

```bash
$ docker compose -f lb-dc.yml -p lb --env-file .env up
```

You can watch the logs with

```bash
$ docker compose -p lb logs
```

If everything worked, if you visit `https://lb.${your-domain}` you should see Traefik's UI
(notice that https should work)

### Run the Media Server

Create the volume directory

```bash
$ mkdir volumes
```

```bash
$ docker compose -f media-dc.yml -p --env-file .env media up
```

Similarly, you can watch the logs with

```bash
$ docker compose -p media logs
```
#### Service Index

The following table contains all the services that should be running after a successful deployment.
Each one is accessible at `https://${subdomain}.${your-domain}`

| Name     | Description              | Subdomain |
|----------|--------------------------|-----------|
| Heimdall | Dashboard/App Launcher   | index     |
| Jackett  | Torrent tracker/RSS feed | trackers  |
| Sonarr   | Series tracker           | series    |
| Radarr   | Movies tracker           | movies    |
| Deluge   | Torrent Client           | torrents  |
| Jellyfin | Media Player             | jellyfin  |

### Setup services

#### Deluge

1. Visit `http://torrents.${your-domain}`
2. Default password is `deluge`. I recommend changing that
3. You need to setup to download to `/downloads/incomplete` and move the completed downloads to `/downloads/complete`
   from preferences option in the top nav bar.

#### Jackett

1. Visit `http://trakcers.${your-domain}`
2. Setup some trackers (the legal ones)

#### Sonarr/Radarr

1. Add root folder at `Settings > Media Management` (`/series` and `/movies` respectively)
2. These two are forks so they have almost identical setups. Visit either of the URLs and first connect to deluge
   `Settings > Download Clients > + > Deulge`
3. Connect Jackett. For each indexer in your Jackett installation you need to copy the Tornzab URL and use it with
   `Settings > Indexers > + > Tornzab`

#### Jellyfin

If you visit the URL you should get a nice installation wizard that would walk you through it. The path for movies
is `/media/movies` and for series it is `/media/series`

<b>Recommended Settings</b>

Dashboard > Users > Turn of Audio Transcoding
Use a pin for local network devices

#### (Optional) Add Links to Heimdall

If you want a dashboard with links, then you can visit `https://index.${your-domain}` and setup the URLs accordingly

### Wireguard

Provided that you have a wireguard network setup, you can expose this infrastructure directly by only routing traffic
to Traefik container. We assume the following the wireguard subnet is 10.2.0.0/24 and the IP of the machine running
the infrastructure is 10.2.0.2. You can set these accordingly. We need to do the following steps.

1. Make our public DNS provider (Cloudflare) resolve to the private wireguard IP for the wildcard domain. In this
   case add an A record for `*.<your-domain>` that resolves to 10.2.0.2
2. Create a new bridge network on docker with a different subnet
   ```bash
   $ docker network create --subnet 10.2.1.0/24 wireguard
   ```
   The compose file will try to bind the IP `10.2.1.2` so if you want a different subnet you need to change this
   as well
3. Add iptables rules to [DNAT](http://linux-ip.net/html/nat-dnat.html) the traffic coming from wireguard to the
   docker network

   ```bash
   # Accept traffic both on port 80 and 443
   $ sudo iptables -A FORWARD -i wg0 -p tcp --dports 80,443 --dst 10.2.0.2/32 -j ACCEPT
   # DNAT traffic to docker network
   $ sudo iptables -A PREROUTING -t nat -i wg0 -p tcp --dport 80 -j DNAT --dst 10.2.0.2/32 --to 10.3.0.2:80
   $ sudo iptables -A PREROUTING -t nat -i wg0 -p tcp --dport 443 -j DNAT --dst 10.2.0.2/32 --to 10.3.0.2:443
   ```
   To make these rules persistent you must use a tool like iptables-persistent

### TODO

- [ ] Migrate to Kubernetes/Nomad
- [X] Wireguard
