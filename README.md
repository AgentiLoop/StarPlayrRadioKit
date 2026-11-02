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

## Examples

Every helper that waits (`PostSync`, `TextSync`, `dataSync`, `GetPdtSync`, `Session`) runs its completion handler **before it returns**. The usual pattern is to capture the result in a local variable and read it after the call. Run all of these examples on a background queue.

### 1. Run kit calls off the main thread

```swift
import StarPlayrRadioKit

let radioQueue = DispatchQueue(label: "radio", qos: .userInitiated)

/// Turns any of the blocking kit calls into an async function.
func onRadioQueue<T>(_ work: @escaping () -> T) async -> T {
    await withCheckedContinuation { continuation in
        radioQueue.async { continuation.resume(returning: work()) }
    }
}
```

### 2. Region setup and automatic sign-in

`preflightConfig` restores `userX` from `UserDefaults` only when there's a saved gupid **and** a saved channel lineup.

```swift
func bootstrap(region: String) -> Bool {
    preflightConfig(location: region)          // "US" or "CA"
    let restored = !userX.gupid.isEmpty && userX.channels.count > 1
    print(restored ? "Welcome back, \(userX.email)" : "Sign-in needed")
    return restored
}
```

`Config()` downloads the CDN hosts without waiting. `hls_sources` may be empty for a moment after `preflightConfig` returns. If an earlier launch stored them, the cached copy is used when the download fails.

```swift
func waitForHLSSources(timeout: TimeInterval = 5) -> Bool {
    let deadline = Date().addingTimeInterval(timeout)
    while hls_sources["Live_Secondary_HLS"] == nil, Date() < deadline {
        Thread.sleep(forTimeInterval: 0.1)
    }
    return hls_sources["Live_Secondary_HLS"] != nil
}
```

### 3. Signing in with error handling

```swift
enum LoginResult {
    case success(gupid: String)
    case failure(String)
}

func signIn(user: String, pass: String) -> LoginResult {
    let login = LoginX(username: user, pass: pass)
    var outcome = LoginResult.failure("No response from SiriusXM.")

    PostSync(request: login.request, endpoint: login.endpoint, method: login.method) { result in
        guard let result else { return }

        // HTTP 403 → "Too many incorrect logins …" (IP blocked for 24 hours)
        let login = processLogin(username: user, pass: pass, result: result)

        if login.success {
            storeCookiesX()                    // keep the session cookies for the next launch
            outcome = .success(gupid: login.data)
        } else {
            outcome = .failure(login.message)
        }
    }
    return outcome
}

// From SwiftUI or UIKit:
Task {
    switch await onRadioQueue({ signIn(user: email, pass: password) }) {
    case .success(let gupid): print("Signed in, gupid \(gupid)")
    case .failure(let message): print("Login failed: \(message)")
    }
}
```

### 4. Loading the channel lineup

```swift
struct Channel {
    let id: String          // SiriusXM channel ID
    let number: Int         // 2
    let name: String
    let logo: URL?          // 720 px logo for dark backgrounds
    let category: String
}

func loadChannels() -> (channels: [Channel], categories: [String])? {
    let api = Channels()
    var lineup: (channels: [Channel], categories: [String])?

    PostSync(request: api.request, endpoint: api.endpoint, method: api.method) { result in
        guard let result else { return }
        let processed = processChannels(result: result)
        guard processed.success else { print(processed.message); return }

        storeCookiesX()

        let channels = processed.data.values.compactMap { value -> Channel? in
            guard
                let ch = value as? [String: Any],
                let id = ch["channelId"] as? String,
                let numberText = ch["channelNumber"] as? String,
                let number = Int(numberText),
                let name = ch["name"] as? String,
                let category = ch["category"] as? String
            else { return nil }

            return Channel(id: id, number: number, name: name,
                           logo: URL(string: ch["mediumImage"] as? String ?? ""),
                           category: category)
        }
        .sorted { $0.number < $1.number }

        lineup = (channels, processed.categories)
    }
    return lineup
}
```

Group the lineup by category, using the category order the kit returns:

```swift
if let lineup = loadChannels() {
    let byCategory = Dictionary(grouping: lineup.channels, by: \.category)
    for category in lineup.categories {
        let names = byCategory[category, default: []].map { "\($0.number) \($0.name)" }
        print("\(category): \(names.joined(separator: ", "))")
    }
}
```

### 5. Converting between channel number and channel ID

`userX.channels` is keyed by channel **number**. `userX.ids` is keyed by channel **ID**.

```swift
func channelId(forNumber number: String) -> String? {
    (userX.channels[number] as? [String: Any])?["channelId"] as? String
}

func channelNumber(forId id: String) -> String? {
    (userX.ids[id] as? [String: Any])?["channelNumber"] as? String
}

let id = channelId(forNumber: "2")             // the channel ID for channel 2
let number = id.flatMap(channelNumber(forId:)) // "2"
```

