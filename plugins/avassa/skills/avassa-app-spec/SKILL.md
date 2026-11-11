# Avassa application specifications + Strongbox

Produce correct, deployable Avassa YAML: an **application**, its **application-deployment**, and the **Strongbox vaults/secrets** (and optionally a TLS CA) it depends on. Sources: https://docs.avassa.io and https://avassa-api.redoc.ly (tags `Applications`, `Application-Deployments`, `Vault`).

## Workflow

1. **Gather requirements** (ask only for what's missing and can't be defaulted):
   - Images (with tags), how containers group into services (containers that must run together / share localhost = same service).
   - Ports to expose on the site network (→ ingress), outbound internet needs.
   - Config files, env vars, persistent data.
   - Secrets: which values, consumed as env var or file, per-site or global, TLS certs needed?
   - Target sites (labels) and rollout style (all at once, rolling, canary).
2. **Verify the schema if possible.** If the `supd` MCP tools are available (`get_model`, `search_model`, `get_docs`, `read_doc`), use `get_model ["config","applications",...]` / `["config","strongbox","vaults",...]` to confirm field names before writing anything unusual. Otherwise rely on the reference below and the redoc API docs.
3. **Write files** using this layout (one resource per file, names as in Avassa's public examples):
   ```
   <app>/
     <app>.ca.yaml               # optional TLS CA (strongbox tls ca)
     <app>.vault.yaml            # vault
     credentials.<app>.secret.yaml   # secret(s) in that vault
     <app>.app.yaml              # application
     <app>.dep.yaml              # application-deployment
     apply.sh                    # supctl commands in dependency order
   ```
4. **Run the checklist** at the end before handing over.

Never put real secret values into files that may be committed. Use obvious placeholders (`CHANGE_ME`) or show the secret being created from env vars in `apply.sh`, and say so.

## Apply order (dependencies)

CA → vault → secrets → application → application-deployment.

```sh
supctl create strongbox tls ca < myapp.ca.yaml          # only if auto-cert is used
supctl create strongbox vaults < myapp.vault.yaml
supctl create strongbox vaults myapp secrets < credentials.myapp.secret.yaml
supctl create applications < myapp.app.yaml
supctl create application-deployments < myapp.dep.yaml
```
Use `supctl replace ...` (PUT) or `supctl merge ...` (PATCH) for updates, e.g. `supctl replace applications myapp < myapp.app.yaml`. Check with `supctl show applications myapp` (look for `image-status: present`) and `supctl show application-deployments myapp-dep`.

REST equivalents: `POST /v1/config/strongbox/vaults`, `POST /v1/config/strongbox/vaults/{vault}/secrets`, `POST /v1/config/applications`, `POST /v1/config/application_deployments` (YAML or JSON body; `?dry-run=true` / `?validate=true` are useful).

## Strongbox vault

```yaml
name: myapp                 # vault name
distribute:                 # REQUIRED in practice: secrets go nowhere unless distributed
  deployments:              # same sites as these application-DEPLOYMENT names (not app names)
    - myapp-dep
  # alternatives (pick one):
  # to: all                 # all sites the tenant has
  # to: none                # Control Tower only (e.g. registry creds)
  # sites: [stockholm-sergel, helsingborg-bergakungen]
max-versions: 1             # >1 = versioned vault (rollback via delete/undelete)
cas-required: false         # true = writes must pass -q version=<current> (0 to create)
# delete-version-after: 0s
```
Prefer `distribute.deployments` so secrets only land where the app runs.

## Vault secret (key/value map)

```yaml
name: credentials
allow-application-access:          # "<application>.<service>.<container>"; avoid "*" in production
  - myapp.api.server
data:                              # plain-string values
  username: CHANGE_ME
  password: CHANGE_ME
# base64-data:                     # binary / multi-line values; wins over `data` on same key
#   keystore.p12: <base64>
# allow-tenant-access: [...]       # share with another tenant (see how-to/tenant-secret-sharing)
```
Other secret fields: `ttl`, `allow-config-access`, `allow-image-access` (legacy), `auto-cert`, `auto-acme-cert`, `auto-ssh-cert`.

Creating with CAS: `supctl -q version=0 create strongbox vaults <vault> secrets < file`. Updating one key: `supctl merge strongbox vaults <vault> secrets credentials <<EOF\ndata:\n  password: new\nEOF`.

### Auto-generated TLS certificate

```yaml
# myapp.ca.yaml
name: myapp-ca
ttl: 2y
auto-renew:
  renew-threshold: 145d
  activate-threshold: 90d
distribute:
  deployments: [myapp-dep]
---
# cert.myapp.secret.yaml
name: server-cert
allow-application-access: [myapp.api.server]
auto-cert:
  issuing-ca: myapp-ca
  cert-type: server          # server | client
  ttl: 15d
  refresh-threshold: 10d     # renewed automatically
  host: myapp-api            # CN
  alt-name:
    - type: DNSName
      value: myapp-api
```
The secret exposes keys `cert.pem`, `cert.key`, `ca-cert.pem`. Mounted per service instance, each instance gets its own cert with its app-network IP, ingress IP and DNS name added as SANs.

## Application spec

Top level: `name`, `version`, `upgrade-from`, `services`, `network`, `labels`, `resources`, `icon`, `on-mutable-variable-change`.

```yaml
name: myapp
version: "1.0"                       # quoted string; if set, MUST change on every edit
services:
  - name: api
    mode: replicated
    replicas: 1                      # per site
    variables:                       # service-scoped; referenced as ${NAME}
      - name: DB_USER
        value-from-vault-secret:
          vault: myapp
          secret: credentials
          key: username
      - name: LOG_LEVEL
        value: "info"
    volumes:
      - name: credentials
        vault-secret:                # NOT vault-secret-ref (some docs pages show the old name)
          vault: myapp
          secret: credentials
          file-mode: "400"
          file-ownership: "0:0"
      - name: tls
        vault-secret:
          vault: myapp
          secret: server-cert
      - name: config
        config-map:
          items:                     # `items` list is required
            - name: app.conf
              data: |
                user=${DB_USER}
                site=${SYS_SITE}
      - name: data
        persistent-volume:           # survives app removal until manually deleted
          size: 10 GB
      - name: cache
        ephemeral-volume:            # lives while service stays on the host
          size: 1 GB
    containers:
      - name: server
        image: registry.example.com/myorg/server:1.4.2
        env:
          DB_USER: ${DB_USER}
          LOG_LEVEL: ${LOG_LEVEL}
          SITE_NAME: ${SYS_SITE}
        mounts:                      # keyed by volume-name
          - volume-name: credentials
            mount-path: /run/secrets          # one file per key: /run/secrets/password
          - volume-name: tls
            files:
              - name: cert.pem
                mount-path: /etc/tls/tls.crt
              - name: cert.key
                mount-path: /etc/tls/tls.key
              - name: ca-cert.pem
                mount-path: /etc/tls/ca.crt
          - volume-name: config
            files:
              - name: app.conf
                mount-path: /etc/myapp/app.conf
          - volume-name: data
            mount-path: /var/lib/myapp
        probes:
          liveness:
            http: { port: 8080, path: /healthz }
            initial-delay: 10s
            period: 10s
            failure-threshold: 3
          readiness:
            exec: { cmd: ["/bin/ready.sh"] }
            period: 10s
        # cmd: [...]  entrypoint: [...]  user: "1000:1000"
        # cpus: 0.5   memory: 256 MiB   container-log-size: 100 MiB
        # container-layer-size: 0 B   (read-only root fs)
        # on-mounted-file-change: { restart: true }   (default restarts on secret change)
    network:
      ingress-ip-per-instance:       # IP on the site network; mutually exclusive with host: true
        protocols:
          - name: tcp
            port-ranges: "8080,8443" # quoted string
        inbound-access:
          allow-all: true            # or rules: {192.0.2.0/24: allow}, default-action
      outbound-access:               # outbound is restricted unless granted here
        allow-all: true              # or default-action + rules
    placement:
      match-host-labels: gpu = true
      # preferred-anti-affinity: { services: [other-svc] }
# network:
#   shared-application-network: my-net   # app-level: talk to other apps on same name
```

Other service fields: `init-containers` (same shape as containers), `share-pid-namespace`, `delayed-shutdown`, `site-dns-records`, `vm`. Container extras: `devices`, `gpu`, `security`, `additional-capabilities`, `approle`, `user-namespace`, `shm-size`, `shutdown-timeout`.

### Variables

- Declared per service under `variables` (`value` or `value-from-vault-secret`), used as `${NAME}`.
- System variables: `SYS_SITE`, `SYS_SITE_LABELS[label]`, `SYS_TENANT`, `SYS_HOST`, `SYS_GLOBAL_DOMAIN`, `SYS_SERVICE_INSTANCE_INDEX`, `SYS_INGRESS_IPV4_ADDRESS`, `SYS_APP_NET_IPV4_ADDRESS`, `SYS_GW_NET_IPV4_ADDRESS`, `SYS_HOST_DEVICE_LABELS`, `SYS_DNS_ZONES`, `SYS_API_CA_CERT`, `SYS_APPROLE_SECRET_ID`, `SYS_CONTAINER_CPUS`, `SYS_CONTAINER_MEMORY`.
- Expansion works in: container `cmd`, `entrypoint`, `env` values, `container-log-size`, mount paths, probe `exec.cmd`, `config-map.items[].data`, `vault-secret.vault/secret`, `value-from-vault-secret.vault/secret/key`, `replicas`, ingress `match-*-labels`, DNS records. Not in `image`.
- Instance-level vars (host, IPs, instance index) are not available at scheduling time, so don't use them in `replicas` or placement.
- `on-mutable-variable-change` (app level): `restart-service-instance` (default) | `upgrade-application` | `ignore`. It reacts to mutable vars like `SYS_SITE_LABELS`.

### Per-site secrets pattern

One vault per site (`config-<site>`, each `distribute.sites: [<site>]`), referenced as `vault: config-${SYS_SITE}`. For groups, use a site label: `vault: config-${SYS_SITE_LABELS[region]}`. Each per-site vault needs the same secret name and `allow-application-access`.

### Versioning

- No `version` + deployment `application-version: "*"` = every edit rolls out immediately (good for dev).
- With `version`, every spec change needs a new version string. Add `upgrade-from` entries for controlled upgrades. The system keeps the last 3 versions.

## Application deployment

```yaml
name: myapp-dep
application: myapp
application-version: "1.0"           # or "*" = latest
placement:
  match-site-labels: >
    system/type = edge               # or system/name = <site>, or custom labels (case sensitive)
deploy-to-sites:                      # optional rollout control
  sites-in-parallel: 2
  healthy-time: 10m
  # canary-sites:
  #   match-site-labels: city = stockholm
  # canary-healthy-time: 15m
```
Operations: `supctl do application-deployments myapp-dep pause|resume|redeploy|canary-continue`. Run `redeploy` after new sites start matching the labels.

## Checklist before delivering

- [ ] Every `vault`/`secret`/`key` referenced in the app exists in the vault/secret files (names match exactly, including `${SYS_SITE}` expansions).
- [ ] Each secret's `allow-application-access` lists every `app.service.container` that mounts it or reads it through a variable. No `"*"` unless the user asked for it.
- [ ] Vault `distribute` covers the deployment's sites: `deployments: [<deployment name>]` matches the deployment's `name`.
- [ ] Any `auto-cert.issuing-ca` CA exists and is distributed to the same sites.
- [ ] Every `mounts[].volume-name` matches a service `volumes[].name`. `files[].name` matches a config-map item name or a secret key.
- [ ] `config-map` uses `items:`. Vault volumes use `vault-secret:`.
- [ ] `version` and `port-ranges` are quoted strings. Images have explicit tags.
- [ ] Outbound access is granted if the app calls out. Ingress exists if it must be reached from the site LAN.
- [ ] No real secret values in committed files.
- [ ] `apply.sh` creates resources in dependency order.

If the schema can't be confirmed (no supd tools, docs unreachable), say so and point to the matching redoc section so the user can validate with `supctl create ... ` against `?dry-run=true`.
