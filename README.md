<h1 align="center">screenview</h1>

<p align="center">
  <b>Token-efficient screen inspection &amp; control for AI agents on Linux/X11.</b><br>
  <i>See the screen with the cheapest method that answers the question — not a screenshot every time.</i>
</p>

<p align="center">
  <img alt="platform" src="https://img.shields.io/badge/platform-Linux%2FX11-blue">
  <img alt="language" src="https://img.shields.io/badge/python-3.x-3776AB">
  <img alt="license" src="https://img.shields.io/badge/license-MIT-green">
</p>

---

When an AI agent "looks" at a screen by reading a full-resolution PNG, that single image
costs roughly **1,200–1,600 tokens**. Most of the time the agent only needs *text* — which
window is open, what the terminal says, the network state — none of which requires a picture.
`screenview` exposes the screen through a **cheapest-first** ladder so the agent (or you)
picks the lightest method that actually answers the question.

### Highlights

- 🪜 **Cheapest-first ladder** — CLI text → accessibility tree → OCR → downscaled image → (rarely) full screenshot.
- 🌐 **Token-efficient browser** — drive any web page as ~100–400 token *element refs*, click by `ref` (no pixel coordinates).
- 🔑 **Auth by session-reuse** — log the automation browser in by borrowing your existing cookies (no Google "browser not secure" wall).
- 🎮 **Full GUI control** — mouse, keyboard, window targeting via `xdotool`.
- 📉 **Measured 60–99% token savings** on real web/text tasks (see [Performance](#performance--benchmarks)).

### Table of contents

- [Philosophy](#philosophy-cheapest-method-that-answers-the-task)
- [Install](#install)
- [Usage](#usage) — [inspect](#inspect-text--cheap) · [read](#read-ocr--cheap-no-image) · [look](#look-smart-auto) · [shot](#see-downscaled-image--only-when-layout-matters) · [control](#control-drive-the-gui) · [web](#web-token-efficient-browser--best-for-any-web-task) · [auth](#authenticating-reuse-your-existing-session-no-fresh-login)
- [When each tier wins](#when-each-tier-wins)
- [Limitations](#limitations)
- [Performance & benchmarks](#performance--benchmarks)
- [Security uses & responsible use](#security-uses--responsible-use)
- [Built on / credits](#built-on--credits)
- [License](#license)

## Philosophy: cheapest method that answers the task

| Tier | Method | What you get | Approx. tokens/look |
|------|--------|--------------|---------------------|
| 1 | CLI text (`windows`, `apps`) | facts: open windows, app list, geometry | ~20–100 |
| 2 | Accessibility tree (`tree`) | native UI as structured text (buttons, fields, labels) | ~150–400 |
| 3 | OCR (`read`) | all on-screen *text*, no image sent | ~150–400 |
| 4 | Downscaled JPG (`shot`) | layout/visuals when they actually matter | ~400–700 |
| — | Full-res PNG | almost never needed | ~1,200–1,600 |

A full screenshot is the **last** resort, not the first.

## Install

Dependencies (Debian/Kali):

```bash
sudo apt install -y tesseract-ocr python3-pyatspi xdotool imagemagick
# xfce4-screenshooter optional; ImageMagick's `import` is used for capture
```

Put it on your PATH:

```bash
chmod +x screenview
sudo ln -sf "$PWD/screenview" /usr/local/bin/screenview
```

Requires an X11 session (`echo $XDG_SESSION_TYPE` → `x11`). Reads `$DISPLAY` (defaults `:0.0`).

## Usage

### Inspect (text — cheap)

```bash
screenview windows              # list visible windows: [id] WxH+X+Y  title
screenview apps                 # list accessibility-exposed applications
screenview tree --active        # active window's UI as a text tree
screenview tree --app Firefox   # a specific app's UI tree (match by name)
screenview tree --app Firefox --depth 6
```

### Read (OCR — cheap, no image)

```bash
screenview read --active                 # OCR the active window to plain text
screenview read --window 60817415        # OCR a specific X11 window id
screenview read --region 0,0,800,600     # OCR a rectangle  X,Y,W,H
```

### Look (smart auto)

```bash
screenview look          # try the a11y tree; if the app is custom-drawn, fall back to OCR
```

### See (downscaled image — only when layout matters)

```bash
screenview shot                          # full screen → ~1000px JPG in ~/Screenshots, prints path
screenview shot --active                 # just the active window
screenview shot --region 0,0,800,600     # a rectangle
screenview shot --scale 1200 --quality 80 --out /tmp/x.jpg
```

### Control (drive the GUI)

```bash
screenview click 640 400        # move mouse and left-click at x,y
screenview type "hello world"   # type a string
screenview key ctrl+s           # send a key combo (xdotool syntax)
```

### Web (token-efficient browser — best for any web task)

Wraps [`playwright-cli`](https://www.npmjs.com/package/@playwright/cli). A page becomes a
compact **YAML snapshot of element refs** (~100–400 tokens) — `heading`, `link`, `button`
each tagged `[ref=eN]` — and you act on elements **by ref**, with no pixel coordinates and no
screenshots. This is the cheapest *and* most reliable way to drive a browser.

```bash
screenview browse https://example.com      # open/navigate + print the snapshot in one step
screenview browse https://app.com --wait 4  # wait 4s for a heavy SPA to load before snapshot
screenview web click e6                     # click the element with ref e6 (no coordinates!)
screenview web fill e3 "myusername"         # fill an input by ref
screenview web type "hello"                 # type into the focused element
screenview web snapshot                     # re-capture refs after the page changes
screenview web goto https://other.site      # navigate the open browser
screenview web close                        # close the browser
```

Install once: `sudo npm install -g @playwright/cli` then
`node $(npm root -g)/@playwright/cli/node_modules/playwright-core/cli.js install firefox`.

**Why this matters:** ~100–400 tokens/page vs ~1,200–1,800 for a full screenshot, and clicking
by `ref` eliminates the coordinate-scaling mistakes that plague pixel-based control. Caveat:
heavy single-page apps may need a moment to load before `snapshot` shows all elements.

### Authenticating: reuse your existing session (no fresh login)

The Playwright browser is a separate session — it is **not** logged into your accounts, and you
**cannot** log it in via "Sign in with Google": Google detects the automated browser and blocks
it with *"this browser or app may not be secure."* Stealth forks (patchright, etc.) are
Chromium-only and still hard-block on Google login, so they don't help either.

The robust fix is **session reuse**: you're already logged in elsewhere (your normal Firefox),
so extract that domain's cookies and inject them — the automated browser is then authenticated
without any login flow, and Google never enters the picture.

```bash
# 1) extract the domain's login cookies from your Firefox profile -> storage-state json
screenview web-auth tryhackme.com
#    -> /home/you/.pw-auth/tryhackme.com.json   (written 0600)

# 2) browse authenticated, by ref
screenview browse https://tryhackme.com/dashboard \
    --storage-state ~/.pw-auth/tryhackme.com.json --wait 5
#    -> snapshot shows your logged-in dashboard (e.g. "Hey <you>", "My Learning")
```

**Security:** the storage-state file contains live session cookies that can impersonate your
account. It is written `0600` under `~/.pw-auth/` and is **gitignored** — never commit it, never
share it. Cookies also expire, so re-run `web-auth` when the session goes stale.

## When each tier wins

- **`windows`/`apps`** — "what's open?", "is app X running?", window geometry for targeting clicks.
- **`tree`** — reading/locating controls in native GTK/Qt apps (button labels, field contents,
  menu items) with exact text and no OCR errors.
- **`read`** — terminals, custom-drawn apps, or anything `tree` returns empty for. Pure text out.
- **`shot`** — only when you must judge *visual* layout, spacing, rendering, or images.

## Limitations

- **No live feed.** Each call is a still snapshot — there is no continuous video stream.
- **AT-SPI coverage varies.** Most GTK/Qt apps expose their UI; some custom-drawn apps
  (e.g. certain terminals) expose almost nothing — `tree` will be sparse and you should use
  `read` (OCR) instead. `look` handles this fallback automatically.
- **X11 only.** Capture and `xdotool` control assume X11, not Wayland.
- **OCR is approximate.** Recognized text can contain minor errors; prefer `tree` when available.

## Performance & benchmarks

Real, measured results from live use (estimates: text ≈ chars ÷ 4; images ≈ w×h ÷ 750 after
the API caps the long edge at 1568 px). Full detail in [PERFORMANCE.md](PERFORMANCE.md) and
[BENCHMARK.md](BENCHMARK.md).

| Task | Old way (full PNG) | screenview | Saving |
|------|--------------------|-----------|--------|
| "Which page is open?" (window title) | ~1,798 | ~18 | **−99%** |
| Read a web page's text (Wikipedia, OCR) | ~1,798 | ~69 | **−96%** |
| See a page, image+text (downscaled JPG) | ~1,798 | ~732 | **−59%** |
| Read a browser page as refs (`browse`) | ~1,798 | ~100–400 | **−80%+** |
| Native app controls (Thunar `tree`, shallow) | ~410 | ~144 | **−65%** |

**Where it does *not* help:** driving a remote-desktop **VNC canvas** (e.g. a TryHackMe
AttackBox). That has no text layer, so every step needs a screenshot — the one case the ladder
can't cheapen. Pure-web tasks, by contrast, run entirely in cheap text.

## Security uses & responsible use

`screenview` is a useful tool for **authorized** security work, and a dual-use one — worth
being clear-eyed about both sides.

**Legitimate uses:**
- **CTF / pentest automation** — drive a target lab through the browser, run recon, read tool
  output as text instead of eyeballing screenshots (demonstrated end-to-end on TryHackMe).
- **Authenticated web-app testing** — `web-auth` lets you test an app *as a logged-in user*
  (reusing your own session) without scripting a login each run — handy for authenticated
  endpoints, access-control checks, and form enumeration.
- **Recon / enumeration** — a `browse` snapshot surfaces every form, input, hidden field, and
  link as structured text, which is exactly what you want for mapping an app.

**Responsible-use boundaries — read this:**
- **Only target systems you own or are explicitly authorized to test** (your boxes, CTF labs,
  engagements with scope). This tool makes interaction easier; it does not grant authorization.
- **`web-auth` is session-cookie extraction** — the same technique infostealer malware uses to
  hijack sessions. On your own machine for your own accounts it is legitimate, but the
  resulting `~/.pw-auth/*.json` files are **as sensitive as your password**: anyone who copies
  one is logged in as you. They are written `0600` and gitignored — keep them secret, never
  commit or share them, and delete them when done.
- Cookies expire; re-run `web-auth` to refresh rather than hoarding old state files.

## Built on / credits

`screenview` is an original CLI that **orchestrates** established open-source tools — it does
not reimplement them. Credit and thanks to:

| Tool | Role | License |
|------|------|---------|
| [@playwright/cli](https://www.npmjs.com/package/@playwright/cli) / Playwright | browser engine (`browse`/`web`) | Apache-2.0 (browser binaries have their own terms) |
| [Tesseract OCR](https://github.com/tesseract-ocr/tesseract) | `read` text extraction | Apache-2.0 |
| AT-SPI / `python3-pyatspi` | `tree` accessibility | LGPL/GPL |
| [xdotool](https://github.com/jordansissel/xdotool) | `click`/`type`/`key` control | BSD-3-Clause |
| ImageMagick | screen capture + downscaling | ImageMagick License |

The session-reuse and Set-of-Mark ideas come from public research; the implementation here
(CLI design, cookie→storage-state extraction, integration, docs) is original work under MIT.

## License

MIT — see [LICENSE](LICENSE). The wrapped tools above retain their own licenses.
