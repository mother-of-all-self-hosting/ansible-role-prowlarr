<!--
SPDX-FileCopyrightText: 2020 Aaron Raimist
SPDX-FileCopyrightText: 2020 Chris van Dijk
SPDX-FileCopyrightText: 2020 Dominik Zajac
SPDX-FileCopyrightText: 2020 Mickaël Cornière
SPDX-FileCopyrightText: 2020-2024 MDAD project contributors
SPDX-FileCopyrightText: 2020-2024 Slavi Pantaleev
SPDX-FileCopyrightText: 2022 François Darveau
SPDX-FileCopyrightText: 2022 Julian Foad
SPDX-FileCopyrightText: 2022 Warren Bailey
SPDX-FileCopyrightText: 2023 Antonis Christofides
SPDX-FileCopyrightText: 2023 Felix Stupp
SPDX-FileCopyrightText: 2023 Pierre 'McFly' Marty
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up Prowlarr

This is an [Ansible](https://www.ansible.com/) role which installs [Prowlarr](https://prowlarr.com/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

Prowlarr is an indexer manager/proxy for Usenet and BitTorrent users. It manages your indexers in one place and syncs them to applications such as Sonarr and Radarr.

See the project's [documentation](https://wiki.servarr.com/prowlarr) to learn what Prowlarr does and why it might be useful to you.

## Adjusting the playbook configuration

To enable Prowlarr with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# prowlarr                                                             #
#                                                                      #
########################################################################

prowlarr_enabled: true

########################################################################
#                                                                      #
# /prowlarr                                                            #
#                                                                      #
########################################################################
```

### Set the hostname

To enable Prowlarr you need to set the hostname as well. To do so, add the following configuration to your `vars.yml` file. Make sure to replace `example.com` with your own value.

```yaml
prowlarr_hostname: "example.com"
```

After adjusting the hostname, make sure to adjust your DNS records to point the domain to your server.

>[!NOTE]
> The `prowlarr_path_prefix` variable can be adjusted to host under a subpath (e.g. `prowlarr_path_prefix: /prowlarr`), but this hasn't been tested yet.

### Configuring HTTP Basic authentication (optional)

To put HTTP Basic authentication on Traefik in front of Prowlarr, add the following configuration to your `vars.yml` file. Refer to [this page](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/basicauth/) on the Traefik's documentation for details.

```yaml
prowlarr_container_labels_traefik_middleware_basic_auth_enabled: true

prowlarr_container_labels_traefik_middleware_basic_auth_users: 'USERNAME_HERE:HASHED_PASSWORD_HERE'
```

Make sure to replace `USERNAME_HERE` and `HASHED_PASSWORD_HERE` with your own values. You can create users by running `htpasswd -nb USERNAME_HERE PASSWORD_HERE`.

### Mounting additional data directories (optional)

To mount additional data directories, add the following configuration to your `vars.yml` file (adapt to your needs):

```yaml
prowlarr_container_additional_volumes_custom:
  - type: bind
    src: /path/to/blackhole
    dst: /downloads
```

### Configuring trusted networks

Prowlarr only trusts forwarded headers from loopback addresses by default. For Traefik to pass the original client address and HTTPS scheme to Prowlarr, it is necessary to configure **Trusted Networks** with the proxy's address or network. Refer to [Prowlarr's security settings](https://wiki.servarr.com/prowlarr/settings#security) for details.

First, inspect the Docker network shared by Traefik and Prowlarr on the server:

```sh
docker network inspect NETWORK_NAME --format '{{ range .IPAM.Config }}{{ println .Subnet }}{{ end }}'
```

Replace `NETWORK_NAME` with that network's actual name. Keep in mind that only the proxy's address or the specific subnet it connects from should be trusted. Trusting a subnet also trusts other containers attached to it. For an external proxy, use its source address or subnet as seen by Prowlarr.

You can configure **Settings → General → Security → Trusted Networks** in Prowlarr. To apply the setting with an environment variable, add the following configuration to your `vars.yml` file (adapt to your needs):

```yaml
# This is an example. Replace the value with the actual proxy subnet.
prowlarr_environment_variables_additional_variables: |
  PROWLARR__SERVER__TRUSTEDNETWORKS=172.20.0.0/24
```

You can specify multiple addresses or subnets by comma-separating them. This environment setting takes precedence over the value saved in Prowlarr's configuration.

It is recommended to keep authentication required for all addresses, especially when using a reverse proxy. If you configure **Allowed Hosts**, make sure to include `prowlarr_hostname` and any additional names used by API clients; an empty list currently accepts all hostnames.

After upgrading, make sure to verify login and API access through the public HTTPS URL, and check Prowlarr's logs for the configured trusted network and any rejected hosts.

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using the `prowlarr_environment_variables_additional_variables` variable

Refer to [this page](https://wiki.servarr.com/prowlarr/environment-variables) for available options which can be set to `prowlarr_environment_variables_additional_variables`.

### Notes on configuration

A freshly installed Prowlarr has no authentication of its own, and this role does not add any. Prowlarr also serves its API key to unauthenticated callers on `/initialize.json`, and that key is enough to drive the whole API. It is recommended to turn authentication on under *Settings -> General -> Security* in Prowlarr itself, or to [enable HTTP Basic authentication](#configuring-http-basic-authentication-optional) on Traefik, before making an installation reachable from the internet.

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, Prowlarr becomes available at the specified hostname like `https://example.com`.

To get started, open the URL with a web browser to create an account. The recommended authentication method is `Forms (Login Page)`.

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu prowlarr` (or how you/your playbook named the service, e.g. `mash-prowlarr`).
