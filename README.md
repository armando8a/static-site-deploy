# static-site-deploy

Reusable GitHub Actions workflow that deploys a static site to
`<subdomain>.8ai-projects.fyi`, hosted on the Hetzner box behind Caddy
(see `armando8a/n8n-stack`).

## How it works

- Uploads the contents of a given build directory over SFTP to
  `/srv/static-sites/sites/<subdomain>/` on the server, using a
  restricted `deploy` user that is chrooted to that directory and can
  only transfer files (no shell access, no ability to touch anything
  else on the server).
- Caddy serves that directory directly at `https://<subdomain>.8ai-projects.fyi`
  (already mounted read-only into the Caddy container). No reload is
  needed for content updates — only for registering a *new* subdomain,
  which is a one-time manual step (add `sites-enabled/<subdomain>.caddy`
  in `n8n-stack` and reload Caddy) done once when a new project is
  bootstrapped.

## Usage from a project repo

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: armando8a/static-site-deploy/.github/workflows/deploy.yml@main
    with:
      subdomain: resume
      build-dir: site
    secrets:
      DEPLOY_SSH_HOST: ${{ secrets.DEPLOY_SSH_HOST }}
      DEPLOY_SSH_USER: ${{ secrets.DEPLOY_SSH_USER }}
      DEPLOY_SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
```

Each project repo needs its own copies of the three secrets
(`DEPLOY_SSH_HOST`, `DEPLOY_SSH_USER`, `DEPLOY_SSH_KEY`) — the same
values every time, since they all deploy through the same restricted
`deploy` user.

## Bootstrapping a brand new project

1. Create the project's static site locally.
2. Create a new private GitHub repo for it, add the workflow above
   (set `subdomain` to the desired name), and set the three secrets.
3. On the server: add `sites-enabled/<subdomain>.caddy` to
   `armando8a/n8n-stack` with a block like:

   ```
   <subdomain>.8ai-projects.fyi {
       root * /srv/sites/<subdomain>
       file_server
       encode gzip
   }
   ```

   then `docker exec n8n-caddy-1 caddy reload --config /etc/caddy/Caddyfile`.
4. Push to `main` — the site deploys automatically.

Because `*.8ai-projects.fyi` is a wildcard DNS record, no DNS change
is needed for new subdomains.
