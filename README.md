# Pi-hole v6 + cloudflared in Docker

## Summary

This is a **baseline setup of Pi-hole and cloudflared** using Docker. It
assumes that you already have a gateway/router with a
**separate DHCP and NTP server**. If you want Pi-hole to handle DHCP,
additional configuration is needed.

This setup uses Cloudflare's **cloudflared**, a tunneling daemon, to connect
Pi-hole to DNS-over-HTTPS (DoH) providers like
[1.1.1.1](https://one.one.one.one/) or [Quad9](https://quad9.net/), providing
enhanced privacy and security for DNS queries. It follows the official
[Pi-hole cloudeflared guide](https://docs.pi-hole.net/guides/dns/cloudflared/)
but adapts it for Pihole v6 and Docker Compose.

In this setup, **cloudflared does not have its own network interface**;
instead, it runs using **Pi-hole's network stack**
(`network_mode: service:pihole`). This means:

- **cloudflared is not exposed to the host network** but can still handle DNS-over-HTTPS queries.
- **Pi-hole forwards all upstream DNS queries** to `127.0.0.1#5053`, where cloudflared handles DoH lookups.
- **No additional networking configurations are needed** for cloudflared.

## Prerequisites

Before you begin, ensure you are running:

- A **Debian or Debian-based Linux distribution** (Ubuntu, Raspberry Pi OS, etc.)
- [**Docker** installed](https://docs.docker.com/engine/install/)

## Step 1: Create the Directory Structure for Bind Mounts

Before downloading the repository, set up the necessary directories for your
**bind mounts**.

Run the following commands:

```bash
mkdir -p ~/docker/pihole-cloudflared
sudo mkdir -p /srv/docker/pihole-cloudflared/pihole/etc-pihole
sudo mkdir -p /srv/docker/pihole-cloudflared/pihole/etc-dnsmasq.d
sudo chown -R $USER:$USER /srv/docker
chmod -R 755 /srv/docker
cd ~/docker/pihole-cloudflared
```

### **What These Commands Do**

- `mkdir -p ~/docker/pihole-cloudflared`: Creates a working directory in your home folder.
- `sudo mkdir -p /srv/docker/...`: Creates **bind mounts** for Pi-hole.
- `sudo chown -R $USER:$USER /srv/docker`: Ensures **your user owns the folders**.
- `chmod -R 755 /srv/docker`: Sets **read/write permissions** for better access.

## Step 2: Download the Repository

You can download the latest version of this repository using **`wget`** or **`curl`**:

**Option 1: Using `wget`**

```bash
wget https://github.com/kaczmar2/pihole-cloudflared/archive/refs/heads/main.tar.gz
tar -xzf main.tar.gz --strip-components=1
```

**Option 2: Using `curl`**

```bash
curl -L -o main.tar.gz https://github.com/kaczmar2/pihole-cloudflared/archive/refs/heads/main.tar.gz
tar -xzf main.tar.gz --strip-components=1
```

The `--strip-components=1` flag ensures the contents are extracted directly into
`~/docker/pihole-cloudflared` instead of creating an extra subdirectory.

**Note**: This setup uses cloudflared as a DNS-over-HTTPS proxy, providing
enhanced privacy and security for DNS queries.

Optional: Remove the archive after extraction:

```sh
rm main.tar.gz
```

## Step 3: Configure DoH Provider

By default, the `.env` file is configured to use **Cloudflare's DoH service**
(<https://cloudflare-dns.com/dns-query>). If you want to use a different provider,
edit the `DOH_PRIMARY` variable in the `.env` file before starting the
containers. See the [DNS-over-HTTPS Providers table](#common-dns-over-https-providers)
at the bottom for other options.

## Step 4: Start the Pi-hole + cloudflared Containers

Now, deploy the Pi-hole and cloudflared services using:

```bash
docker compose up -d
```

## Step 5: Verify cloudflared is Working

To confirm cloudflared is resolving queries correctly, run the following
commands **in the pihole container**:

Open a `bash` shell in the container:

```bash
docker exec -it pihole /bin/bash
```

Test that cloudflared is operational:

```bash
dig pi-hole.net @127.0.0.1 -p 5053
```

The first query may be quite slow, but subsequent queries should be fairly
quick.

## Step 6: Set the Pi-hole Admin Password

To set the pihole web admin password, run the following commands
**in the pihole container**, if you're not already there from the previous
step (`docker exec -it pihole /bin/bash`):

```bash
pihole setpassword 'mypassword'
```

Get the hashed password from `pihole.toml`:

```bash
cat /etc/pihole/pihole.toml | grep -w pwhash
```

`exit` the container and copy the hashed password into your `.env` file on the
host.

Make sure to enclose the value in single quotes (`''`).

```bash
WEB_PWHASH='$BALLOON-SHA256$v=1$s=1024,t=32$pZCbBIUH/Ew2n144eLn3vw==$vgej+obQip4DvSmNlywD0LUHlsHcqgLdbQLvDscZs78='
```

Uncomment the `FTLCONF_webserver_api_pwhash` environment variable in
`docker-compose.yml`:

```bash
FTLCONF_webserver_api_pwhash: ${WEB_PWHASH}
```

Restart the containers:

```bash
docker compose down && docker compose up -d
```

## Step 7: Access the Pi-hole Web Interface

Once running, open your web browser and go to:

```bash
http://<your-server-ip>/admin/
```

Login using the password you set.

## Step 8: Secure with SSL (Optional)

For enhanced security, see my other guides on **configuring SSL encryption**
for the Pi-hole web interface.

- [Pi-hole v6 + Docker: Automating Let's Encrypt SSL Renewal with Cloudflare DNS](https://gist.github.com/kaczmar2/027fd6f64f4e4e7ebbb0c75cb3409787#file-pihole-v6-docker-le-cf-md)

## Check Docker logs

This will show logs for both the `pihole` and `cloudflared` containers.

```bash
docker logs pihole
docker logs cloudflared
```

## Notes

### Common DNS-over-HTTPS Providers

| Provider | DoH Endpoint |
|----------|-------------|
| [Cloudflare](https://one.one.one.one) | <https://cloudflare-dns.com/dns-query> |
| [Quad9](https://quad9.net/service/service-addresses-and-features) | <https://dns.quad9.net/dns-query> |
| [Google](https://developers.google.com/speed/public-dns/docs/doh) | <https://dns.google/dns-query> |
| [OpenDNS](https://support.opendns.com/hc/en-us/articles/360038086532-Using-DNS-over-HTTPS-DoH-with-OpenDNS) | <https://doh.opendns.com/dns-query> |

**Note**: To use a different DoH provider, update the `DOH_PRIMARY` variable
in your `.env` file with the desired endpoint from the table above.
