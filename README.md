# home-operations

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
| Storage | [OpenEBS LocalPV](https://openebs.io): `local-ssd` (default) and `local-hdd`; static volumes with fixed paths for media and photos on the HDD |
| Databases | [CloudNativePG](https://cloudnative-pg.io) + barman-cloud plugin (WAL archiving to S3 storage) |
| Backups | [VolSync](https://volsync.readthedocs.io) (restic → Backblaze B2, EU Central), nightly |
| Observability | LGTM: Loki, Grafana, Tempo, Mimir; Alloy collects (Fluent Bit forwards Talos' own logs) |
| Updates | Renovate (presets from [home-operations](https://github.com/home-operations/renovate-presets)), CI renders every PR with [flate](https://github.com/home-operations/flate) |

## Layout

```
talos/                          talconfig.yaml + patches -> machine config (talos/clusterconfig, gitignored)
bootstrap/helmfile.yaml.gotmpl  the 5 releases Flux needs to start (Cilium, CoreDNS, CRDs, Flux)
kubernetes/
  flux/cluster/                 entrypoint Flux syncs; cluster-settings.yaml holds account IDs
  components/volsync            "give this app a backed-up PVC"
  components/oidc               "put a Pocket ID login in front of this route"
  apps/<namespace>/<app>/       ks.yaml + app/: one Flux Kustomization per app
.taskfiles/                     task talos:*, bootstrap:*, volsync:*
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
2. Create a project (any name, e.g. `home-operations`; only its ID is used) and two machine accounts with access to it: `workstation` with *read & write* (it stores the Talos secrets), and `cluster` with *read* only (External Secrets only reads).
3. Create an access token for each. Put them in `.env` (`BWS_CLUSTER_ACCESS_TOKEN`, `BWS_ACCESS_TOKEN`) together with `BWS_PROJECT_ID`.
4. Put the organization ID and project ID in `kubernetes/flux/cluster/cluster-settings.yaml`. If your account is on bitwarden.eu, change the two URLs there as well.

**Backup bucket** (Backblaze B2): create the account with **EU Central** selected at sign-up (the region reportedly can't be changed later), then a private bucket and an application key limited to that bucket. Put the bucket name in `cluster-settings.yaml`, and check `S3_ENDPOINT` and `S3_REGION` there against the endpoint on the bucket's page. Set the bucket's lifecycle rule to *Keep prior versions for this number of days*, with **30** days: B2 keeps deleted files as hidden versions, so this makes a delete undoable for 30 days (also one by a compromised cluster) while the snapshots restic prunes and the WAL files CNPG expires still stop being billed afterwards. It only protects you if the cluster's key can't permanently delete a version, so try that with the key before relying on it.

**Cloudflare**: create an API token from the *Edit zone DNS* template, add the permission *Zone → Zone → Read*, and scope it to `sunnyvale.be`. It replaces the global API key you used before.

**GitHub**: push this repo to `github.com/Yan-Solo/home-operations`, or change `sync.url` in `kubernetes/apps/flux-system/flux-instance/app/helm/values.yaml`. **Make the repo public before `task bootstrap:apps`**: Flux pulls it over HTTPS without credentials (nothing secret is committed; secrets live in Bitwarden). Install the [Renovate app](https://github.com/apps/renovate) on it. For barpath, create a fine-grained token with read-only *Contents* access to `barpath-coach/platform`.

### 2. Secrets in Bitwarden Secrets Manager

Create these secrets in that project. The **name** must match exactly.

| Name | Value |
|---|---|
| `cloudflare-api-token` | the Cloudflare API token |
| `s3-access-key-id` | access key ID for the backup bucket |
| `s3-secret-access-key` | secret access key for the backup bucket |
| `volsync-restic-password` | `openssl rand -base64 32` (**also save it in your password vault**: without it the backups are unreadable) |
| `pocket-id-encryption-key` | `openssl rand -base64 32` |
| `barpath-github-token` | the GitHub token for barpath |
| `frigate-tapo-username` / `frigate-tapo-password` | the Tapo C200's *Camera Account* (Tapo app → camera → Settings → Advanced settings → Camera Account) |
| `gateway-oidc-client-id` / `gateway-oidc-client-secret` | after Pocket ID is up (see below); use `placeholder` for now |
| `grafana-oidc-client-id` / `grafana-oidc-client-secret` | same |
| `flux-oidc-client-id` / `flux-oidc-client-secret` | same |
| `gameplane-oidc-client-id` / `gameplane-oidc-client-secret` | same |
| `immich-oidc-client-id` / `immich-oidc-client-secret` | same |
| `talos-secrets` | created by `task talos:secrets:init` |

Then run `task talos:secrets:init BWS_PROJECT_ID=<id>`. It generates the Talos cluster secrets and stores them in Bitwarden, so a lost laptop never locks you out of the cluster.

`task bootstrap:check` tells you what's still missing: unfilled settings, a repo Flux can't read, unpushed commits, or a secret that isn't in Bitwarden yet. `task bootstrap:apps` runs it first.

---

## Migration (one shot, onto the EliteDesk)

The disks move from the old server into the EliteDesk 805 G6, so the old server stops for good in step A5. All three disks are wiped and Talos formats the data disks as XFS. Only what you copy out in steps A2 and A3 survives: the Immich photos and, optionally, the Zigbee2MQTT and Home Assistant data. Media is downloaded again.

### A. On the old server, before it goes off

1. **Write down the disks**:
   ```sh
   lsblk -o NAME,SIZE,MODEL,SERIAL,FSTYPE,UUID,MOUNTPOINTS
   ```
   You need the serials of the **OS SSD**, the **storage SSD** and the **HDD**. Fill in every `CHANGEME` in `talos/talconfig.yaml` (OS SSD) and `talos/patches/node/volumes.yaml` (storage SSD, HDD).
2. **Immich photos**: copy the originals to your workstation. They're in `library/` and `upload/` inside the pictures folder; skip `thumbs/` and `encoded-video/`, which Immich regenerates. Check that the copy is complete (`du -sh`, open a few files) before going on: after the wipe it's the only copy.
   ```sh
   rsync -av --exclude thumbs --exclude encoded-video --exclude backups <old-server>:<pictures-folder>/ ./backup/immich/
   ```
3. **Keep the Zigbee network**: copy the Zigbee2MQTT data folder (`configuration.yaml`, `coordinator_backup.json`, `database.db`) to your workstation. That way you don't have to re-pair every device. Do the same with the Home Assistant `/config` folder if you want to keep it:
   ```sh
   kubectl -n home-automation cp <zigbee2mqtt-pod>:/app/data ./backup/zigbee2mqtt
   kubectl -n home-automation cp <home-assistant-pod>:/config ./backup/home-assistant
   ```
4. **Check which IP your Telenet modem forwards 80/443 to**: `kubectl get svc -A | grep LoadBalancer`, the Envoy one. This repo pins Envoy to `192.168.0.60`; either change the forward or `lbipam.cilium.io/ips` in `kubernetes/apps/network/envoy-gateway/config/envoyproxy.yaml`.
5. Shut the old server down. Move the disks and the **Zigbee dongle** into the EliteDesk. Memory goes in balanced so it all runs dual channel: one 16 GB and one 8 GB in DIMM3/4 (channel A), the same in DIMM1/2 (channel B). The slowest stick sets the speed for all of them.
6. In the EliteDesk BIOS (F10):
   - **Update System BIOS** first, from USB or hp.com: this is awkward to do once Talos runs and the box is in the garage.
   - *Advanced → Boot Options*: **After Power Loss: Power On** (the default is Power Off, so a power cut would otherwise mean a walk to the garage), USB Storage Boot on, Network (PXE) Boot off, Prompt on Memory Size Change off.
   - *Secure Boot Configuration*: **Legacy Support Disable and Secure Boot Disable** (the standard Talos ISO isn't signed for Secure Boot).
7. **Leave the data disks unconnected for the install.** Then the installer can only see the OS disk.

### B. Install Talos (at your desk, once)

```sh
task talos:genconfig                 # renders the config; needs the CHANGEMEs filled
talhelper genurl image --config-file talos/talconfig.yaml --env-file talos/talenv.yaml  # ISO URL with the amdgpu, amd-ucode and realtek-firmware extensions
```
Write the ISO to a USB stick and boot the server from it. The console shows its DHCP IP.

```sh
task talos:disks IP=<dhcp-ip>        # double-check the serials, and that there's one physical NIC (the static IP uses `physical: true`)
task talos:install IP=<dhcp-ip>      # installs to the OS disk and reboots onto 192.168.0.50
```
Shut down, plug the data disks back in, and move the box to the garage. On this boot Talos formats them:

```sh
talosctl -n 192.168.0.50 get volumestatus   # u-ssd and u-hdd must be "ready"
# Talos only creates a volume on free space. If u-ssd or u-hdd stays "waiting" because the
# disk still has its old partitions, wipe it (check the device name with `talosctl get disks`):
#   talosctl -n 192.168.0.50 wipe disk sdX
task talos:bootstrap                 # starts etcd, writes ./kubeconfig
task bootstrap:apps                  # Cilium, CoreDNS, Flux; Flux deploys the rest
flux get ks -A --watch
```

### C. After Flux is green

1. **Pocket ID**: open https://id.sunnyvale.be/setup, create your admin user and register a passkey. Do this first, as soon as the `pocket-id` Kustomization is ready: until an admin exists, anyone who reaches `/setup` can become it, and your modem already forwards 443 to the gateway. If you'll be away from the keyboard, turn the port forward off until this is done. Create a group `admins` and add yourself. Then create five OIDC clients:

   | Client | Callback URL(s) | Bitwarden secrets |
   |---|---|---|
   | `flux` | `https://flux.sunnyvale.be/oauth2/callback` | `flux-oidc-client-*` |
   | `grafana` | `https://grafana.sunnyvale.be/login/generic_oauth` | `grafana-oidc-client-*` |
   | `gameplane` | `https://gameplane.sunnyvale.be/auth/oidc/helm/callback` | `gameplane-oidc-client-*` |
   | `immich` | `https://immich.sunnyvale.be/auth/login`, `https://immich.sunnyvale.be/user-settings`, `app.immich:///oauth-callback` | `immich-oidc-client-*` |
   | `gateway` | `https://{sonarr,radarr,prowlarr,bazarr,sabnzbd,qbit,shelfmark,zigbee2mqtt,frigate,hubble,home-assistant}.sunnyvale.be/oauth2/callback` (one line each) | `gateway-oidc-client-*` |

   Restrict each client to the `admins` group, paste the IDs and secrets into Bitwarden, and run `task reconcile`. External Secrets picks up the new values within an hour; `kubectl annotate es -A --all force-sync=$(date +%s) --overwrite` makes it immediate.

2. **Immich photos**: open https://immich.sunnyvale.be, create your admin user, and create an API key under *Account settings → API keys*. Then upload the copy from step A2:
   ```sh
   npx @immich/cli login https://immich.sunnyvale.be/api <api-key>
   npx @immich/cli upload --recursive ./backup/immich
   ```
   Immich rebuilds thumbnails, faces and places from the photos themselves. Albums and favourites start empty.

3. **Zigbee2MQTT / Home Assistant data** (if you kept it):
   ```sh
   kubectl -n home-automation cp ./backup/zigbee2mqtt/. <zigbee2mqtt-pod>:/data
   kubectl -n home-automation cp ./backup/home-assistant/. <home-assistant-pod>:/config
   kubectl -n home-automation rollout restart deploy zigbee2mqtt home-assistant
   ```
   In the restored Zigbee2MQTT `configuration.yaml`, point `mqtt.server` at `mqtt://mosquitto.home-automation.svc.cluster.local` (the env vars in the HelmRelease already override it).

4. **The *arr apps** start empty: point them at `/media`, `/sab-downloads` and `/qbittorrent-downloads`. Their auth is set to *External*, because Pocket ID guards them at the gateway.

5. **Trigger a first backup** of everything: `task volsync:list`, then `task volsync:snapshot APP=<app> NS=<ns>`. The photo library (`immich-library`) runs nightly at 04:00; the first run uploads everything and takes a while.

---

## Day 2

- **Updates**: Renovate opens PRs, CI renders them, and minor/patch versions automerge. Talos and Kubernetes are never automerged: merge the PR, then `task talos:upgrade-node` or `task talos:upgrade-k8s`.
- **Config changes to the node**: edit `talos/`, check with `task talos:diff`, then `task talos:apply`. For a risky network change, first run `task talos:apply MODE=try`: it applies the change for one minute and then always rolls back, so a mistake doesn't send you to the garage. If the node stayed reachable, run `task talos:apply` to keep it.
- **See what the node is doing**: `task talos:dashboard`, or Grafana → *Node Exporter Full*.
- **Restore an app**: `task volsync:restore APP=home-assistant NS=home-automation`. Add `PREVIOUS=1` (skip the newest snapshot) or `AS_OF=<RFC 3339 time>` to go further back; `task --summary volsync:restore` explains both.
- **Render locally like CI**: `task validate`.

### Rebuild from scratch (new disk, reinstall, ...)

The HDD keeps media and photos. Everything on the SSDs comes back from git and the backup bucket. If the HDD itself was lost, the photos come back from their own backup: `task volsync:restore APP=immich-library NS=media PVC=pictures KS=immich` (media is downloaded again).

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
