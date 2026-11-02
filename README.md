![Platform](https://img.shields.io/badge/Platform-iOS%2015%2B%20%7C%20macOS%2012%2B-4BC51D.svg?style=flat)
![Swift](https://img.shields.io/badge/Swift-5.6%2B-4BC51D.svg?style=flat)
![SPM](https://img.shields.io/badge/SPM-compatible-4BC51D.svg?style=flat)

# StarPlayrRadioKit

StarPlayrRadioKit is the SiriusXM layer behind **StarPlayrX** on iOS and macOS. It handles:

- **Login and session**: signs in, keeps the cookies, and refreshes the stream token.
- **Channel lineup**: fetches it, picks a logo for each channel and sorts channels into categories.
- **Live HLS**: builds playlist, AES key and AAC segment URLs for the primary or secondary CDN, and downloads them.
- **Now playing**: gets the artist, song and album art for every channel.
- **Audio helpers**: fades `AVPlayer` volume in and out.

The kit doesn't include an HTTP server. Each StarPlayrX app runs its own local [SwifterLite](https://github.com/AgentiLoop/SwifterLite) server (in its `Routes/` folder), and those routes call this kit. See [How StarPlayrX uses it](#how-starplayrx-uses-it).

## Requirements

- iOS 15+ or macOS 12+ (declared in `Package.swift`). tvOS and watchOS may work, but they aren't declared or tested.
- Swift 5.6+.
- Apple frameworks only: Foundation, Network, AVFoundation and CommonCrypto. **No third-party dependencies.**
- Linux is not supported.

## Installation

```swift
dependencies: [
    .package(url: "https://github.com/AgentiLoop/StarPlayrRadioKit", branch: "mustang")
],
targets: [
    .target(
        name: "YourTarget",
        dependencies: [.product(name: "StarPlayrRadioKit", package: "StarPlayrRadioKit")]
    )
]
```

## Things to know first

- **Most calls block.** `PostSync`, `Session`, `TextSync`, `dataSync` and `GetPdtSync` start a `URLSession` task and wait on a semaphore until it finishes. They're meant to be called from server request threads, **not the main thread**. Only `GetAsync` (internal) and `nowPlayingLiveAsync` don't wait.
- **State is global.** The signed-in user, tokens, keys and channel lineup live in the module-level `userX` tuple. Region and endpoints live in globals such as `playerDomain`, `root`, `appRegion`, `hls_sources` and `usePrime`. There's no client object to create.
- **It saves data to `UserDefaults.standard`**, including the username and **password** (keys `user` and `pass`), the gupid, token, key, key URL, consumer, channel lineup, HLS sources and the SiriusXM cookies (key `siriusxm`). `preflightConfig()` reads these back to sign in again automatically.
- Requests use the Safari user agent in `userAgent`.

## Quick start

This is the full sequence from launch to playable audio. It runs off the main thread.

```swift
import StarPlayrRadioKit

// 1. Pick a region, load the CDN config, start the network monitor and restore the saved session.
preflightConfig(location: "US")            // or "CA"

// 2. Sign in.
let login = LoginX(username: user, pass: pass)
PostSync(request: login.request, endpoint: login.endpoint, method: login.method) { result in
    guard let result else { return }
    let outcome = processLogin(username: user, pass: pass, result: result)
    if outcome.success { storeCookiesX() }  // outcome.data is the gupid
}

// 3. Get the channel lineup. This fills userX.channels (by channel number) and userX.ids (by channel ID).
let api = Channels()
PostSync(request: api.request, endpoint: api.endpoint, method: api.method) { result in
    guard let result else { return }
    let lineup = processChannels(result: result)   // .data, .categories, .success, .message
}

// 4. Tune in. This refreshes the stream token, AES key, key URL and consumer for the channel.
userX.channel = channelId
Session(channelid: channelId, updateToken: true, updateUser: true)

// 5. Fetch the live playlist.
TextSync(endpoint: Playlist(channelid: channelId)) { m3u8 in /* rewrite and serve */ }

// 6. Fetch an AAC segment named in the playlist. Each segment is still encrypted.
dataSync(endpoint: AudioX(data: segmentName, channelId: channelId), method: audioFormat) { aac in /* ... */ }

// The AES-128 key for the playlist's #EXT-X-KEY:
let key = Data(base64Encoded: userX.key)
```

The stream token expires after about 8 minutes. StarPlayrX calls `Session(channelid:updateToken:true, updateUser:false)` again every 480 seconds.

## API reference

### Setup and region (`Setup/`)

| Symbol | What it does |
|---|---|
| `preflightConfig(location: String = "US")` | Sets `playerDomain`, `root` and `appRegion` for `"US"` or `"CA"`. It then runs `Config()`, starts the network monitor, restores `userX` from `UserDefaults` if a saved login exists, and restores the cookies. |
| `Config()` | Downloads the SiriusXM configuration and fills `hls_sources` (`Live_Primary_HLS`, `Live_Secondary_HLS`, …). If the download fails, it uses the cached copy. |
| `Session(channelid:updateToken:updateUser:) -> String` | POSTs `/resume` (this call blocks). `updateToken` takes the `SXMAKTOKEN` cookie and stores it in `userX.token`. `updateUser` stores `userX.key`, `keyurl` and `consumer`. Returns the channel lineup ID (default `"350"`). |
| `storeCookiesX()` | Saves the SiriusXM cookies to `UserDefaults`. `preflightConfig` restores them. |

### Login (`Login/`)

| Symbol | What it does |
|---|---|
| `LoginX(username:pass:) -> (request, endpoint, method)` | Builds the authentication request. Send it with `PostSync`. |
| `processLogin(username:pass:result:) -> (success, message, data)` | Reads the login response. It sets `userX.email` and `userX.gupid`, and saves the credentials for automatic sign-in. Treats HTTP 403 as "IP blocked for 24 hours". |

### Channels (`Channels/`)

| Symbol | What it does |
|---|---|
| `Channels() -> (request, endpoint, method)` | Builds the channel listing request (`/rest/v4/experience/modules/get`). |
| `processChannels(result:) -> (success, message, data, categories)` | Builds the lineup, keyed by channel number. Each entry has `channelId`, `channelNumber`, `name`, `mediumImage` (the 720 px logo for dark backgrounds), `category` and `preset`. Renames some categories, for example "Latino Music" becomes "Latin Music". Fills and saves `userX.channels` and `userX.ids`. |
| `DiscoverChannelList` | `Codable` model for the discover-channel-list (PDT) response. |

### Streaming (`Audio/`)

| Symbol | What it does |
|---|---|
| `Playlist(channelid:) -> String` | URL of the live `.m3u8` for the channel, built from `userX.keyurl`, the selected HLS host, the bitrate, and `consumer` and `token`. |
| `AudioX(data:channelId:) -> String` | URL of one AAC segment, `/AAC_Data/<id>/HLS_<id>_<bitrate>_v3/<segment>`. |
| `audioFormat` | `"audio/aac"` |
| `usePrime` | `true` uses `Live_Primary_HLS`, `false` (the default) uses `Live_Secondary_HLS`. |
| `AVPlayer.fadeVolume(from:to:duration:completion:)` | Fades the volume with a 0.1-second timer and returns the `Timer`. |

The bitrate is **64k on cellular** and **256k otherwise**. The internal `Network` monitor (`NWPathMonitor`) detects cellular.

### Now playing (`Turf/`)

| Symbol | What it does |
|---|---|
| `nowPlayingLive(channelid:) -> String` | Endpoint for the channel's now-playing-live data. |
| `nowPlayingLiveAsync(endpoint:LiveHandler:)` | Fetches and decodes `NowPlayingLiveStruct` without blocking. |
| `processNPL(data:)` | Caches album-art URLs for the channel's cuts, keyed by `sha256(artist + song)`. |
| `PDTendpoint() -> String` | Endpoint for the discover-channel-list with PDT. It returns now-playing for every channel at once. |
| `GetPdtSync(endpoint:method:PdtHandler:)` | Fetches and decodes a `DiscoverChannelList` (this call blocks). |
| `processPDT(data:) -> [String: Any]` | Returns `["<channelNumber>": ["artist", "song", "image"]]` for every channel. It uses the album art cached by `processNPL`, and falls back to placeholder text when the guide is empty. |
| `sha256(_:) -> String?` | Hex SHA-256 of a string (CommonCrypto). |

### Networking helpers (`API/`)

All of them send the `userAgent` header and `Content-Type: application/json`.

| Symbol | Method | Timeout | Notes |
|---|---|---|---|
| `PostSync(request:endpoint:method:PostTupleHandler:)` | POST JSON | 60 s | Calls back with a `PostReturnTuple` `(message, success, data, response)`. |
| `TextSync(endpoint:TextHandler:)` | GET | 60 s | Calls back with the body as UTF-8 text. |
| `dataSync(endpoint:method:DataHandler:)` | GET | 15 s request, 20 s wait | Calls back with `nil` on HTTP 400 or higher. Always releases the waiting thread. |

### Hooks for the Mac app (`Setup/Global.swift`)

| Symbol | Purpose |
|---|---|
| `realSegmentDurations` | When `true`, the app's playlist route keeps the real segment durations so `AVPlayerItem.currentDate()` matches the stream's `EXT-X-PROGRAM-DATE-TIME`. |
| `livePlaylistObserver` | Called with each live playlist the server returns. |
| `liveSegmentObserver` | Called with each AAC segment (still encrypted) and its AES key. The Mac app uses it to draw its spectrum bars. |

## How StarPlayrX uses it

Both apps start a SwifterLite server with `startServer(port)` and point `AVPlayer` at it. The routes, found in `StarPlayrX/.../Routes/` and `StarPlayrXMac/Sources/Routes/`, connect HTTP requests to the kit:

| Route | Kit calls |
|---|---|
| `GET  /api/v3/ping` | `Session(...)` to refresh the token every 480 s |
| `GET  /api/v3/us`, `/api/v3/ca` | sets `playerDomain`, `root` and `appRegion` |
| `POST /api/v3/login` | `LoginX`, then `PostSync`, then `processLogin`, then `storeCookiesX` |
| `POST /api/v3/session` | `Session(channelid:updateToken:true, updateUser:true)` |
| `POST /api/v3/channels` | `Channels`, then `PostSync`, then `processChannels` |
| `GET  /api/v3/pdt` | `nowPlayingLive` and `processNPL`, then `PDTendpoint`, `GetPdtSync` and `processPDT` |
| `GET  /api/v3/key` | `Data(base64Encoded: userX.key)` |
| `GET  /api/v3/m3u/:channelid` | `Session`, then `Playlist`, then `TextSync`. The route rewrites key and segment URLs to `/api/v3/key` and `/api/v3/aac/…` |
| `GET  /api/v3/aac/:aac` | `AudioX`, then `dataSync` (retried once), then `liveSegmentObserver` |

## Package layout

```
Sources/StarPlayrRadioKit/
├── API/        PostSync, TextSync, dataSync, GetPdtSync, GetAsync, Network (NWPathMonitor)
├── Audio/      Playlist, AudioX, AVPlayer.fadeVolume
├── Channels/   Channels, processChannels, DiscoverChannelList models
├── Krpyto/     AES-128-CBC decrypt helper (internal; nothing in the kit calls it at the moment)
├── Login/      LoginX, processLogin
├── Setup/      Global state, preflightConfig, Config, Session, cookies
└── Turf/       Now-playing (NPL/PDT) models and processing, sha256, extensions
```

## Credits and license

MIT © 2022 Todd Bruss. See [LICENSE](LICENSE).
`AVPlayer.fadeVolume` is adapted from AVPlayerFade by Evandro Harrison Hoffmann.

---

**AgentiLoop:** [agentiloop.ai](https://agentiloop.ai/)

Copyright © 2026 AgentiLoop.ai, a Logos InkPen LLC company. All rights reserved.
