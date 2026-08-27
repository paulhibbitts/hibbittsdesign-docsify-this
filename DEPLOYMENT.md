# Deployment

This site (the example Docsify-This instance at
<https://docsify-this.hibbittsdesign.org>) is published by a **cron job on the
web server that pulls this repository**, not by GitHub Actions.

## Why not GitHub Actions / FTP?

The host (Reclaim Hosting, cPanel) only offers **SFTP**, and its firewall blocks
GitHub Actions runner IP ranges on every port (21, 990, 22 all time out). So a
push-based deploy from CI isn't possible. Instead the server pulls.

## How it works

1. A shallow clone of this repo lives at `~/repos/docsify-this` on the server.
2. A cron job runs every 10 minutes: fetch `main`, hard-reset to it, then
   `rsync` the working tree into the site directory.
3. `git push` to `main` → live within 10 minutes. Progress/errors go to
   `~/deploy.log`.

The **entire repo** is synced to `~/<site-dir>/`, and the web server's document
root is the `docs/` subfolder inside it.

## One-time setup

SSH into the server (SFTP credentials work for SSH too), then:

```bash
# Shallow clone. A full clone runs the shared-host shell out of memory,
# so limit history and pack memory.
mkdir -p ~/repos
git clone --depth 1 --single-branch -b main \
  -c pack.threads=1 -c pack.windowMemory=32m -c pack.deltaCacheSize=16m \
  https://github.com/paulhibbitts/hibbittsdesign-docsify-this.git \
  ~/repos/docsify-this

# First deploy (adjust the target path to your site directory)
rsync -a --delete --exclude='.git' --exclude='.DS_Store' \
  ~/repos/docsify-this/ ~/docsify-this.hibbittsdesign.org/
```

Confirm the site loads, then verify in cPanel that the domain's **document
root** points at the `docs/` subfolder, e.g.
`/home/<user>/docsify-this.hibbittsdesign.org/docs`.

## The cron job

cPanel → **Cron Jobs** → *Once per ten minutes* (`*/10 * * * *`):

```
cd ~/repos/docsify-this && git fetch -q --depth 1 origin main && git reset -q --hard origin/main && rsync -a --delete --exclude='.git' --exclude='.DS_Store' ~/repos/docsify-this/ ~/docsify-this.hibbittsdesign.org/ >> ~/deploy.log 2>&1
```

Clear the cron **email** field so it doesn't mail every run (output already goes
to `~/deploy.log`).

## Notes

- `git reset --hard origin/main` means the server copy is disposable — never edit
  files there directly; they'll be overwritten.
- `--delete` mirrors exactly: files removed from the repo are removed from the
  site on the next run.
- If a file must exist on the server but not in the repo, add it to the `docs/`
  document root and it's safe (rsync only touches `~/<site-dir>/`, and `.git` is
  excluded).
- Permissions: if you ever see Apache 403 *"Server unable to read htaccess
  file"*, fix directory/file permissions (dirs `755`, files `644`, owned by your
  account) — it's a permission error traversing the docroot, not a missing file.
