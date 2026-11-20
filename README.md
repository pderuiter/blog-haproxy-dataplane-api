# blog-haproxy-dataplane-api

Companion code for the blog post
[HAProxy Data Plane API: Dynamic Configuration Management](https://blog.bsdserver.nl/haproxy-data-plane-api-dynamic-configuration-management/).

It builds the post's setup on one host: HAProxy from the official HAProxy
packages, the HAProxy Data Plane API 3.x on `127.0.0.1:5555` with the post's
`dataplaneapi.yaml`, and an HAProxy TLS frontend on port 5556 that publishes
the API with basic authentication (`api_users` userlist), next to the
application frontend and the `web_servers` backend from the post.

## What is in here

| Path | Purpose |
|------|---------|
| `terraform/` | One Ubuntu 24.04 VM on Proxmox VE (bpg/proxmox provider, cloud-init), outputs an Ansible inventory |
| `ansible/site.yml` | One playbook for the whole setup |
| `ansible/roles/haproxy_install` | HAProxy from the vbernat PPA (Ubuntu) or haproxy.debian.net (Debian) |
| `ansible/roles/dataplaneapi` | Data Plane API binary, `dataplaneapi.yaml`, `haproxy.cfg`, API certificate and systemd unit |

## Requirements

- Terraform >= 1.6 and a Proxmox VE API token (only if you want Terraform to create the VM)
- ansible-core >= 2.15 (tested with the current ansible-core release)
- Target host: Ubuntu 24.04, amd64, with Internet access

## Quick start

```bash
# 1. Machine (optional; skip if you already have a host)
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in your values
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

# 2. HAProxy and the Data Plane API
cd ../ansible
cp inventory.example.yml inventory.yml         # if you skipped Terraform
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
mkpasswd -m sha-512 'your-secure-password'     # put the hash in vault.yml
ansible-vault encrypt group_vars/all/vault.yml
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml --ask-vault-pass

# 3. Test (the "API Operations" section of the post)
export HAPROXY_API="https://10.0.0.10:5556/v3" AUTH="admin:your-secure-password"
curl -sk -u $AUTH $HAPROXY_API/info | jq .
curl -sk -u $AUTH "$HAPROXY_API/services/haproxy/configuration/backends/web_servers/servers" | jq .
```

`-k` is only needed with the generated self-signed certificate.

## Variables

Set in `ansible/group_vars/all/vars.yml` (defaults in each role's `defaults/main.yml`):

| Variable | Default | Meaning |
|----------|---------|---------|
| `haproxy_install_version` | `3.3` | HAProxy branch. 3.3 is the newest the PPA builds for Ubuntu 24.04 (maintained until about Q1 2027); `3.2` is LTS |
| `dataplaneapi_version` | `3.4.7` | Data Plane API release. Changing it upgrades (or downgrades) an existing install |
| `dataplaneapi_user` | `admin` | API user, in `dataplaneapi.yaml` and in the `api_users` userlist |
| `vault_dataplaneapi_password_hash` | (vault) | SHA-512 crypt hash from `mkpasswd -m sha-512` |
| `dataplaneapi_host` | `127.0.0.1` | Where the API listens. Plain HTTP, so localhost only; clients use the TLS frontend |
| `dataplaneapi_port` / `dataplaneapi_tls_port` | `5555` / `5556` | Data Plane API, and the HAProxy TLS frontend in front of it |
| `dataplaneapi_cert_pem` | `""` | Certificate and key (one PEM) for the TLS frontend; empty generates a self-signed one for `dataplaneapi_cert_cn` |
| `dataplaneapi_web_servers` | `web1`, `web2` | Servers in the `web_servers` backend (`name`, `address` as `ip:port`) |

## Differences from the post

- `dataplaneapi.yaml` is written in the normalized form the Data Plane API
  rewrites on every start (key order, the default `advertised`, `userlist` and
  `reload_strategy` entries, mode 0600). The settings are the post's; without
  this every restart would make the next run report a change.
- The post assumes `api.pem` exists; the role generates a self-signed one when
  you do not provide a certificate.

## Ansible or the API

`haproxy.cfg` is bootstrap-only. The first run replaces the package's default
file with the template (including `dataplaneapi_web_servers`) and writes the
marker `/etc/haproxy/.haproxy-cfg-bootstrapped`. After that the template task
runs with `force: false` and the Data Plane API owns `haproxy.cfg`: later runs
leave changes made through the API alone. Delete the marker to write the
template again (this overwrites the API's changes).

## Tested

`site.yml` was run against a fresh Ubuntu 24.04 systemd container (HAProxy
3.3.17, Data Plane API 3.4.7): the second run reports `changed=0`, and so does
a run after restarting both services. After that, `/v3/info` answered through
the TLS frontend (401 without credentials), a v3 transaction added a server
that showed up in the runtime, a runtime drain took effect, and the application
frontend served HTTP and HTTPS. Port 5555 is not reachable from other hosts.
The post's `canary-deploy.sh` gave the canary 99 of 1000 requests after
`shift 10` and 249 after `shift 25`, also after a reload, and a playbook run
after those API changes kept them (`changed=0`). ansible-lint (production profile), yamllint,
`terraform validate` and tflint are clean. The Terraform code has not been
applied against a real Proxmox cluster.

## License

MIT
