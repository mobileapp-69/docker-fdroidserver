# F-Droid repository server

This Compose setup runs an F-Droid repository server behind an Nginx reverse
proxy. Repository data and signing credentials are stored in the local
`fdroid` directory.

## Requirements

- Docker with Docker Compose.
- Ports and firewall rules appropriate for your deployment. The default
  Compose configuration publishes HTTP on port 8080.

Run the commands below from the directory containing `docker-compose.yml` and
`nginx.conf`.

## 1. Initialize a new repository

For a new repository, generate its configuration and signing key once:

```sh
docker compose run --rm --no-deps --entrypoint fdroid fdroid init --no-prompt
```

This creates `fdroid/config.yml` and `fdroid/keystore.p12`. Keep both files
private and back them up securely. The configuration contains signing
passwords. Do not commit these files or publish them on the web.

**Do not generate a new key for an existing repository.** Its signing key
identifies the repository; changing it can prevent existing clients from
trusting updates. Follow the restoration instructions below instead.

## 2. Restore an existing repository (if applicable)

If this is the same store as an existing repository, restore its original
`config.yml` and keystore into the local `fdroid` directory instead of
initializing a new one. Stop the services first if they are running:

```sh
docker compose stop
```

Use the original configuration and the corresponding keystore together. If
the configuration specifies a different keystore filename or path, restore it
at that location and preserve its signing alias and passwords. Also restore the
repository's APKs and metadata from your backups as needed.

If you already initialized a new repository, securely replace the newly
generated configuration and keystore with your existing backups while the
services are stopped. Keep a secure backup before replacing anything.

## 3. Configure the repository

Edit `fdroid/config.yml` directly, preserving the signing settings. For example:

```yaml
repo_url: https://fdroid.example.com/fdroid/repo
repo_name: CPE F-Droid App Store
repo_description: An app store for Android devices.
repo_icon: logo.png
```

Replace `repo_url` with the actual address clients will use, ending in
`/fdroid/repo`. For local testing, this can be
`http://localhost:8081/fdroid/repo`; `localhost` on another device refers to
that device, not your server. Place your icon at `fdroid/fdroid.png`, or
adjust `repo_icon` to match its location. If the icon is missing, F-Droid
generates a placeholder.

The `FDROID_UPDATE_INTERVAL` setting in `docker-compose.yml` controls how often
the container updates the index. `TZ` controls the container's time zone.

## 4. Start and check the services

```sh
docker compose up -d
docker compose ps
docker compose logs --tail 50 fdroid
```

Visit <http://localhost:8081/fdroid/repo/>. Requests to `/` redirect there.
Nginx removes the `/fdroid/repo/` prefix before forwarding requests to the
F-Droid web server.

The included Nginx configuration serves HTTP only. For a public repository,
provide HTTPS using a TLS-terminating reverse proxy and set `repo_url` to the
public HTTPS address. Do not expose `config.yml` or the keystore.

## 5. Add apps and update the index

Place APKs in `fdroid/repo`. F-Droid periodically runs `fdroid update -c`,
which creates missing metadata skeletons and signs the repository index. Add
the configured repository URL to the F-Droid client. APKs must already be
signed with their app signing keys; the repository signing key signs the
repository index, not the APKs.

### Manually refresh the repository

To update the index immediately, stop the F-Droid service first to avoid
overlapping with its scheduled update, then run a one-off update:

```sh
docker compose stop fdroid
docker compose run --rm --no-deps --entrypoint fdroid fdroid update -c
docker compose up -d fdroid
docker compose logs --tail 50 fdroid
```

Check the one-off update's output for errors; start the service again even if
the update fails, then investigate the error. The one-off command does not run
scripts in `updates`; the normal service runs them before each scheduled
update, including on startup. Nginx cannot serve the repository while the
F-Droid service is stopped.

## Cloudflare Tunnel

For local development, you can use this setting to host your store.

- Fill in the `.env` file with your Cloudflare Tunnel credentials
- Start the container
  - `docker compose --env-file ./.env -f docker-compose-tunnel.yml up -d`
- Remove the container
  - `docker compose --env-file ./.env -f docker-compose-tunnel.yml down`
