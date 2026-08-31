# Site ads — cross-project reference

How Chatbun CMS ads flow through every frontend app in this monorepo.

## Pipeline (all apps)

```
Admin CMS  →  GET {ADMIN_API}/site/ads
ChatbunApi →  GET /api/ads
Each app   →  GET /api/ads  (Next proxy, SITE_API_KEY)
           →  fetchAdsShared()  (10s in-memory cache)
           →  <Advertisement /> | reel helpers | TV popup
```

**Separate systems (not site ads):**
- **Room ads** — `/api/rooms/:id/ads` (Reel, insta, Messenger, Feed)
- **User promoted reels** — `allow_reel_ads` upload metadata (Reel, insta, Feed)

---

## Reference implementations

| App | Maturity | Notes |
|-----|----------|-------|
| **Feed** | Full | 9 page slots, reel interleave, TV gate with real inventory, docs |
| **vTube** | Full + player | Watch-page overlays (`VideoPlayer`), shorts interleave, home grid; TV gate now real inventory |
| **Reel** | Core | `/api/ads`, `Advertisement`, reel interleave, TV gate + catalog banner |
| **insta** | Core | Same as Reel |
| **Messenger** | Minimal | `/api/ads`, rooms list 728×90 banner |
| **omegle** | Minimal | `/api/ads`, inbox sidebar 300×250 |
| **ChatbunApi** | Backend only | `src/routes/ads.ts` |

---

## Shared files (copy from Feed)

| File | Role |
|------|------|
| `components/advertisement/Advertisement.tsx` | Universal renderer + `fetchAdsShared` |
| `components/advertisement/advertisement-component.md` | Props / types |
| `lib/reelAdUtils.ts` | Filter + interleave `reel_ad` / `reel_banner` |
| `components/ReelAdItem.tsx` | Snap slot wrapper |
| `lib/tv/siteAds.ts` | TV popup creative picker |
| `lib/tv/adUnlock.ts` | Unlock state + `lastAdId` rotation |
| `components/tv/SeriesAdUnlockGate.tsx` | Series watch gate |
| `app/api/ads/route.ts` | Proxy to ChatbunApi |

---

## Ad types (`Advertisement.tsx`)

| Type | Rendered |
|------|----------|
| `simple_banner` | Sized image; optional link |
| `reel_ad` / `video_player_ad` | Video/image; full-bleed with `videoOverlayFullSize` |
| `video_banner` / `reel_banner` | Banner-sized video/image |
| `google_adsense` | Script tag |
| `adserver` | Iframe / Revive HTML |
| `link_text` | TV popup only (not in `<Advertisement>`) |

---

## Slot matrix by app

### Feed
| Slot | Filter | File |
|------|--------|------|
| Home top | `position="top"` | `app/page.tsx` |
| Feed grid top / inline | `grid_top`, `grid_inline` | `PostContent.tsx` |
| Sidebar | `300x250` | `SuggestedBox.tsx` |
| Profile | `channel_top` | `[username]/page.tsx` |
| Video watch | `728x90` | `VideoDetailView.tsx` |
| TV catalog | `top` | `TvCatalog.tsx` |
| Reels | interleave | `ReelScreen` + `ReelAdItem` |
| TV unlock | random picker | `SeriesAdUnlockGate` |

### vTube
| Slot | Filter | File |
|------|--------|------|
| Home grid | `grid_top`, `grid_inline` | `video-grid.tsx` |
| Home top | `top` | `app/page.tsx` |
| Channel / likes / playlist | `channel_top`, `likes_top`, `playlist_top` | respective pages |
| Watch under player | `728x90` | `WatchPageClient.tsx` |
| Watch sidebar | `adserver`, `336x280` | `WatchPageClient.tsx` |
| Shorts | interleave | `shorts/page.tsx` + `ReelAdItem` |
| Player overlays | `video_player_ad`, `video_banner` | `VideoPlayer.tsx` |
| TV catalog | `top` | `TvCatalog.tsx` |
| TV unlock | random picker | `SeriesAdUnlockGate` |

### Reel / insta
| Slot | Filter | File |
|------|--------|------|
| Reels feed | interleave | `ReelScreen.tsx` |
| TV catalog | `top` | `TvCatalog.tsx` |
| TV unlock | random picker | `SeriesAdUnlockGate` |

### Messenger
| Slot | Filter | File |
|------|--------|------|
| Rooms list | `728x90` | `app/rooms/page.tsx` |

### omegle
| Slot | Filter | File |
|------|--------|------|
| Inbox sidebar | `300x250` | `ConversationList.tsx` |

---

## CMS position guide

`position` is a **strict string match**. Ads with `position: null` never fill a `position="top"` slot.

| CMS `position` | Typical slot |
|----------------|--------------|
| `top` | Home / TV catalog header |
| `grid_top` | Above feed grid |
| `grid_inline` | Every N posts in feed |
| `channel_top` | Profile / channel page |
| `inline` | Sidebar (when slot omits position filter) |

---

## Checks

```bash
npx tsx lib/tv/siteAds.check.ts
npx tsx lib/tv/adUnlock.check.ts
npx tsx lib/reelAdUtils.check.ts
```

(Reel/insta: run from `src/` if files live under `src/lib/`.)

---

## Gaps (future work)

- Video player overlays in Feed/Reel/insta (vTube has these)
- `is_active` / date filtering inside `<Advertisement>` (helpers enforce for TV/reels only)
- Impression/click analytics
- Server-enforced TV unlock (currently client `localStorage`)
- `likes_top` / `playlist_top` in Feed-family apps

See per-app `docs/ads.md` (Feed) or `docs/tv-ad-unlock.md` for unlock flow details.
