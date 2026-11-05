# iPhone 4 → TV, natively (iOS 6 field notes)

What it takes to make a jailbroken **iPhone 4 on iOS 6** drive a TV as a real TV box through
Apple's 30-pin AV adapters: a native 10-foot interface on the TV, the phone as a remote, and
modern video streams. These are the findings behind [iBox](https://github.com/sachomatic/iBox),
each one checked on the device. Code references point to that repository.

## The TV is a second screen, not a mirror

The iPhone 4 can't mirror its screen (that came with the 4S). Instead, the app draws its own
interface on the external `UIScreen`, while the phone's screen becomes a remote (trackpad and
buttons).

```objc
UIScreen *tv = [UIScreen screens].lastObject;          // when count > 1
tv.currentMode = /* largest mode: 1280×720 with the Digital AV adapter */;
UIWindow *w = [[UIWindow alloc] initWithFrame:tv.bounds];
w.screen = tv;
```

- **Overscan.** `overscanCompensation = Scale` shrinks everything and left a black border on the
  HDMI screen we tested. `InsetApplicationFrame` keeps 1280×720 but makes `applicationFrame` an
  inset rectangle ({64,36} 1152×648). A root view controller's view then gets laid out inside it
  (a border on the left and top) and is re-added on top when the window shows (a black box over
  everything). Put the interface view directly on the window, give the root view a clear
  background, and pick Scale only for SD/analog outputs.
- **Pixel aspect ratio.** Composite/component modes report non-square pixel sizes; correct for
  them before laying out.
- **Sound.** Through the Digital AV adapter the route is `HDMIOutput` at a fixed level: changing
  the system volume has no audible effect, so volume belongs to the TV's remote.
- **Power.** The adapter has a 30-pin charging port; without it the phone runs on battery, so
  show the battery level on the TV.

## Web video gets its own TV window

When a `UIWebView` plays inline video while a TV is attached, iOS 6 shows that video in
**`MPTVOutWindow`**, a window it creates itself on the TV screen, above the app's window. The
app's own views (OSD, menus, captions) are still there but covered. Seeking reveals them briefly,
because the video restarts.

Fix: draw anything that must appear over web video in a transparent window of higher level on
the TV screen (iBox uses `UIWindowLevelStatusBar + 50`; see `-[IBTV chrome]`). A screenshot made
with `renderInContext:` doesn't show this; listing `[UIApplication sharedApplication].windows`
does. `AVPlayer` videos played by the app itself are not affected.

## One hardware video decoder

The A4 decodes one video at a time. A web-view `<video>` that still holds a source makes every
`AVPlayerItem` fail with "Cannot Complete Action". Empty it first
(`removeAttribute('src'); load()`). On a playing inline video, `pause()` blocks the main thread
for about a second and `stopVideo()` about 0.8 s (they go through mediaserverd). Destroying the
page (`loadHTMLString:@""`) is the only reliable way to make sure nothing keeps playing.

Decoder limits: H.264 Baseline/Main up to level 3.1 (720p30), AAC/MP3. High profile sometimes
works and sometimes fails (-12911, -12642). `readyForDisplay` on the `AVPlayerLayer` tells whether
frames are really being decoded (sound without picture otherwise looks like normal playback).
`AVPlayerItemVideoOutput` wedged the media stack on iOS 6: don't use it.

## Modern HLS on iOS 6

iOS 6's HLS client predates most of today's playlists. A local proxy that rewrites playlists
(never the media) makes most channels play:

- **`#EXT-X-DISCONTINUITY-SEQUENCE` makes live streams jump back.** iOS 6 reads it as a plain
  `#EXT-X-DISCONTINUITY`, so every reload of a live playlist resets its timeline. The seekable
  range jumps by ±70 s, segments are fetched twice or skipped, and the picture "rolls back every
  few seconds" (AWS MediaTailor channels, for example). Dropping the line fixes it.
- Relabel `#EXT-X-VERSION` above 4 when no newer feature is used; drop WebVTT/closed-caption
  renditions, I-frame playlists, low-latency and `DATERANGE` lines; keep one audio rendition.
- Raise `#EXT-X-TARGETDURATION` when segments are longer than announced.
- Drop variants the decoder can't take (above 720p, above 30 fps at 720p, HEVC/VP9/AV1/AC-3, H.264
  above Main 3.1), and trim very long live windows (hours of segments) to the last few.
- iOS 6's TLS fails on many modern servers: fetch HTTPS through the proxy (iBox uses BearSSL with
  Mozilla's CA list). Google's front ends still accept iOS 6's TLS.

## Keeping a box running

- A suspended app loses its listening sockets: rebind them on `didBecomeActive` and on a timer.
- A LaunchDaemon (running as `mobile`) uses `SBSLaunchApplicationWithIdentifier` to relaunch the
  app when the home screen shows, and restarts it when a main-thread ping stops answering. It
  skips that while the phone is locked (`com.apple.springboard.lockstate`): SpringBoard refuses
  launches then, and a suspended app doesn't answer anyway.

## Building for armv7 / iOS 6 in 2026

The Theos Linux toolchain (clang + ld64) with the iOS 10.3 SDK works:
- lld can't link armv7 Mach-O, so link with ld64;
- link without `-fobjc-arc` (iOS 6 has the ARC runtime, and libarclite isn't shipped);
- add `-lgcc_s.1` for SjLj unwinding and `__divsi3`;
- borrow libobjc's stub from the 9.3 SDK for `objc_msgSend_stret`.

See iBox's `build.sh`.
