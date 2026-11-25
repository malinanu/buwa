# Buwa — Title Reveal & Muhurath Ceremony

Invitation microsite for **Buwa** by Ayanna Yata Films.

- **Date:** Sunday, 29 November 2026, 5:00 PM (Sri Lanka time)
- **Venue:** The Grand Maitland, Colombo 07

## Structure

Static site, no build step. Mobile-first, about 120 KB on first load.

| File | Purpose |
|---|---|
| `index.html` | The whole page (inline CSS and JS) |
| `logo.webp` | Ayanna Yata Films logo |
| `maitland.webp`, `maitland-l.webp` | Hero image (phone / large screens) |
| `colombo.webp` | Venue section image (lazy-loaded) |
| `protest.svg` | Line-art illustration above the verses (lazy-loaded) |
| `music.mp3` | Background music, loaded only after the guest taps "Open invitation" |

## Personalised guest links

Append the guest's name after `#`, words separated by hyphens:

```
https://your-domain/#Malin-Jayaweera
```

The name appears on the opening screen, in the letter and on the pass. Without it, the page reads "Distinguished Guest".

## Deploy

Serve the folder as-is from any static host (Cloudflare Pages, GitHub Pages, Netlify, or Nginx). The `music.mp3` file benefits from long cache headers.
