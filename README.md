# Oratorian Unraid templates

This repository provides two separate Emby Watch Party templates:

| Template | Stable image | Default host port | Default appdata |
| --- | --- | --- | --- |
| [emby-watchparty](emby-watchparty/emby-watchparty.xml) | `ghcr.io/oratorian/emby-watchparty:2` | `5000` | `/mnt/user/appdata/emby-watchparty` |
| [emby-watchparty-3](emby-watchparty-3/emby-watchparty-3.xml) | `ghcr.io/oratorian/emby-watchparty:3` | `5001` | `/mnt/user/appdata/emby-watchparty-3` |

Both map their published host port to container port `5000`. The 2.x template remains available with its existing configuration and storage paths. The 3.x template supports Emby or Jellyfin and uses its own name, port, and storage defaults.

The old 1.x `emby-watchparty-legacy` catalog entry has been retired. Removing that listing does not uninstall existing containers or delete their appdata.

## Publish the 3.x template

The new template targets the stable `:3` tag. Publish a stable 3.x application image and the updated wiki before publishing this template to the catalog. Do not publish a stable listing that points at a nonexistent image tag. For pre-release testing, temporarily select an exact published beta image instead of `:3`; the stable template itself remains pinned to the 3.x major line.

Template URLs target this repository's `main` branch. Local feature-branch changes become catalog content only after they are merged and pushed there. Both templates use the retained shared icon under `emby-watchparty/`.

## Set up a fresh 3.x installation

1. Prepare the default storage on the Unraid host:

   ```sh
   mkdir -p /mnt/user/appdata/emby-watchparty-3/data \
     /mnt/user/appdata/emby-watchparty-3/avatars \
     /mnt/user/appdata/emby-watchparty-3/logs
   config=/mnt/user/appdata/emby-watchparty-3/config.json
   if [ -e "$config" ] && [ ! -f "$config" ]; then
     printf '%s\n' 'config.json must be a file; correct the host path before starting.' >&2
     exit 1
   fi
   if [ ! -e "$config" ] || [ ! -s "$config" ]; then
     printf '%s\n' '{}' > "$config"
   fi
   ```

   Run preparation with the container stopped. `config.json` must be a writable file
   containing valid JSON. The command initializes only a missing or zero-byte file;
   it preserves every nonempty config. Do not use `touch`. If Docker already created
   a directory at that specific path, inspect and preserve its contents before
   correcting the path and recreating the container from the template. Do not delete
   the appdata directory or overwrite existing settings. For the retained 2.x template,
   use its separate `/mnt/user/appdata/emby-watchparty/config.json` path instead.
2. Install the **emby-watchparty-3** template. Select `emby` or `jellyfin`; supply a URL reachable from the container and a dedicated administrative API key issued by that same server.
3. Generate `SESSION_SECRET` once with `openssl rand -hex 32`. Paste the result into the masked field and keep it across updates.
4. Configure an HTTPS reverse proxy pointing to the chosen host port, default `5001`. Fill **Public HTTPS Web UI** with the complete public URL. Include `APP_PREFIX` if using a subpath.
5. Set **Allowed Public Origins** to the public origin without a path or trailing slash. Set **Behind Reverse Proxy** explicitly and identify the proxy's actual peer address in **Trusted Proxy CIDRs**. A proxy container or host gateway can have a different address from the public proxy hostname.
6. Keep production mode, secure cookies, HLS token validation, the one-worker command, and container bind/port values. Recreate the container after changing boot settings.

Production starts fail closed until required values are valid. `/api/health` can return HTTP 200 with `setup_required` while normal routes and `/api/ready` return 503. Correct the field names shown in container stderr. Accept the installation only after readiness returns HTTP 200 `ready` and a two-device party passes playback, seeking, track changes, and reconnect checks.

The public Web UI field is a Docker label consumed by Unraid, not a backend environment setting. The backend still serves HTTP behind the TLS proxy. Masked fields hide values in the editor; host administrators and container inspection can still read environment values.

Full instructions: [Unraid deployment](https://github.com/Oratorian/emby-watchparty/wiki/Deployment-Unraid), [Configuration](https://github.com/Oratorian/emby-watchparty/wiki/Configuration), and [Reverse Proxy](https://github.com/Oratorian/emby-watchparty/wiki/Reverse-Proxy).

## Keep 2.x and 3.x side by side

Keep their separate container names, host ports, and appdata paths. Never mount the same live `config.json`, avatar database, or avatar directory into both installations.

An already installed 2.x user template may still have `ghcr.io/oratorian/emby-watchparty:latest` saved in **Repository**. Change that installed value to `ghcr.io/oratorian/emby-watchparty:2` before `:latest` moves to 3.x. Updating this catalog XML does not automatically repin every saved installation.

Use separate public hostnames, such as `watchparty2.example.com` and `watchparty3.example.com`. Both application lines use the `ewp_session` cookie with the root path; different ports or `APP_PREFIX` values on one hostname do not isolate their browser sessions. Separate browser profiles can isolate a local comparison test.

Configure hardware acceleration, GPU device mappings, and device permissions on **Emby or Jellyfin**, which runs FFmpeg. The Watch Party templates do not request GPU devices or privileged mode. Practical steps remain in the wiki's [hardware acceleration guide](https://github.com/Oratorian/emby-watchparty/wiki/Hardware-Requirements#hardware-acceleration).

## Deliberate migration from 2.x

The 3.x catalog entry is not an automatic replacement for a saved 2.x user template. Back up the old XML, image digest, environment, networks/ports, `config.json`, data, and avatars while the old app is stopped. Keep that complete installation available for rollback.

If transferring existing identities and settings, copy the stopped backup into the new 3.x appdata paths, including the existing `avatars` directory. Use the separate copy for the candidate; do not point two running installations at the same files. Preserve existing `config.json` instead of initializing it with `{}`.

The 3.x template uses `MEDIA_SERVER_TYPE`, `MEDIA_SERVER_URL`, and `MEDIA_SERVER_API_KEY`. Remove retired `EMBY_SERVER_URL`, `EMBY_API_KEY`, `JELLYFIN_SERVER_URL`, and `JELLYFIN_API_KEY` from any saved candidate template; even empty retired declarations block startup. Old host username/password fields are not needed in the 3.x environment. Host login is an in-app action.

Before normal candidate startup, run the read-only preflight with that candidate's exact environment and mounts, as a temporary one-shot container with restart disabled:

```sh
python -m backend.migration_preflight --root /app --target production --deployment docker
```

Stop for every `ERROR` and `REQUIRED ACTION`, even when the command exits zero. The template explicitly sets `ENABLE_HLS_TOKEN_VALIDATION=true`, overriding an old disabled persisted value. Its `SESSION_EXPIRY=86400` controls the actual cookie lifetime; set `1209600` if retaining the old 14-day lifetime is intended.

Rollback restores the previous image, saved user template/environment, network/port settings, and matching full appdata backup. Changing the image alone does not restore configuration or data.

See [Migration to 3.0](https://github.com/Oratorian/emby-watchparty/wiki/Migration-to-3.0) for the complete upgrade sequence.