### 6. Tuning in and keeping the token fresh

```swift
final class Tuner {
    private var currentId = ""
    private var tokenRefreshed = Date.distantPast
    private let tokenLifetime: TimeInterval = 480   // SXMAKTOKEN lasts about 8 minutes

    /// Call this before each playlist request.
    func prepare(channelId: String) {
        userX.channel = channelId

        if channelId != currentId {
            // New channel: get a new token *and* this channel's key, key URL and consumer.
            Session(channelid: channelId, updateToken: true, updateUser: true)
            currentId = channelId
            tokenRefreshed = Date()
        } else if Date().timeIntervalSince(tokenRefreshed) >= tokenLifetime {
            // Same channel, old token: refresh only the token.
            Session(channelid: channelId, updateToken: true, updateUser: false)
            tokenRefreshed = Date()
        }
    }
}
```

`Session` returns the channel lineup ID from the server, or `"350"` if the response doesn't have one:

```swift
let lineupId = Session(channelid: channelId, updateToken: true, updateUser: true)
print("lineup \(lineupId), key loaded: \(!userX.key.isEmpty)")
```

### 7. Fetching a playlist and rewriting it for a local server

The kit returns SiriusXM's playlist without changes. To play it through your own server, point the key and segment URLs at your routes. This is what StarPlayrX does:

```swift
func localPlaylist(channelId: String) -> String? {
    var playlist: String?

    TextSync(endpoint: Playlist(channelid: channelId)) { text in
        guard let text, !text.hasPrefix("error") else { return }   // TextSync reports failures as "error1"/"error2"

        playlist = text
            .replacingOccurrences(of: "key/1", with: "/api/v3/key")                  // #EXT-X-KEY URI
            .replacingOccurrences(of: channelId, with: "/api/v3/aac/" + channelId)   // segment URIs
    }
    return playlist
}
```

To list the segment names in a playlist:

```swift
func segmentNames(in playlist: String) -> [String] {
    playlist
        .split(whereSeparator: \.isNewline)
        .map(String.init)
        .filter { !$0.hasPrefix("#") && $0.hasSuffix(".aac") }
}
```

### 8. Downloading AAC segments, with a retry and CDN failover

```swift
func downloadSegment(_ name: String, channelId: String) -> Data? {
    func attempt() -> Data? {
        var segment: Data?
        dataSync(endpoint: AudioX(data: name, channelId: channelId), method: audioFormat) { segment = $0 }
        return segment
    }

    if let data = attempt() ?? attempt() { return data }   // one quick retry

    // Still failing: switch CDN (secondary ↔ primary) and try once more.
    usePrime.toggle()
    return attempt()
}
```

Each segment is AES-128 encrypted. `AVPlayer` decrypts it with the key from the playlist's `#EXT-X-KEY`, which comes from `userX.key`:

```swift
let aesKey: Data? = Data(base64Encoded: userX.key)   // 16 bytes
```

### 9. Serving the stream with SwifterLite

This is a minimal version of the StarPlayrX routes. It connects `AVPlayer` → localhost → this kit.

```swift
import SwifterLite
import StarPlayrRadioKit

let tuner = Tuner()

func makeServer() -> HttpServer {
    let server = HttpServer()

    // http://127.0.0.1:9999/api/v3/m3u/2.m3u8  (channel number 2)
    server.get["/api/v3/m3u/:channel"] = { request in
        guard
            let number = request.params[":channel"]?.split(separator: ".").first.map(String.init),
            let id = channelId(forNumber: number)
        else { return HttpResponse.notFound(.none) }

        tuner.prepare(channelId: id)
        guard let m3u8 = localPlaylist(channelId: id) else { return HttpResponse.notFound(.none) }
        return HttpResponse.ok(.data(Data(m3u8.utf8), contentType: "application/x-mpegURL"))
    }

    server.get["/api/v3/key"] = { _ in
        guard let key = Data(base64Encoded: userX.key) else { return HttpResponse.notFound(.none) }
        return HttpResponse.ok(.data(key, contentType: "application/octet-stream"))
    }

    server.get["/api/v3/aac/:aac"] = { request in
        guard
            let name = request.params[":aac"],
            let audio = downloadSegment(name, channelId: userX.channel)
        else { return HttpResponse.notFound(.none) }
        return HttpResponse.ok(.data(audio, contentType: audioFormat))
    }

    return server
}

let server = makeServer()
try server.start(9999)

let player = AVPlayer(url: URL(string: "http://127.0.0.1:9999/api/v3/m3u/2.m3u8")!)
player.play()
```

### 10. Now playing for one channel (with album art)

`nowPlayingLiveAsync` doesn't block. `processNPL` stores the album art in an internal cache, and `processPDT` reads from that cache later.

```swift
func refreshAlbumArt(channelId: String, done: @escaping () -> Void = {}) {
    nowPlayingLiveAsync(endpoint: nowPlayingLive(channelid: channelId)) { live in
        if let live { processNPL(data: live) }   // talk channels often return nil
        done()
    }
}
```

