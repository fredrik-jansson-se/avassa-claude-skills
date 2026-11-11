---
name: avassa-app-spec
description: Write deployable Avassa YAML in supctl patch format (x-path/x-operation headers) - application specs, application deployments, Strongbox vaults and secrets, and TLS CAs with auto-generated certs. Use when the user asks to deploy, package or configure an app/container on Avassa, write an Avassa application spec or deployment, create Strongbox vaults/secrets, or produce supctl payloads.
---

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
3. **Write the payloads** in `supctl patch` format (see below), using one of two output layouts:

   **Single file (default).** All payloads go in one multi-document YAML file, in dependency order, each document starting with `---`:
   ```
   <app>/
     <app>.yaml     # CA, vault, secrets, application, deployment, in that order
     README.md      # how to apply (supctl patch < <app>.yaml) and test
   ```

   **One file per payload.** Use this when the user asks for it (e.g. "separate files", "one file per resource", "like the avassa examples"), or when the target directory already uses that layout. Number the files in apply order as in Avassa's public examples (https://gitlab.com/avassa-public/application-examples, e.g. `mosquitto/`):
   ```
   <app>/
     01-tls-ca.yaml              # optional TLS CA (only if auto-cert is used)
     02-vault.yaml               # vault
     03-cert-secret.yaml         # optional auto-cert secret
     04-credentials-secret.yaml  # secret(s) in that vault
     05-app.yaml                 # application
     06-deployment.yaml          # application-deployment
     README.md                   # how to apply (supctl patch commands) and test
   ```
   Drop the files that aren't needed and renumber so the numbers stay contiguous.

   The user can switch layouts at any time ("split it up", "merge into one file"). Convert the existing output rather than regenerating it, and keep the content identical. The payloads are the same in both layouts; only the file boundaries change.
4. **Run the checklist** at the end before handing over.

Never put real secret values into files that may be committed. Use obvious placeholders (`CHANGE_ME`) and say so.

## supctl patch format

Each payload is a YAML document that starts with `---` and two header keys telling `supctl patch` where and how to write it, followed by the resource body exactly as the REST API expects it. `supctl patch` accepts several such documents in one file and applies them in the order they appear:

```yaml
---
x-path: /v1/config/applications/myapp     # REST path of the resource itself (ends in its name)
x-operation: replace                      # replace (PUT: create or overwrite) | create (POST: fails if it exists)
name: myapp
...
```

- Use `x-operation: replace` by default: it is idempotent, so the same files work for the first deploy and for updates.
- `x-path` always ends with the resource's own name, which must equal the body's `name`.
- Paths (note hyphens, not underscores):

  | Resource               | `x-path`                                                   |
  |------------------------|------------------------------------------------------------|
  | TLS CA                 | `/v1/config/strongbox/tls/ca/<ca>`                         |
  | Vault                  | `/v1/config/strongbox/vaults/<vault>`                      |
  | Vault secret           | `/v1/config/strongbox/vaults/<vault>/secrets/<secret>`     |
  | Application            | `/v1/config/applications/<app>`                            |
  | Application deployment | `/v1/config/application-deployments/<deployment>`          |
  | Resource profile       | `/v1/config/system/resource-profiles/<profile>`            |
  | Policy                 | `/v1/config/policy/policies/<policy>`                      |

All examples below show each payload as a full document including the header. In the single-file layout, concatenate them in apply order:

```yaml
---
x-path: /v1/config/strongbox/vaults/myapp
x-operation: replace
name: myapp
distribute:
  deployments: [myapp]
---
x-path: /v1/config/strongbox/vaults/myapp/secrets/credentials
x-operation: replace
name: credentials
...
---
x-path: /v1/config/applications/myapp
x-operation: replace
name: myapp
...
---
x-path: /v1/config/application-deployments/myapp
x-operation: replace
name: myapp
...
```

## Apply order (dependencies)

CA → vault → secrets → application → application-deployment. In the single-file layout this is the order of the documents in the file. In the per-file layout, the numbered file names encode it.

Single file:
```sh
supctl patch < myapp.yaml
```

One file per payload:
```sh
supctl patch < 01-tls-ca.yaml
supctl patch < 02-vault.yaml
supctl patch < 03-cert-secret.yaml
supctl patch < 04-credentials-secret.yaml
supctl patch < 05-app.yaml
supctl patch < 06-deployment.yaml
# or: for f in [0-9][0-9]-*.yaml; do supctl patch < "$f"; done
```
To update, edit the file and run `supctl patch < file` again. Re-applying the single file re-sends every payload; that is harmless with `replace`, but bump the application `version` only when the application itself changed. Check with `supctl show applications myapp` (look for `image-status: present`) and `supctl show application-deployments myapp`. Export an auto-generated CA cert with `supctl -j show strongbox tls ca myapp | jq -r '."ca-cert"' > ca-cert.pem`.

The same payloads can also be pasted into the Control Tower UI, or sent to the REST API at the `x-path` (PUT for `replace`, POST to the parent collection for `create`); drop the `x-*` keys when doing that.

## Strongbox vault

```yaml
---
x-path: /v1/config/strongbox/vaults/myapp
x-operation: replace
name: myapp                 # vault name
distribute:                 # REQUIRED in practice: secrets go nowhere unless distributed
  deployments:              # same sites as these application-DEPLOYMENT names (not app names)
    - myapp
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
---
x-path: /v1/config/strongbox/vaults/myapp/secrets/credentials
x-operation: replace
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

`x-operation: replace` overwrites the whole secret, so the file must contain every key. To change a single key without touching the others, use plain supctl instead: `supctl merge strongbox vaults <vault> secrets credentials <<EOF\ndata:\n  password: new\nEOF`. CAS-protected vaults (`cas-required: true`) need `supctl -q version=<n> ...` and are not a good fit for patch files.

### Auto-generated TLS certificate

```yaml
# TLS CA (01-tls-ca.yaml in the per-file layout)
---
x-path: /v1/config/strongbox/tls/ca/myapp-ca
x-operation: replace
name: myapp-ca
ttl: 2y
cert-key-type: rsa
auto-renew:
  renew-threshold: 145d
  activate-threshold: 90d
distribute:
  deployments: [myapp]
```
```yaml
# auto-cert secret (03-cert-secret.yaml in the per-file layout)
---
x-path: /v1/config/strongbox/vaults/myapp/secrets/server-cert
x-operation: replace
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
---
x-path: /v1/config/applications/myapp
x-operation: replace
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
---
x-path: /v1/config/application-deployments/myapp
x-operation: replace
name: myapp                          # often the same as the application name
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
Operations: `supctl do application-deployments myapp pause|resume|redeploy|canary-continue`. Run `redeploy` after new sites start matching the labels.

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
- [ ] Every payload document starts with `---`, `x-path` and `x-operation`. The last `x-path` segment equals `name`, and secret paths use the right vault.
- [ ] Payloads are in dependency order (CA → vault → secrets → app → deployment): document order in the single file, or file numbering in the per-file layout. The README shows the matching `supctl patch` command(s).
- [ ] The layout is single-file unless the user asked for separate files (or the directory already uses them).

If the schema can't be confirmed (no supd tools, docs unreachable), say so and point to the matching redoc section (https://avassa-api.redoc.ly, tags `Applications`, `Application-Deployments`, `Vault`) so the user can check the fields.
