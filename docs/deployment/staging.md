# Staging Deployment — Firebase Hosting

Staging is deployed to Firebase Hosting on every merge to the `staging` branch.

| Item             | Value                                                                      |
| ---------------- | -------------------------------------------------------------------------- |
| URL              | [https://tensor-labz-website.web.app](https://tensor-labz-website.web.app) |
| Firebase project | `tensor-labz-website`                                                      |
| Trigger branch   | `staging`                                                                  |

---

## Deploy flow

```mermaid
flowchart LR
    Dev["dev branch\n(local changes)"] -- "PR merge" --> Staging["staging branch"]
    Staging -- "npm run deploy:staging" --> Build["npm run build\ntsc -b && vite build"]
    Build -- "dist/" --> Firebase["Firebase Hosting\nCDN upload"]
    Firebase -- "live at" --> URL["tensor-labz-website.web.app"]
```

---

## Deploy manually

```bash
# From tensor-labz-website/
npm run deploy:staging
# Equivalent to: npm run build && firebase deploy --only hosting
```

---

## Caching strategy

```mermaid
flowchart TB
    Request["Browser request"] --> Router{File type?}
    Router -- "*.js *.css *.woff2" --> Long["Cache-Control:\nmax-age=31536000, immutable\n(1 year — content-hashed)"]
    Router -- "*.jpg *.png *.webp" --> Short["Cache-Control:\nmax-age=86400\n(1 day)"]
    Router -- "any route (SPA)" --> Rewrite["Rewrite to /index.html\n(React Router handles)"]
```

---

## Firebase CLI setup (new machine)

```bash
# Install without sudo
npm config set prefix '~/.npm-global'
echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
npm install -g firebase-tools

# Authenticate
firebase login                  # with browser
firebase login --no-localhost   # on headless server

# Verify
firebase projects:list
# should show: tensor-labz-website
```