### 11. Now playing for every channel

```swift
struct NowPlaying {
    let artist: String
    let song: String
    let image: URL?
}

func nowPlayingAll() -> [String: NowPlaying] {      // keyed by channel number
    var guide = [String: NowPlaying]()

    GetPdtSync(endpoint: PDTendpoint(), method: "PDT") { pdt in
        guard let pdt else { return }
        for (number, value) in processPDT(data: pdt) {
            guard let info = value as? [String: Any] else { continue }
            guide[number] = NowPlaying(
                artist: info["artist"] as? String ?? "",
                song:   info["song"]   as? String ?? "",
                image:  URL(string: info["image"] as? String ?? "")
            )
        }
    }
    return guide
}
```

To poll the guide, and refresh the album art for the channel that's playing first:

```swift
let guideTimer = DispatchSource.makeTimerSource(queue: radioQueue)
guideTimer.schedule(deadline: .now(), repeating: 10)
guideTimer.setEventHandler {
    let finished = DispatchSemaphore(value: 0)
    refreshAlbumArt(channelId: userX.channel) { finished.signal() }
    _ = finished.wait(timeout: .now() + 5)

    let guide = nowPlayingAll()
    if let number = channelNumber(forId: userX.channel), let now = guide[number] {
        DispatchQueue.main.async { print("♪ \(now.artist) – \(now.song)") }
    }
}
guideTimer.resume()
```

### 12. Fading the volume and switching channels smoothly

`fadeVolume` sets `volume = from` straight away and then changes it every 0.1 s on a `Timer`. Call it on the **main thread**, which has a run loop.

```swift
import AVFoundation

// Fade in over 2 seconds.
player.play()
player.fadeVolume(from: 0, to: 1, duration: 2)

// Fade out, then pause.
player.fadeVolume(from: player.volume, to: 0, duration: 1.5) {
    player.pause()
}

// Fade out, switch to another channel, then fade back in.
func switchChannel(to number: String) {
    player.fadeVolume(from: player.volume, to: 0, duration: 0.5) {
        let url = URL(string: "http://127.0.0.1:9999/api/v3/m3u/\(number).m3u8")!
        player.replaceCurrentItem(with: AVPlayerItem(url: url))
        player.play()
        player.fadeVolume(from: 0, to: 1, duration: 1)
    }
}

// Keep the Timer if you might need to cancel the fade.
var fade: Timer? = player.fadeVolume(from: 1, to: 0.2, duration: 3)   // "duck"
fade?.invalidate()
```

### 13. Mac app hooks: real durations, playlist and segment observers

```swift
// Keep the real #EXTINF durations so AVPlayerItem.currentDate() matches EXT-X-PROGRAM-DATE-TIME.
// Your playlist route checks this flag (see StarPlayrXMac's playlistRoute).
realSegmentDurations = true

// Called by your playlist route with each playlist it serves.
livePlaylistObserver = { playlist in
    let pdt = playlist
        .split(whereSeparator: \.isNewline)
        .first { $0.hasPrefix("#EXT-X-PROGRAM-DATE-TIME:") }
    print("Playlist starts at \(pdt ?? "?")")
}

// Called by your segment route with each (still encrypted) segment and its AES key.
// StarPlayrX for Mac decrypts and analyses these to draw its spectrum bars before playback.
liveSegmentObserver = { name, data, key in
    guard !data.isEmpty, let key else {
        print("Segment \(name) failed to download")
        return
    }
    print("Segment \(name): \(data.count) bytes, key \(key.count) bytes")
}
```

In your routes, call the observers yourself:

```swift
livePlaylistObserver?(playlist)
liveSegmentObserver?(name, audio, Data(base64Encoded: userX.key))
```

### 14. Switching region at runtime

```swift
func switchRegion(to region: String) {   // "US" or "CA"
    preflightConfig(location: region)    // updates playerDomain, root, appRegion and reloads hls_sources
    _ = loadChannels()                   // the lineup is different in each region
}
```

### 15. Signing out (clearing saved state)

The kit doesn't have a sign-out function. Clear the `UserDefaults` keys it uses, and reset `userX`:

```swift
func signOut() {
    let keys = ["user", "pass", "email", "gupid", "loggedin", "token", "key",
                "keyurl", "consumer", "channels", "ids", "channel", "siriusxm"]
    keys.forEach(UserDefaults.standard.removeObject(forKey:))

    HTTPCookieStorage.shared.cookies?.forEach(HTTPCookieStorage.shared.deleteCookie)

    userX = (email: "", pass: "", channels: [:], ids: [:], channel: "", token: "",
             loggedin: false, gupid: "", consumer: "", key: "", keyurl: "")
}
```

### 16. Utility: `sha256`

```swift
let id = sha256("Daft Punk" + "One More Time")   // 64-character hex string, the album-art cache key
```

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
