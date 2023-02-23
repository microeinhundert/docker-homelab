# Docker Home Lab 

Setup for running a number of services on my private home lab, including:

- Reverse proxy with Traefik
- Ad-blocking with AdGuard Home DNS
- Home automation with Home Assistant and Homebridge
- Object storage with MinIO
- Low-code programming with Node-RED
- Container management with Portainer
- Password management with Bitwarden
- Monitoring with Grafana, Prometheus and various Prometheus scrapers (AdGuard, FRITZ!, Netatmo etc.)

## Router Configuration

Your router should be configured to forward the ports 80, 443 and 853 to your server's IP address.
Your router should also be configured to use the DNS server provided by AdGuard for it's DNS resolution. This will ensure that all devices on your network use AdGuard as their DNS without any additional configuration.

## Domains

For Let's Encrypt certificates to be issued, all domains used must be registered in a public DNS and point to your server via DynDNS. Don't worry, access to services running on your server from outside your home network will be prohibited by Traefik. If certain services should be accessible to the public, the Traefik middleware `only-internal-ips` should be removed from these services.

In order for your devices to access private services behind the `only-internal-ips` middleware, these devices should use the AdGuard DNS.

### Subdomains

The following subdomains are used:

- adguard.*
- minio.*
- console.minio.*
- portainer.*
- edge.portainer.*
- homebridge.*
- homeassistant.*
- nodered.*
- prometheus.*
- traefik.*
- bitwarden.*

All subdomains should have their CNAME record point to the main domain.

## Docker

This setup requires three external Docker networks named `traefik`, `monitoring` and `docker_socket_proxy`. Create them by running the following commands:

```sh
docker network create traefik
docker network create monitoring
docker network create docker_socket_proxy
```

The Docker daemon should also be configured to expose metrics, learn more at https://docs.docker.com/config/daemon/prometheus/.

### Socket Security

Access to the Docker socket at `/var/run/docker.sock` is secured by a proxy running in a separate container. Because the proxy blocks all write access to the API, Portainer is a read-only application.

## Credentials

Replace all occurrences of `### REDACTED ###` with your credentials.

In `networking/services/traefik/conf/users`, replace `### REDACTED ###` with a password hash, either using MD5, SHA1, or BCrypt.
Learn more at https://doc.traefik.io/traefik/middlewares/http/basicauth/.

## SMTP Server

The services `bitwarden` and `grafana` require SMTP server credentials set via environment variables.

**Bitwarden:** Environment variables prefixed with `globalSettings__mail__smtp__`.  
**Grafana:** Environment variables prefixed with `GF_SMTP_HOST`.

By default, the SMTP server host for Strato is set.

## MinIO Prometheus Monitoring

First, create an Access Key in the MinIO Console and write it down somewhere.

Install the MinIO Admin Client on the host system:

```sh
# MacOS
brew install minio/stable/mc
```

Set up the Admin Client to point to the server running MinIO:

```sh
bash +o history
mc config host add homelab <ENDPOINT> <ACCESS_KEY> <SECRET_KEY>
bash -o history
```

Generate a JWT bearer token for use with Prometheus:

```sh
mc admin prometheus generate homelab
```

Add the generated token to `bearer_token` inside of the job named `minio` inside of `monitoring/services/prometheus/conf/prometheus.yml`.

```yml
scrape_configs:
  - job_name: minio
    scrape_interval: 60s
    bearer_token: TOKEN # Add the token here
    metrics_path: /minio/v2/metrics/cluster
    scheme: http
    static_configs:
      - targets: ["minio:9000"]
```

> Learn more: https://min.io/docs/minio/linux/reference/minio-mc-admin/mc-admin-prometheus.html

## Home Assistant behind Traefik proxy

Open `iot/services/homeassistant/conf/configuration.yaml` and add the ingress IP from the Traefik reverse proxy to the list of trusted proxies:

```yml
http:
  ip_ban_enabled: true
  login_attempts_threshold: 5
  use_x_forwarded_for: true
  trusted_proxies:
    - 0.0.0.0 # Add the Traefik ingress IP here
```
