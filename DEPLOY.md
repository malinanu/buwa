# Deploying to CloudPanel

Every push to `main` deploys the site to the CloudPanel server through GitHub Actions (`.github/workflows/deploy.yml`). The workflow copies the files over SSH with rsync and then checks that the site returns HTTP 200. You can also run it manually from **Actions → Deploy to CloudPanel → Run workflow**.

## 1. Create the site in CloudPanel

1. In CloudPanel, go to **Sites → Add Site → Create a Static HTML Site**.
2. Enter the domain (for example `buwa.example.com`). Note the **site user** that CloudPanel creates (for example `buwa`).
3. The site root will be:
   ```
   /home/<site-user>/htdocs/<domain>
   ```
4. On the site's **SSL/TLS** tab, issue a Let's Encrypt certificate.

## 2. Create a deploy SSH key

On your own machine, create a key that is used only for this deploy:

```bash
ssh-keygen -t ed25519 -C "github-actions-buwa" -f ~/.ssh/buwa_deploy -N ""
```

On the server, as root, authorise the public key for the **site user**. Do not use root for deploys.

```bash
SITE_USER=buwa
install -d -m 700 -o $SITE_USER -g $SITE_USER /home/$SITE_USER/.ssh
cat >> /home/$SITE_USER/.ssh/authorized_keys <<'KEY'
<paste the contents of ~/.ssh/buwa_deploy.pub here>
KEY
chown $SITE_USER:$SITE_USER /home/$SITE_USER/.ssh/authorized_keys
chmod 600 /home/$SITE_USER/.ssh/authorized_keys

# rsync must exist on the server
command -v rsync || apt-get install -y rsync
```

Test it from your machine:

```bash
ssh -i ~/.ssh/buwa_deploy buwa@<server-ip> 'ls ~/htdocs'
```

Get the server's host key line for the pinned `known_hosts`. Check that the fingerprint matches the one shown by `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` on the server.

```bash
ssh-keyscan -p 22 -t ed25519 <server-ip>
```

## 3. Add GitHub secrets and variables

In the repo, go to **Settings → Secrets and variables → Actions**.

**Secrets**

| Name | Example |
|---|---|
| `SSH_HOST` | `203.0.113.10` |
| `SSH_PORT` | `22` (optional, defaults to 22) |
| `SSH_USER` | `buwa` |
| `SSH_PRIVATE_KEY` | the full contents of `~/.ssh/buwa_deploy` |
| `SSH_KNOWN_HOSTS` | the line printed by `ssh-keyscan` |
| `DEPLOY_PATH` | `/home/buwa/htdocs/buwa.example.com` |

**Variables**

| Name | Example |
|---|---|
| `SITE_URL` | `https://buwa.example.com` (used for the smoke test) |

The workflow refuses to deploy if `DEPLOY_PATH` is not inside a CloudPanel `htdocs` folder, so a typo cannot wipe another directory. rsync runs with `--delete`, so anything in that folder that is not in the repo is removed.

## 4. Caching and compression (recommended)

In CloudPanel, open **Sites → your domain → Vhost** and add these blocks inside the `server { ... }` block for port 443, above the existing `location /`:

```nginx
# HTML: always revalidate so guests get updates immediately
location = /index.html {
  add_header Cache-Control "no-cache";
}

# Images, SVG and audio: cache for 30 days
location ~* \.(webp|svg|mp3)$ {
  add_header Cache-Control "public, max-age=2592000";
  access_log off;
}

# Compress text assets (the SVG compresses from 62 KB to about 25 KB)
gzip on;
gzip_comp_level 6;
gzip_types image/svg+xml text/css application/javascript;
```

Save, and CloudPanel reloads Nginx.

Because images keep their file names, change a file name (for example `maitland-v2.webp`) when you replace an image, so cached copies are not shown.

## Troubleshooting

- **`Permission denied (publickey)`**: the key is not in the site user's `authorized_keys`, or the file permissions are wrong. Use `chmod 700 ~/.ssh` and `chmod 600 ~/.ssh/authorized_keys`, owned by the site user.
- **`Host key verification failed`**: `SSH_KNOWN_HOSTS` does not match the server. Run `ssh-keyscan` again.
- **`rsync: command not found`**: install rsync on the server (`apt-get install -y rsync`).
- **Smoke test fails but the files are uploaded**: check DNS, SSL and that `DEPLOY_PATH` is the site's actual root.
