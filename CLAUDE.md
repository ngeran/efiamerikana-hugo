# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Hugo static site with no Dockerfile, content managed by Decap CMS. Nix builds the site and bakes it into an nginx OCI image; `just` orchestrates dev → build → push → deploy. Two deploy targets: a local **k3s** cluster (default) and **Cloudflare Pages** (public URL). Generated from the `omni-nix#hugo` flake template.

## Commands

All run from the devShell (auto-loaded by direnv via `.envrc` → `use flake`).

| Command | Purpose |
| --- | --- |
| `just serve` | Local dev server with live/drafts → http://localhost:1313 |
| `just css` | Compile Tailwind v4 (`site/src/input.css` → `site/static/css/main.css`); run after editing CSS/classes, then commit the output |
| `just check` | Validate the Hugo build (catches broken layouts/frontmatter); no cluster needed |
| `just build` | Build the nginx image via `nix build .#image` → `result` |
| `just push` | Push `result` to the local registry `localhost:5000` via skopeo (not docker) |
| `just deploy` | `kubectl apply` manifests + rollout restart (depends on `push`) |
| `just test` | Local smoke: `docker load` the image, run it, curl `/`, expect 200 |
| `just doctor` | Pre-flight: k3s running, registry reachable, git index clean |
| `just logs` / `just forward` | Tail pod logs / port-forward svc `:80` → local `:8080` |
| `just cf` / `just cf-preview` | Deploy the built static site to Cloudflare Pages (prod / preview branch) |

k3s is on-demand — start it before deploy: `sudo systemctl start k3s`.

## Architecture — the parts that span multiple files

**Three Nix outputs in `flake.nix`** drive everything:
- `packages.image` — an OCI image (the `default` output) built by `dockerTools.buildImage`. `copyToRoot` layers nginx + the built static assets + runtime dirs + a synthetic `/etc/passwd`. There is **no Dockerfile, no `docker build`** — the image is a Nix store path.
- `packages.site` — just the built static site (the Hugo output). This is what `just cf` uploads to Cloudflare, byte-for-byte the same content nginx serves in k3s.
- `devShells.default` — hugo, nodejs_22, just, skopeo, kubectl, wrangler.

