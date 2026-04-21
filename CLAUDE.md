# Wheelhouse Geoscience — project notes

Three candidate builds of the marketing site live as sibling folders and get bundled together under `/review/` for client review on HostGator.

## Folder map

```
/Users/ct/Code/
├── wheelhousegeo/          v1 — original multi-page build (current production)
├── wheelhousegeo_v2/       v2 — warm paper / editorial single-page
├── wheelhousegeo_v3/       v3 — dark blueprint single-page
└── wheelhousegeo-bundle/   v1 + v2 + v3 + selector index.html (review staging)
```

- **v1** is multi-page (index, services, about, founder, portfolio). Assets in `newassets/`, `logo/`, plus loose images at root.
- **v2** and **v3** are each single-page with `index.html`, `styles.css`, `script.js`, `assets/`.
- **bundle** is assembled by `rsync`-ing v1, v2, v3 into `v1/`, `v2/`, `v3/` subfolders with a neutral selector `index.html` at root.

## Local preview

Each version is a static site — serve any folder with Python and open it:

```
cd <folder> && python3 -m http.server <port>
```

Ports used previously: v2=8765, v3=8766, bundle=8767.

## Deploying the review bundle to HostGator

HostGator account `ijxvqate` on `sh00059.hostgator.com`. **Shell access is disabled** — only SFTP and SCP work; don't try `ssh host <cmd>` or `rsync` (rsync needs a remote shell).

### One-time setup

- Private key: `~/.ssh/hostgator_wheelhouse` (chmod 600)
- Passphrase lives in macOS Keychain (added via `ssh-add --apple-use-keychain`)
- Fingerprint: `SHA256:5J/sB1Fmd0vR1M6zEJ8WID4ovE1ehe6vl29WNAkpof0`
- Key must be marked **Authorized** in cPanel → SSH Access → Manage SSH Keys

### Rebuild the bundle + zip

```
# refresh v1/v2/v3 into the bundle
rsync -a --exclude='.git' --exclude='.DS_Store' /Users/ct/Code/wheelhousegeo/     /Users/ct/Code/wheelhousegeo-bundle/v1/
rsync -a --exclude='.DS_Store'                  /Users/ct/Code/wheelhousegeo_v2/ /Users/ct/Code/wheelhousegeo-bundle/v2/
rsync -a --exclude='.DS_Store'                  /Users/ct/Code/wheelhousegeo_v3/ /Users/ct/Code/wheelhousegeo-bundle/v3/

# strip source/working files that shouldn't be on a public web server
find /Users/ct/Code/wheelhousegeo-bundle -type f \
  \( -name '*.pptx' -o -name '*.psd' -o -name '*.ai' -o -name '*.tif' -o -name '*.tiff' -o -name '*.eps' \) \
  -delete

# optional: zip for manual cPanel upload
cd /Users/ct/Code && rm -f wheelhousegeo-review.zip && \
  zip -rq wheelhousegeo-review.zip wheelhousegeo-bundle -x '*.DS_Store' -x '*/__MACOSX*'
```

### Push to the server (preferred — SSH/SCP)

Target directory: `~/public_html/review/`. Leaves the production site untouched.

```
# ensure target dir exists
sftp -i ~/.ssh/hostgator_wheelhouse -P 2222 ijxvqate@sh00059.hostgator.com <<'EOF'
-mkdir public_html/review
ls public_html/review
bye
EOF

# upload bundle contents (not the bundle folder itself)
cd /Users/ct/Code/wheelhousegeo-bundle && \
scp -i ~/.ssh/hostgator_wheelhouse -P 2222 -r \
  index.html logo-full.svg logo-full-dark.svg v1 v2 v3 \
  ijxvqate@sh00059.hostgator.com:public_html/review/

# verify
curl -sI -o /dev/null -w "%{http_code}\n" https://wheelhousegeo.com/review/
```

Live review URL: **https://wheelhousegeo.com/review/** (selector is `noindex,nofollow`).

### Promoting a chosen version to production (future)

Production currently lives at the domain root. When the client picks a version, a safe swap is:

1. Back up current root: `scp -r ... public_html/* ./prod-backup-<date>/` (locally).
2. Upload the chosen version's files (flattened — no `v2/` or `v3/` prefix) to `~/public_html/`.
3. Keep `/review/` up for a bit in case of rollback, then remove via sftp.

**Never** `scp -r chosen/ ...:public_html/` — that'd nest the site a level too deep. Always upload the *contents* of the chosen folder, not the folder itself.

## Assets & variants

- `logo-mark.png` / `logo-tagline.png` / `logo-full.svg` are the original brand assets, designed for light backgrounds. They do not read on dark backgrounds as-is.
- `v3/assets/logo-full-dark.svg` — original SVG with colors remapped: `#333333` → `#F2EAD3` (cream), `#1A567B` → `#E56A3A` (rust). Built via sed from `logo-full.svg`.
- `v3/assets/logo-mark-dark.svg` — mark-only extract of the dark variant (lines 7–282 of the full SVG wrapped with viewBox `0 0 250 342`).
- `v3/assets/share-card.png` — 1200×630 OG share image. Built from `share-card.svg` using `rsvg-convert` (installed via Homebrew). Regenerate with:

  ```
  rsvg-convert -w 1200 -h 630 -b '#0A0F15' \
    /Users/ct/Code/wheelhousegeo_v3/assets/share-card.svg \
    -o /Users/ct/Code/wheelhousegeo_v3/assets/share-card.png
  ```

Social platforms (FB, Twitter/X, LinkedIn, Slack) cache OG images aggressively — after updating, re-scrape via each platform's debugger tool to force refresh.

## Contact forms

All three versions submit to `https://formsubmit.co/info@wheelhousegeo.com` via POST. The first submission from a new domain requires an activation click in the inbox (FormSubmit's anti-spam). Honeypot field `_honey` is already present.

## Gotchas to remember

- **Don't enable `scroll-snap-type` on `html`** for these layouts — the last section traps the user before the footer. Sections have `scroll-snap-align: start` declarations that are harmlessly inert without the type on html. Removed from v2 and v3.
- **HostGator shell is off** — `ssh host "echo hello"` returns "Shell access is not enabled". Use SFTP (`sftp`) or SCP (`scp`) only. `rsync` over SSH will not work.
- **SSH key passphrase** lives in Keychain — if you move to a new machine, re-run `ssh-add --apple-use-keychain ~/.ssh/hostgator_wheelhouse` there.
- **`public_html` currently contains a stray `website_9b81ef20/` folder** (HostGator's default placeholder). Don't touch it unless intentionally cleaning the account.
- **v1 has source files** (`.pptx`, `.psd`, `.ai`, `.tif`, `.eps`) in `newassets/` and `logo/`. Keep them locally but always strip them before shipping to the server (bundle rebuild command above does this).
