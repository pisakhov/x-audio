# Architecture

These are the decisions behind Listen. They are framework- and model-agnostic. The numbers are the ones we ship.

## 1. Script: markdown → speech

A deterministic pass runs before any audio is made. It has no model, so the same reply always becomes the same script.

- **Headings** become short spoken titles, followed by a longer pause.
- **Lists:** bullets become short sentences, and numbered steps become "First, … Second, …".
- **Emoji:** decorative ones are dropped. A status icon that leads a line is spoken (⚠️ "Heads up:", ❌ "Problem:", ▶️ "Next:"). ✅ is dropped.
- **Links and paths:** links read their label, bare URLs read their domain, and file paths read only the file name.
- **Code, diagrams, config and large tables are skipped out loud.** Each becomes one natural line, such as "There's a short Python snippet here, which I'll skip." The wording varies, so a reply with five code blocks doesn't sound like a loop.
- **Small tables** are read row by row. **Images** are mentioned by their alt text.
- **Suggested replies** become "Suggested replies. First: … Second: …".

Each block carries a kind (`heading`, `para`, `item`, `skip`, `replies`), and the kind sets the pause after it.

## 2. Voice: a live agent reading verbatim

- **Prompt:** each passage goes to a live speech-to-speech session framed as *"read this aloud, verbatim"*.
- **Check:** the session's own transcript is compared with the source. If the model ad-libbed, the passage is retried once.
- **Clean-up:** passages are trimmed, then joined with pauses sized to the seam (sentence < list item < block < heading). Everything is lifted to one shared speech level, measured on voiced audio only.
- **Output:** one compressed file (64 kbps AAC), plus a 64-bar loudness outline for the waveform.

## 3. Pacing: start in seconds, never stall

Live voices speak at about real time. With `N` sessions reading in parallel (we use 3), passage `k` is rendered while the audio before it plays. So passage `k` must be no longer than the audio queued ahead of it:

```
p0        ≤ 90 bytes                      (a clause is fine)
p1        ≤ 0.85 × 2·p0
p2        ≤ 0.85 × (2·p0 + p1)
pk (k≥3)  ≤ 0.85 × (p[k-3] + p[k-2] + p[k-1])
floor 60 bytes; cut at sentence ends; if no sentence fits, take one sentence
```

Passages grow as the sessions pull ahead, so a long reply needs only a handful of round trips. On a real reply this moved first audio from 20.7 s to 9.5 s, with 0 s of stalls.

## 4. Segments: one timeline

- **Segments:** as soon as the next passages *in order* finish, they are encoded as one segment. Each segment is sent with its length, a loudness outline (one level per 200 ms, on a fixed scale) and an updated estimate of the total length.
- **No jump at the switch:** every segment uses the same gain as the final file, and boundaries fall on passage seams. Played back to back, the segments match the saved file sample for sample, so the player switches to the saved file at a boundary without a jump.
- **Delivery:** segments are fetched whole (Safari won't stream media without byte ranges), and the next one is prefetched. Seeking is limited to what has been rendered so far.

## 5. Player

- **Placement:** at the end of the reply, the order is reply → player strip (once opened) → **Listen** · **Copy**.
- **Pill states:** `Listen` → progress ring "Preparing" → play/pause with "~m:ss" left while it renders → exact time left once saved.
- **Strip:** play, a seekable waveform (click, drag, arrows ±5 s, Page ±30 s, Home/End), the elapsed and total time, the voice and version, back 10 s, speed 1×–2× (remembered), read again, and close.
- **While streaming:** rendered audio shows as real bars, and the unrendered part as a dim track labelled "Reading ahead · NN%". If playback ever catches up, the strip says "Catching up…" and resumes by itself.
- **Smooth:** each state is built once and then patched in place, so rings and waveforms animate without restarting on every poll.
- **Everywhere:** a single shared audio element (one reply plays at a time), lock-screen controls via Media Session, and no truncation down to 320 px.

## 6. API and storage

| Call | Purpose |
| --- | --- |
| `GET  …/narrations` | status of every reply/version in a conversation |
| `POST …/narrations` | `{messageId, view, text, voice, regenerate?}` → ready, or queued |
| `GET  …/narrations/:message/:view/audio` | the saved file, with HTTP Range |
| `GET  …/narrations/:message/:view/segments/:stream/:index` | one finished segment of the current render |

- **Status:** `ready {voice, durationMs, peaks}`, or `queued|rendering|failed {done, total, stream, estimateMs, segments[]}`. The `stream` id names one render, so stale segments are never mixed with a new one.
- **Cache key:** `hash(text, voice, view)`. A refresh reuses the file, and edited text renders fresh.
- **Polling:** the client polls fast while rendering, faster while playing ahead of the render, and slowly in a hidden tab.
- **Isolation:** rendering runs off the hot path, at lower priority than live calls, so it never blocks a conversation.
- **Caps:** text 56 KiB, audio 48 MiB, store 1 GiB, at most 400 entries per conversation (oldest evicted first).