**The nginx config is hand-rolled inline in `flake.nix`** (not from nixpkgs's default) because the from-scratch image has no writable `/tmp`, no `/etc/passwd`, and nginx's stock config points pid/temp/logs at read-only paths. Every override matters: `daemon off` (foreground as PID 1, else the container exits and "Completed"-loops), pid + all temp paths under `/tmp`, `error_log /dev/stderr`, listens on **8080** (non-root can't bind <1024), and no `user` directive (the container runs as UID 1000; nginx can't `setuid` when non-root). Writable `/tmp`, `/var/cache/nginx`, and `/var/log/nginx` come from **emptyDir mounts in the manifest**, not from the image.

**The image name `localhost:5000/hugo-site:latest` must stay in lockstep** across three places: `flake.nix` (`imageName`/`imageTag`), the `justfile` (`image`/`tag`), and `manifests/deployment.yaml` (`spec.containers[].image`). Change it in all three.

**Safe-by-default manifest** (`manifests/`): non-root UID/GID 1000 (matches `flake.nix` `config.User`), `readOnlyRootFilesystem`, all capabilities dropped, no privilege escalation, seccomp=`RuntimeDefault`, resource limits, and `/` HTTP probes. The Service fronts the container's `:8080` on `:80`.

## Critical gotcha: Nix reads the git INDEX, not the worktree

`nix build` evaluates files referenced by the flake (e.g. `${./site}`) through the **git index**, so unstaged/untracked edits to `site/` or `flake.nix` are invisible to the build — you'll build stale content with no warning. **`git add` before `just build`/`just deploy`**, or run `just doctor` which flags this. This is the single most common "why didn't my change deploy?" failure.

## Hugo site architecture

All site source lives under `site/`.

**Layout (`site/layouts/`):** `_default/baseof.html` is the shell — `<head>` with Google Fonts + Material Symbols + the prebuilt `css/main.css` link, then `header`/`main`/`footer` partials. `index.html` composes the home page from the section partials in `layouts/partials/`: `header`, `hero`, `about`, `videos`, `gallery`, `analytics`, `contact`, `footer`. Hardcoded copy is gone — each partial pulls text from frontmatter via `.Param "…"`, and the Videos/Gallery partials `range` over their content sections.

The design is mobile-first (Tailwind responsive utilities throughout). The desktop nav is `hidden md:flex`, so `header.html` also renders a hamburger + slide-down `#mobile-menu` panel toggled by a small inline vanilla-JS snippet (open/close, body-scroll lock, close-on-tap). `.section-pad` sets section spacing (`py-12` → `md:py-20`); it lives on each section's content wrapper `<div>`.

**Content (`site/content/`):** `_index.md` holds all landing-page text (Hero, About, section intros, Analytics numbers + platform bars, Contact, socials) as frontmatter. `content/videos/` and `content/gallery/` are **headless** sections — each item carries a `build:` block (`render: never`, `list: local`) so it never emits a standalone URL but is pulled into the home page via `(.Site.GetPage "videos").Pages` (same for gallery). `disableKinds = ["section","taxonomy","term"]` in `hugo.toml` suppresses the unused list pages, keeping the build warning-free.

**Tailwind v4 asset pipeline:** `site/src/input.css` holds `@import "tailwindcss"`, an `@source` glob over the layouts, the `@theme` block (brand colors + `font-serif`/`font-sans`), and `@layer` component classes ported from the original hand-written CSS. It is compiled **outside** Hugo by `just css` → `site/static/css/main.css`, which Hugo serves verbatim — **no Hugo Pipes**, so the Nix image build stays offline and reproducible. Edit `input.css` or any layout class, run `just css`, and commit the regenerated `main.css`. Requires `npm install` once.

**Decap CMS (`site/static/admin/`):** `index.html` loads the editor from CDN; `config.yml` uses a **GitHub backend through an OAuth proxy** (`backend.base_url` → a Cloudflare Worker, `auth_endpoint: /auth`) — *not* Netlify Identity or PKCE. Three collections: a `file` collection editing `site/content/_index.md` (all landing copy **plus the `nav` menu and `footer_links` lists**), and `folder` collections for `site/content/videos` and `site/content/gallery`. **Every collection `file`/`folder` and `media_folder` is repo-relative and prefixed with `site/`** because the Hugo project lives in `site/`, not the repo root (`public_folder: "/images/upload"` is the URL path, no prefix). The nav (`header.html`, desktop + mobile) and footer links (`footer.html`) `range` over those params, so they're fully CMS-editable. Each video/gallery item's headless `build:` block is preserved via an inlined `object` field (no YAML anchors — they parse poorly in some Decap builds). `local_backend: true` allows local editing via `npx decap-server` at `/admin/`; production falls back to the proxy.

## Deployment targets

- **k3s (default):** `just build && just push && just deploy`. Requires the local registry at `localhost:5000` (k3s's bundled registry). Push is plain HTTP via skopeo (`--dest-tls-verify=false`).
- **Cloudflare Pages (public — the live site):** deployed by **GitHub Actions**, not `just cf`. `.github/workflows/deploy.yml` runs on every push to `main`: installs Nix → `nix build .#site` (→ `./result`) → `cloudflare/wrangler-action@v3` uploads to the **`efiamerikana-site`** Pages project → `https://efiamerikana-site.pages.dev`. The project must already exist in Cloudflare (wrangler doesn't create it); GitHub repo secrets `CLOUDFLARE_API_TOKEN` (Account → Cloudflare Pages → Edit) + `CLOUDFLARE_ACCOUNT_ID` are required. Because Decap commits land on `main`, **CMS edits rebuild and go live automatically.** (`just cf` still works as a manual one-shot upload if ever needed.)
