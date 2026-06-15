Side project outside my usual PHP world, but worth sharing.

I have a habit of Googling old websites just to see what they looked like. GitHub circa 2008. Wikipedia in 2003. The Wayback Machine has it all, but clicking through snapshots one at a time never gives you the full picture of how a site evolved.

So I built **wayback-video** - a Python CLI tool that takes a URL, pulls its entire Wayback Machine archive, renders each snapshot with a headless browser, and assembles everything into an MP4.

```bash
wayback-video https://github.com --scroll --interval year --from 2008
```

That's it. You get a video showing GitHub's design history from 2008 to today, one full-page scroll per year.

{% youtube a1pCBR3kmI4 %}

## How It Works

The tool runs a 4-phase pipeline:

**Fetch.** Query the Wayback CDX API for all successful HTML captures of the URL. For a site archived monthly over 20 years, that's thousands of records. The tool collapses captures server-side by timestamp, then deduplicates by content digest locally, and selects a few candidates per time period.

**Sample.** Group captures by month, quarter, or year. Pick the best representative from each bucket.

**Render.** Open each archived URL in Playwright using `if_` replay mode (a Wayback URL prefix that strips the toolbar). Archived asset URLs are still rewritten to archive copies, so CSS, images, and scripts load from the archive, not the live server. Playwright takes a full-height screenshot.

Then ffmpeg stitches it all into an MP4. Simple concat, crossfade, or scroll-pan.

## The Part That Breaks

Rendering archived pages is fine until you hit SPAs. Any SPA that registered a Service Worker - when it was archived, the SW got archived too. When you replay it, the SW registers and starts intercepting requests, but now it points to the live origin, not the archive. The page breaks.

wayback-video blocks Service Workers before the page loads. It also waits for JS to finish rendering after page load (default 2.5s - Playwright's network-idle events don't work reliably against Wayback's proxy, so a fixed wait is the practical fallback). If the resulting page has fewer than 200 characters of body text, it's probably a spinner or an archived 404 - so the tool skips it and tries the next candidate (the next archived snapshot from the same time bucket).

Old static sites render fine. JS-heavy SPAs are hit or miss - depends on what Wayback actually captured.

## Deduplication

Not every year looks different. Some sites go through five years without a major redesign. Without dedup, you get the same frame repeated five times in a row.

Two passes run automatically in `--scroll` mode:

1. Exact dedup by SHA-256 hash. Byte-identical renders are dropped.
2. Average hash (aHash) comparison. Consecutive frames with low visual distance get merged into one clip with a combined label like `2011-2014`.

The threshold is configurable (`--similarity-threshold`, default Hamming distance of 10, max 64). Raise it to merge more aggressively, lower it to keep subtle changes.

## Modes

| Mode | When to reach for it |
|------|---------------------|
| Default (CDX) | Fixed-viewport screenshots per period |
| `--scroll` | Full-page height with pan animation (recommended) |
| `--hybrid year` | Wayback pre-captured PNGs first, Playwright as fallback |
| `--wayback-screenshot year` | Wayback PNGs only, no browser, fast |
| `--at-interval year` | One capture per year at a fixed date, via Availability API |
| `--image` | Logo or image file evolution, no page render |

`--scroll` is what I use by default. Full page height means you actually see the layout, not just the above-the-fold hero. Scroll speed is auto-calculated from page height, so nothing crawls or blurs past.

## Get Started

```bash
git clone https://github.com/tegos/wayback-video.git
cd wayback-video
pip install -e .
playwright install chromium

# Ubuntu/Debian
sudo apt install ffmpeg

# macOS
brew install ffmpeg
```

Heads up: `playwright install chromium` downloads ~130MB of Chromium - required even if you already have Chrome installed.

Then pick a site you're curious about:

```bash
# Laravel's homepage through the years
wayback-video https://laravel.com --scroll --interval year --crossfade 0.4

# Wikipedia, month by month, over its first decade
wayback-video https://wikipedia.org --scroll --interval month --from 2001 --to 2010

# Fast mode: Wayback's pre-captured PNGs only, no browser needed
wayback-video https://mozilla.org --wayback-screenshot year --scroll
```

Requires Python 3.10+ and ffmpeg installed separately.

## TL;DR

- wayback-video turns any site's Wayback Machine history into an MP4
- Archived pages render in `if_` mode with Service Workers blocked so SPAs and assets load cleanly from the archive
- aHash dedup merges visually identical consecutive years into one labeled clip
- `--scroll` is what I use by default: full-page height, smooth pan animation

👉 [github.com/tegos/wayback-video](https://github.com/tegos/wayback-video)

## Author's Note

Thanks for sticking around!
Find me on [dev.to](https://dev.to/tegos), [linkedin](https://www.linkedin.com/in/ivan-mykhavko/), or you can check out my work on [github](https://github.com/tegos).

**Laravel, after the happy path.**
