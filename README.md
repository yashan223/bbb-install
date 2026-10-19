<img width="1012" alt="bbb-install-banner" src="https://user-images.githubusercontent.com/1273169/141152865-f497c883-6c96-41c6-9040-613b0858878a.png">

# bbb-install

`bbb-install.sh` is a shell script that installs, upgrades, and configures a BigBlueButton server in under 20 minutes. Use it on a fresh Ubuntu server that meets the [minimum requirements](https://docs.bigbluebutton.org/administration/install#minimum-server-requirements) and has a fully qualified domain name (FQDN) pointing to it.

## Quickstart

Log in as root to a fresh Ubuntu server and run:

```
wget -qO- https://raw.githubusercontent.com/yashan223/bbb-install/v4.0.x-release/bbb-install.sh | bash -s -- -v noble-400 -s bbb.example.com -e info@example.com
```

Substitute your own hostname and email. For the full list of flags and more example command lines:

```
wget -qO- https://raw.githubusercontent.com/yashan223/bbb-install/v4.0.x-release/bbb-install.sh | bash -s -- -h
```

The `-v` flag selects the version line, e.g. `noble-400` for BigBlueButton 4.0 on Ubuntu 24.04. Re-running the same command later upgrades to the latest iteration of that line; change `-v` to jump to a newer line.

## Before you run the script

There are three things the script cannot do for you.

### DNS

Configure an FQDN (such as `bbb.example.com`) with an A record resolving to your server's public IPv4 address. Verify before running:

```
dig bbb.example.com @8.8.8.8
```

HTTPS is required — browsers block access to microphone, camera, and screen-share over plain HTTP, so `-s` and `-e` are effectively mandatory for any usable install.

### Server sizing

BigBlueButton is CPU- and bandwidth-intensive. See the [minimum requirements](https://docs.bigbluebutton.org/administration/install#minimum-server-requirements) for guidance.

- **Dedicated public IP:** Digital Ocean, Hetzner.
- **Behind NAT:** Scaleway, Google Compute Engine, Amazon EC2 (`c5.2xlarge`+ recommended), Azure, Oracle Cloud Infrastructure (OCI). `bbb-install.sh` detects internal/external addresses automatically, but you need to configure the external firewall yourself.

### External firewall

If your server is behind an external firewall (AWS security group, Azure NSG, GCP rules, OCI security lists, corporate firewall), open the following inbound ports:

| Port           | Protocol | Purpose |
| -------------- | -------- | ------- |
| 22             | TCP      | SSH |
| 80, 443        | TCP      | HTTP / HTTPS |
| 16384 – 32768  | UDP      | FreeSWITCH / WebRTC media |

Amazon EC2 users should also assign an [Elastic IP](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html) so the server's address is stable across reboots.

#### Oracle Cloud Infrastructure (OCI) Notes

Oracle Cloud Ubuntu images configure strict OS-level `iptables` rules by default in `/etc/iptables/rules.v4` that reject all inbound traffic except port 22 (`-A INPUT -j REJECT --reject-with icmp-host-prohibited`).

Even after opening ports in the OCI VCN Security List / NSG, you must allow incoming traffic in the OS firewall before running the install script, otherwise Let's Encrypt / Certbot validation will fail:

```bash
sudo iptables -I INPUT 5 -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT 6 -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
sudo iptables -I INPUT 7 -p udp -m udp --dport 16384:32768 -j ACCEPT
sudo netfilter-persistent save
```

Even with an external firewall in place, the script installs UFW on the BigBlueButton host itself — defense in depth. The default rules allow SSH only on port 22, so the script stops if sshd listens on another port. Pass `-W` to skip UFW and manage the host firewall yourself.

<details>
<summary>Screenshots: Azure and GCE firewall rules</summary>

Microsoft Azure inbound rules:

![Azure firewall](images/azure-firewall.png?raw=true)

Google Compute Engine — allow 80/443 at instance creation:

![GCE 80-443](images/gce-80-443.png?raw=true)

Then add a firewall rule for UDP 16384-32768:

![GCE firewall](images/gce-firewall.png?raw=true)

</details>

## Common scenarios

Run `bbb-install.sh -h` for every flag and a copy-pasteable command for each of the scenarios below.

### Greenlight

[Greenlight](https://docs.bigbluebutton.org/greenlight/v3/install) is BigBlueButton's official room-manager web UI. Pass `-g` to install it; add `-k` to also install Keycloak for external authentication. After install, Greenlight is served at `https://<your-hostname>/`.

### BigBlueButton LTI framework

The [LTI framework](https://github.com/bigbluebutton/bbb-lti-broker) integrates BigBlueButton with any LTI 1.0 Learning Management System. Install with `-t <KEY>:<SECRET>`, where those values are used by your LMS to authenticate with the broker. Re-running with the same key rotates its secret; a new key adds another consumer. Check the [list of native integrations](https://bigbluebutton.org/schools/integrations/) first — many LMS platforms integrate directly without LTI.

### Private networks

For servers not reachable from the public internet, add `-x` to use Let's Encrypt's manual DNS-01 challenge. You'll be prompted to create a `_acme-challenge.<hostname>` TXT record during install. Certificates expire after 90 days and must be renewed manually — certbot emails you before expiry.

### External TURN server

Most deployments work with BigBlueButton's built-in TURN configuration. Set up a separate TURN server only if your users are behind restrictive firewalls that block UDP.

A TURN host needs its own FQDN (with A and AAAA records), its own email for Let's Encrypt, and a shared secret (any 8–16 character random string). The installation is two steps: (1) install coturn on a separate host with `-c <fqdn>:<secret> -e <email>` and no `-v`; (2) point BigBlueButton at it by re-running the BBB install with `-c <fqdn>:<secret>` appended. One TURN server can be shared across multiple BigBlueButton servers. See `-h` for the exact commands.

### Storing recordings on a separate volume

`/var/bigbluebutton` can grow large. Pass `-m /mnt/recordings` to symlink it to a separately-mounted volume.

## Upgrading

Re-run the same `bbb-install.sh` command you used to install. The script upgrades BigBlueButton and all installed add-ons (Greenlight, Keycloak, LTI) to the latest stable version of the same version line. To jump to a newer line, change `-v`.

Updates are announced on the [bigbluebutton-dev](https://groups.google.com/forum/#!forum/bigbluebutton-dev) mailing list.

## Troubleshooting

### Let's Encrypt SSL Failures ("Error getting validation data")

If the script exits with `Let's Encrypt SSL request for <domain> did not succeed`:

1. **Verify DNS:** Confirm that your domain's A record resolves correctly to your server's public IP:
   ```bash
   dig +short <hostname> A
   ```
2. **Verify Port 80 Reachability:** Check if the HTTP challenge port can be reached from outside:
   ```bash
   curl -I http://<hostname>
   ```
   If curl returns an empty reply or connection error, check both your cloud security list and OS firewall rules (especially on [Oracle Cloud](#oracle-cloud-infrastructure-oci-notes)).
3. **Inspect Certbot Logs:** Check `/var/log/letsencrypt/letsencrypt.log` for detailed failure diagnostics.

If the script fails or misbehaves, open a [GitHub issue](https://github.com/bigbluebutton/bbb-install/issues) with steps to reproduce.

For help with BigBlueButton itself, post to the [BigBlueButton community](https://bigbluebutton.org/community-support/).

## Limitations

- If you are behind an external firewall, `bbb-install.sh` won't configure it. See [External firewall](#external-firewall).
- Cross-major-OS upgrades are not supported. If you're moving from an older Ubuntu release, install on a fresh server and [transfer your recordings](https://docs.bigbluebutton.org/administration/customize/#transfer-recordings).
