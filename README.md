# homelab

Single-node Talos cluster in the garage, managed entirely from this repo with Flux.

| Layer | Tool |
|---|---|
| OS + Kubernetes | [Talos Linux](https://www.talos.dev) 1.14, config rendered by [talhelper](https://budimanjojo.github.io/talhelper) |
| Network | [Cilium](https://cilium.io) (no kube-proxy; L2 announcements give LoadBalancer IPs on the LAN, like MetalLB did) |
| DNS | CoreDNS via Helm: inside the cluster `*.sunnyvale.be` resolves straight to the gateway |
| Ingress | [Envoy Gateway](https://gateway.envoyproxy.io) (Gateway API) + cert-manager wildcard cert (Cloudflare DNS-01) |
| GitOps | [Flux](https://fluxcd.io) through the [Flux Operator](https://fluxoperator.dev), web UI at flux.sunnyvale.be |
| Secrets | [External Secrets](https://external-secrets.io) + [Bitwarden Secrets Manager](https://bitwarden.com/products/secrets-manager/) (free plan) |
| Login (SSO) | [Pocket ID](https://pocket-id.org) (passkeys): own OIDC for Grafana/Flux, gateway-level OIDC for apps without auth |
| Storage | [OpenEBS LocalPV](https://openebs.io): `local-ssd` (default) and `local-hdd`; static volumes for the existing HDD data |
| Databases | [CloudNativePG](https://cloudnative-pg.io) + barman-cloud plugin (WAL archiving to B2) |
| Backups | [VolSync](https://volsync.readthedocs.io) (restic → Backblaze B2), nightly |
| Observability | LGTM: Loki, Grafana, Tempo, Mimir; Alloy collects (Fluent Bit forwards Talos' own logs) |
| Updates | Renovate (presets from [home-operations](https://github.com/home-operations/renovate-presets)), CI renders every PR with [flate](https://github.com/home-operations/flate) |

## Layout

```
talos/                  talconfig.yaml + patches -> machine config (talos/clusterconfig, gitignored)
bootstrap/helmfile.yaml the 5 releases Flux needs to start (Cilium, CoreDNS, CRDs, Flux)
kubernetes/
  flux/cluster/         entrypoint Flux syncs; cluster-settings.yaml holds account IDs
  components/volsync    "give this app a backed-up PVC"
  components/oidc       "put a Pocket ID login in front of this route"
  apps/<namespace>/<app>/ks.yaml + app/   one Flux Kustomization per app
.taskfiles/             task talos:*, bootstrap:*, volsync:*
```

Adding an app: copy an existing folder (for example `kubernetes/apps/media/bazarr`), rename it, and add it to its namespace's `kustomization.yaml`. For a backed-up PVC add the `volsync` component. For a login in front, add the `oidc` component and its callback URL in Pocket ID.

## Tooling

```sh
curl https://mise.run | sh     # once; then in this directory:
mise trust && mise install     # talosctl, talhelper, kubectl, helm, helmfile, flux, task, bws, flate, ...
cp .env.example .env           # fill in the Bitwarden tokens
task                           # list all tasks
```

---

## One-time preparation (before touching the server)

### 1. Accounts

**Bitwarden Secrets Manager** (free plan, separate from your password vault):
1. Bitwarden web vault → product switcher → **Secrets Manager** → create a free organization.
2. Create a project `homelab`, and two machine accounts with *read & write* access to it: `cluster` and `workstation`.
3. Create an access token for each. Put them in `.env` (`BWS_CLUSTER_ACCESS_TOKEN`, `BWS_ACCESS_TOKEN`) together with `BWS_PROJECT_ID`.
4. Put the organization ID and project ID in `kubernetes/flux/cluster/cluster-settings.yaml`. If your account is on bitwarden.eu, change the two URLs there as well.

**Backblaze B2**: create a private bucket and an application key limited to that bucket. Put the bucket name and its S3 endpoint in `cluster-settings.yaml`.

**Cloudflare**: create an API token from the *Edit zone DNS* template, scoped to `sunnyvale.be`. It replaces the global API key you used before.

**GitHub**: push this repo to `github.com/Yan-Solo/home-operations`, or change `sync.url` in `kubernetes/apps/flux-system/flux-instance/app/helm/values.yaml`. Install the [Renovate app](https://github.com/apps/renovate) on it. For barpath, create a fine-grained token with read-only *Contents* access to `barpath-coach/platform`.

### 2. Secrets in Bitwarden Secrets Manager

Create these secrets in the `homelab` project. The **name** must match exactly.

| Name | Value |
|---|---|
| `cloudflare-api-token` | the Cloudflare API token |
| `b2-access-key-id` | B2 application key ID |
| `b2-secret-access-key` | B2 application key |
| `volsync-restic-password` | `openssl rand -base64 32` (**also save it in your password vault**: without it the backups are unreadable) |
| `pocket-id-encryption-key` | `openssl rand -base64 32` |
| `barpath-github-token` | the GitHub token for barpath |
| `gateway-oidc-client-id` / `gateway-oidc-client-secret` | after Pocket ID is up (see below); use `placeholder` for now |
| `grafana-oidc-client-id` / `grafana-oidc-client-secret` | same |
| `flux-oidc-client-id` / `flux-oidc-client-secret` | same |
| `talos-secrets` | created by `task talos:secrets:init` |

Then run `task talos:secrets:init BWS_PROJECT_ID=<id>`. It generates the Talos cluster secrets and stores them in Bitwarden, so a lost laptop never locks you out of the cluster.

---

## Migration (one shot, same machine)

The HDD with your media and photos is **mounted as-is** by Talos and never formatted. App configs on the SSD are rebuilt from scratch, except what you copy out in step A3.

### A. On the old server, before the wipe

1. **Write down the disks**:
   ```sh
   lsblk -o NAME,SIZE,MODEL,SERIAL,FSTYPE,UUID,MOUNTPOINTS
   ```
   You need the serials of the **OS SSD** and the **storage SSD**, and the **filesystem UUID** of the HDD. Fill in every `CHANGEME` in `talos/talconfig.yaml` (OS SSD) and `talos/patches/node/volumes.yaml` (storage SSD, HDD). Both SSDs get wiped; the HDD doesn't.
2. **Immich**: in Immich, *Administration → Jobs → Create job → Database dump*, then copy the newest `backups/immich-db-backup-*.sql.gz` from the pictures folder to your workstation.
   **Strongly recommended:** also copy the whole photo library to another disk. Until the new B2 backups have run, the HDD is the only copy.
3. **Keep the Zigbee network**: copy the Zigbee2MQTT data folder (`configuration.yaml`, `coordinator_backup.json`, `database.db`) to your workstation. That way you don't have to re-pair every device. Do the same with the Home Assistant `/config` folder if you want to keep it:
   ```sh
   kubectl -n home-automation cp <zigbee2mqtt-pod>:/app/data ./backup/zigbee2mqtt
   kubectl -n home-automation cp <home-assistant-pod>:/config ./backup/home-assistant
   ```
4. **Check which IP your Telenet modem forwards 80/443 to**: `kubectl get svc -A | grep LoadBalancer`, the Envoy one. This repo pins Envoy to `192.168.0.60`; either change the forward or `lbipam.cilium.io/ips` in `kubernetes/apps/network/envoy-gateway/config/envoyproxy.yaml`.
5. **Stop everything** (`microk8s stop`), then give the HDD folders fixed names and the new app user:
   ```sh
   cd /var/mnt/hdd
   mv pvc-*_media_media media
   mv pvc-*_media_pictures pictures
   chown -R 568:568 media        # the new containers run as uid 568; leave pictures as is (Immich runs as root)
   ```
6. Shut down. **Remove the GTX 960**, add the 2×8 GB, and enable the iGPU in the BIOS (often called *iGPU Multi-Monitor* or *Primary display: IGD*). Plug the monitor into the motherboard.
7. **Unplug the HDD's SATA cable for the install.** Then the installer physically cannot touch it.

### B. Install Talos (at your desk, once)

```sh
task talos:genconfig                 # renders the config; needs the CHANGEMEs filled
talhelper genurl iso --config-file talos/talconfig.yaml   # ISO with the i915 + intel-ucode extensions
```
Write the ISO to a USB stick and boot the server from it. The console shows its DHCP IP.

```sh
task talos:disks IP=<dhcp-ip>        # double-check the serials match what you filled in
task talos:install IP=<dhcp-ip>      # installs to the OS disk and reboots onto 192.168.0.50
```
Shut down, plug the HDD back in, and move the box to the garage. After it boots:

```sh
talosctl -n 192.168.0.50 get volumestatus   # u-ssd and e-hdd must be "ready"
# Talos only creates the `ssd` volume on free space. If u-ssd stays "waiting" because the
# storage SSD still has its old partitions, wipe it (check the device name with `talosctl get disks`):
#   talosctl -n 192.168.0.50 wipe disk sdX
task talos:bootstrap                 # starts etcd, writes ./kubeconfig
task bootstrap:apps                  # Cilium, CoreDNS, Flux; Flux deploys the rest
flux get ks -A --watch
```

### C. After Flux is green

1. **Pocket ID**: open https://id.sunnyvale.be/setup, create your admin user and register a passkey. Create a group `admins` and add yourself. Then create three OIDC clients:

   | Client | Callback URL(s) | Bitwarden secrets |
   |---|---|---|
   | `flux` | `https://flux.sunnyvale.be/oauth2/callback` | `flux-oidc-client-*` |
   | `grafana` | `https://grafana.sunnyvale.be/login/generic_oauth` | `grafana-oidc-client-*` |
   | `gateway` | `https://{sonarr,radarr,prowlarr,bazarr,sabnzbd,qbit,shelfmark,zigbee2mqtt,hubble}.sunnyvale.be/oauth2/callback` (one line each) | `gateway-oidc-client-*` |

   Restrict each client to the `admins` group, paste the IDs and secrets into Bitwarden, and run `task reconcile`. External Secrets picks up the new values within an hour; `kubectl annotate es -A --all force-sync=$(date +%s) --overwrite` makes it immediate.

2. **Immich database**:
   ```sh
   flux -n flux-system suspend ks immich
   kubectl -n media scale deploy immich-server --replicas 0
   gunzip --stdout immich-db-backup-*.sql.gz \
     | sed "s/SELECT pg_catalog.set_config('search_path', '', false);/SELECT pg_catalog.set_config('search_path', 'public, pg_catalog', true);/g" \
     | kubectl -n media exec -i immich-postgresql-1 -c postgres -- psql -U postgres
   flux -n flux-system resume ks immich && flux -n media reconcile hr immich --force
   ```
   Immich stays on the version that wrote the dump (v3.0.0). Let Renovate move it forward afterwards.

3. **Zigbee2MQTT / Home Assistant data** (if you kept it):
   ```sh
   kubectl -n home-automation cp ./backup/zigbee2mqtt/. <zigbee2mqtt-pod>:/data
   kubectl -n home-automation cp ./backup/home-assistant/. <home-assistant-pod>:/config
   kubectl -n home-automation rollout restart deploy zigbee2mqtt home-assistant
   ```
   In the restored Zigbee2MQTT `configuration.yaml`, point `mqtt.server` at `mqtt://mosquitto.home-automation.svc.cluster.local` (the env vars in the HelmRelease already override it).

4. **The *arr apps** start empty: point them at `/media`, `/downloads` (SABnzbd) and `/qbittorrent-downloads`. Their auth is set to *External*, because Pocket ID guards them at the gateway.

5. **Trigger a first backup** of everything: `task volsync:list`, then `task volsync:snapshot APP=<app> NS=<ns>`. The photo library (`immich-library`) runs nightly at 04:00; the first run uploads everything and takes a while.

---

## Day 2

- **Updates**: Renovate opens PRs, CI renders them, and minor/patch versions automerge. Talos and Kubernetes are never automerged: merge the PR, then `task talos:upgrade-node` or `task talos:upgrade-k8s`.
- **Config changes to the node**: edit `talos/`, then `task talos:apply`. It runs in `try` mode, so a change that cuts off the network rolls itself back after a minute instead of sending you to the garage.
- **See what the node is doing**: `task talos:dashboard`, or Grafana → *Node Exporter Full*.
- **Restore an app**: `task volsync:restore APP=home-assistant NS=home-automation`. Add `PREVIOUS=1` (skip the newest snapshot) or `AS_OF=<RFC 3339 time>` to go further back; `task --summary volsync:restore` explains both.
- **Render locally like CI**: `task validate`.

### Rebuild from scratch (new disk, reinstall, ...)

The HDD keeps media and photos. Everything on the SSDs comes back from git and B2:

1. Phase B of the migration above (`task talos:install`, `task talos:bootstrap`, `task bootstrap:apps`). Every app starts with an empty volume.
2. **App data**: as soon as Flux is green, note the time and restore everything from before the rebuild:
   ```sh
   task volsync:restore-all AS_OF=2026-11-01T12:00:00Z   # a moment before you wiped the old install
   ```
   Do this the same day: the 03:00 backup saves the empty volumes as the newest snapshot. Nothing is lost if you're late (7 daily, 4 weekly and 3 monthly snapshots are kept); `AS_OF` simply skips past the empty ones.
3. **Databases** (Immich, barpath): a new cluster can't archive into a folder that already holds backups, so it recovers from the old folder and writes to a new one. In the app's `database.yaml`, replace `bootstrap.initdb` and bump `serverName`:
   ```yaml
   spec:
     bootstrap:
       recovery:
         source: previous
     externalClusters:
       - name: previous
         plugin:
           name: barman-cloud.cloudnative-pg.io
           parameters:
             barmanObjectName: immich-postgresql
             serverName: immich-postgresql-v1   # the folder the old cluster wrote to
     plugins:
       - name: barman-cloud.cloudnative-pg.io
         isWALArchiver: true
         parameters:
           barmanObjectName: immich-postgresql
           serverName: immich-postgresql-v2     # new, empty folder
   ```
   If the cluster already started empty, delete it (`kubectl -n media delete cluster immich-postgresql`) and let Flux recreate it. Once it's running you can leave `recovery` in place; it only runs when the cluster is created. See the [barman-cloud plugin docs](https://cloudnative-pg.io/plugin-barman-cloud/docs/usage/#restoring-a-cluster).
