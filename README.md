# devopsy-recipe-drupal11

Drupal 11 on a [devopsy](https://github.com/hanoii/devopsy-cli) server: the
image is built on the server from this project, the code is read-only at
runtime, and MariaDB 11 keeps the data. Use it to try build-mode releases, or
as a starting point for a Drupal site.

## Try it

```sh
devopsy @prod release deploy   # build on the server, install or update the site
devopsy @prod drush uli        # a one-time login link
devopsy @prod shell            # bash in the app container, as the app user
devopsy @prod --shell          # a shell on the server, in the current release
devopsy @prod logs -f app
devopsy @prod releases
```

The first `deploy` generates `DB_PASSWORD` and `DRUPAL_HASH_SALT` into the
server's `shared/.env`, then installs the site from `config/sync`. Later
deploys run `drush deploy`: database updates, config import, cache rebuild
and deploy hooks.

## How it works

- **Build mode.** `targets.yaml` sets `mode: build`, so a release uploads the
  whole project as git sees it (about 1 MB) and `deploy` runs `devopsy build`
  there. Composer runs inside the image; `vendor/` and `web/core` are never
  committed or uploaded. Docker's build cache makes a release without
  dependency changes take seconds.
- **The image** (`.devopsy/Dockerfile`): FrankenPHP (Caddy and PHP 8.4). Code
  is owned by root and readable only; the app runs as `app`, UID 10001, never
  root and never 1000, the deploy user on devopsy servers. The UID is fixed in
  the image, not passed around as a setting.
- **Read-only.** The container's root filesystem is read-only. The only
  writable paths are `/tmp` (a tmpfs, also Caddy's state) and `/app/storage`,
  a bind mount of `.devopsy/mnt/storage` (on servers `shared/mnt/storage`)
  holding public files (`sites/default/files` links there) and private files.
- **Ownership.** `shared/mnt` belongs to the deploy user, so the entrypoint
  starts as root, gives `/app/storage` to `app` on the first start, and drops
  to `app` with `setpriv`. `drush` in the image is a wrapper that does the
  same, so `devopsy exec app drush` never runs as root. `devopsy shell`
  opens bash as `app` too (`-u root` for root, `-s database` for another
  service): `compose exec` skips the entrypoint.
- **Settings** (`web/sites/default/settings.php`) come from the environment
  set in `compose.yaml`: database, hash salt, trusted hosts (the public URL
  plus `DEVOPSY_DOMAINS`), and Traefik as the reverse proxy for client IPs
  and HTTPS.
- **Code changes need a release.** OPcache never revalidates (the code cannot
  change), and modules are added with composer, then released.

## Changing configuration

Configuration lives in `config/sync`, installed on the first deploy and
imported on every later one. The code is read-only, so export from a site to
`/tmp` and copy it out:

```sh
devopsy drush config:export --destination=/tmp/sync --yes
devopsy exec -T app sh -c 'cd /tmp/sync && tar c .' | tar x -C config/sync
```

## Rolling back

`devopsy @prod rollback deploy` switches to the previous release and builds
it again (from cache). Database updates are not undone.

## Locally

With a local devopsy-traefik, the site is at
`https://devopsy-recipe-drupal11.localhost`:

```sh
devopsy deploy
devopsy drush uli
```

`.devopsy/mnt` holds the local database and files; `devopsy down` and
deleting it starts over.
