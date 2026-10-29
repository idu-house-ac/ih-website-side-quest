# Idu House one-page sites

Two static pages, deployed as two separate projects so the investor page's access control can never leak onto the public page or the other way around. No build step, no framework.

```
public/      This Is Idu House (open, indexed)         -> GitHub Pages   https://whatis.iduhouse.com
investor/    Idu House Pre-Seed (gated, never indexed)  -> Vercel         https://investor.iduhouse.com (password)
source/      original reel (.mov), git-ignored
```

DNS for both lives at the businessidentity.llc nameservers that serve iduhouse.com:

| Host | Type | Value |
|---|---|---|
| `whatis` | CNAME | `idu-house-ac.github.io` |
| `investor` | CNAME | the `*.vercel-dns-016.com` target Vercel assigned to the project |

## Rules that apply to both pages

- Copy is approved. Do not reword, reorder or "improve" it. If something looks wrong, raise it with Amadin.
- No em dashes, en dashes or literal asterisks in visible text. The company is a "Black-owned independent studio". The platform is "Threshold". No physical location.
- The public page carries no prices, raise terms or financial figures. It never links to the investor page.
- The investor page has no analytics or third-party scripts beyond Google Fonts, and is never linked from anywhere.

Run this after any edit. It should print nothing:

```bash
python3 - <<'PY'
import re, html
for f in ["public/index.html", "investor/index.html"]:
    s = open(f, encoding="utf-8").read()
    s = re.sub(r"<(style|script)\b.*?</\1>", "", s, flags=re.S)
    text = html.unescape(re.sub(r"<[^>]+>", " ", s))
    for bad in [chr(0x2014), chr(0x2013), "*", "Passport", "content studio", "incubator"]:
        if bad in text:
            print(f, "contains", repr(bad))
    if f.startswith("public") and "$" in text:
        print(f, "contains a dollar sign")
PY
```

## Assets each page expects

Both `public/assets/` and `investor/assets/` need:

| File | What it is |
|---|---|
| `reel-1080.mp4` | The Idu House reel, 1920 wide, silent, H.264 |
| `reel-poster.jpg` | One frame from the reel, shown before playback and under reduced motion |

`public/assets/` also needs:

| File | What it is |
|---|---|
| `favicon.png`, `favicon-32.png` | The gold house emblem on a transparent background, 192 and 32 px |
| `apple-touch-icon.png` | The same emblem on ebony, 180 by 180 |
| `og-image.jpg` | 1200 by 630 frame from the reel, used for link previews |

The public page's canonical and Open Graph URLs are hard-coded in the `<head>` of `public/index.html`. Update them if the public hostname changes.

## Encoding the reel

Put the original in `source/` (it stays out of git), then:

```bash
ffmpeg -i "source/IH Reel 1.mov" -an -c:v libx264 -preset slow -crf 26 \
  -pix_fmt yuv420p -movflags +faststart -vf "scale=1920:-2" reel-1080.mp4
```

```bash
ffmpeg -ss 2 -i reel-1080.mp4 -frames:v 1 -q:v 3 reel-poster.jpg
```

```bash
ffmpeg -ss 2 -i reel-1080.mp4 -frames:v 1 -q:v 3 -vf "scale=1200:630:force_original_aspect_ratio=increase,crop=1200:630" og-image.jpg
```

The current file was encoded at CRF 26 (15.1 MB). CRF 23 came out at 22.3 MB and looked no better at 1:1. The poster and OG image use the 2-second frame as a placeholder until Amadin picks a second. Copy the mp4 and poster into both `assets/` folders, and the OG image into `public/assets/`.

The reel autoplays muted, loops and plays inline. When the visitor has reduced motion turned on, it does not autoplay. The poster shows with a "Play reel" button that starts and stops the loop. That behaviour is the small inline script directly after each `<video>`.

## Updating the public page

1. Edit `public/index.html`.
2. Run the scan above.
3. Commit and push to `main`. The workflow in `.github/workflows/deploy-public.yml` publishes the `public/` folder to GitHub Pages on every push that touches it.

One-time setup (Amadin's call before it goes live): in the repository settings, set Pages to deploy from GitHub Actions, add the custom domain, and add a `public/CNAME` file (already present, containing `whatis.iduhouse.com`). The DNS record is a CNAME from the chosen subdomain to `idu-house-ac.github.io`, added wherever iduhouse.com's DNS is managed. Do not touch the apex or `www` records, which belong to Squarespace.

## Updating the investor page

1. Edit `investor/index.html`.
2. Run the scan above.
3. Commit and push to `main`. The Vercel project redeploys from the `investor/` folder.

Already done: a Vercel project was created from this repo with the root directory set to `investor` and framework preset "Other". Turn on password protection in the project's deployment protection settings, covering production and preview deployments. Add the custom domain in Vercel and create the CNAME it asks for.

What protects the page:

- Vercel password protection in front of every request, so a signed-out visitor sees the gate and never the page.
- `<meta name="robots" content="noindex, nofollow">` in the page.
- `X-Robots-Tag: noindex, nofollow` on every response, set in `investor/vercel.json`.
- `investor/robots.txt` disallows everything.

Vercel's password protection uses a single shared password set in the dashboard. To let someone in, send them the password out of band. To revoke access, change the password. For per-person access tied to email addresses, an identity-based gate such as Cloudflare Access would replace this; that is a hosting decision for Amadin and securities counsel.

Nobody shares the investor URL until securities counsel has confirmed the setup.

## Local preview

```bash
python3 -m http.server 8081 --directory public
```

```bash
python3 -m http.server 8082 --directory investor
```

Then open http://127.0.0.1:8081/ and http://127.0.0.1:8082/.
