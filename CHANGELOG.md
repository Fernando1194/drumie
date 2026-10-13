# Changelog

Every version: what changed and why, with the measurement behind it when there is one.
Design notes and all measurements in detail (Portuguese): [docs/PROJETO.md](docs/PROJETO.md).
Every version here becomes a GitHub release automatically (tag + these notes): `.github/workflows/release.yml` runs on each push to main. `scripts/release.sh` is the manual fallback.

## v0.37.2 (2026-10-13): The kit centred on the drummer

The kit diagram is laid out around where the drummer sits: the snare in the middle, between the
knees; the two toms right in front of it; the kick to the right, under the right foot; the hi-hat
on the left and the floor tom on the far right, with crash and ride above. Drumie's hands and feet
follow the new layout, with a little more room at the bottom for the hands.

## v0.37.1 (2026-10-13): Floating hands and feet

Drumie on the kit is now just his floating forearms, hands, sticks and feet, without the body, so
the drums stay visible and the movement reads better. Each stroke is easier to see: the hand comes
up toward you (bigger) and back between strokes while the stick shortens, then lands with a ring;
the right foot lifts before a kick and presses on it. Both feet are marked R and L.

## v0.37.0 (2026-10-13): Song difficulty; Drumie plays the kit

**Difficulty, 1 to 10.** Every song gets a rating per level (Full, Medium, Easy) and a band:
Beginner, Intermediate, Advanced or Expert. It weighs how many strokes per second the hands play in
the busiest bars (35%), the kick (20%), kicks and snares on 16th off-beats (15%), the share of
bars that are fills (10%), and how many different grooves and one-off bars there are (20%). It rates
the notes, not the BPM, so a song detected at half or double tempo rates the same. The Library has
a Difficulty column and sorts easiest or hardest first; the header shows the open song's rating at
the chosen level, with the reasons on hover. Ratings are cached and redone when you fix the song.
On MDB Drums the order looks right (country and plain rock 1 to 2.5, disco and punk around 4,
funk and fusion 5 to 6, latin and modal jazz 6.4 to 7.3), but the weights are a first guess: tell
me songs whose rating feels wrong.

**Drumie plays the kit.** On the kit diagram, Drumie's forearms, hands (R and L), sticks and feet
move with the music: the hand travels to the next drum just before its hit, the stick is raised
between strokes (shorter when seen from above) and lands on the hit with a ring, and the right foot
presses the kick pedal. Hands follow right-handed sticking: right hand crossed over to the hi-hat,
left on the snare, crash and floor tom on the right, fast fills and rolls alternating. A **Drumie**
button next to Kit hides or shows him.

Also: a groove played on the toms in most bars is no longer treated as one long fill by the Easy
level (it used to come out empty).

## v0.36.0 (2026-10-13): The mixer in every section; buttons styled in every browser

**Tracks, in the play bar.** The Mixer section is gone: a **Tracks** button in the play bar opens
the mixer right above it, in Learn, Play, Fix, Settings and Export, so you change what you hear
without leaving what you're doing. Same controls as before (presets, a volume with 0/25/50/75/100,
mute and solo per instrument, the Song master), in a compact layout; open or closed is
remembered. An old `#/mixer` link opens Learn with the Tracks open.

**Buttons styled in every browser.** The player's styles were written with CSS nesting, which
browsers before Chrome/Edge 112, Firefox 117 and Safari 16.5 ignore completely, leaving white,
unstyled buttons. The page now gets a flattened copy that every browser reads; checked identical
to the nested version on all 41,574 element styles of the 7 sections.

## v0.35.0 (2026-10-13): Levels: Full, Medium, Easy

**Play the song before you can play all of it.** A level switch at the top of the page:
- **Full**: everything Drumie heard (what you had before).
- **Medium**: no ghost notes, everything on 8th notes (16th hi-hats become 8ths, 16th kicks and
  snares go), fills kept on 8ths.
- **Easy**: hi-hat or ride on quarter notes, snare and kick only on the beat, a crash only on beat 1,
  and every fill replaced by the groove before it, so the beat keeps going.

The grid, the steps, the kit and the highway (and its score) follow the level, and the level is
remembered per song. Medium and Easy are derived from the full transcription by rules, so a fix you
make shows up in every level; the Fix section always works on Full and switches to it. Never more
than two hands at once; a shuffle keeps its swung upbeat.

On 8 MDB recordings, Medium keeps 55 to 98% of the notes and Easy 33 to 69% (one jazz track keeps
93% in Easy: its many short "fills" become full groove bars). A bonus: the easier levels drop most
of what the AI gets wrong (ghost notes, soft hits).

## v0.34.0 (2026-10-13): Free, closed source

**New license.** From this version on Drumie is free to use but closed source: personal,
non-commercial use, no redistribution or modification ([LICENSE](LICENSE)). Versions 0.24.0 to
0.33.0 were published under the MIT License, and copies of those versions keep it. The licenses
of the software and models Drumie is built on are listed in
[THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md). No change to the app itself.

## v0.33.0 (2026-10-13): Drumie

**A name and a face.** The app is now **Drumie**, after its mascot: a crash test dummy drummer,
lanky and a bit clumsy, because learning drums means taking some hits. Logo: "drumi" with the
crash test target as the final "e" (two opposite quarters filled, the opening of the "e" at the
bottom right). The colors stay the same; the yellow that was already the accent becomes the brand
color, paired with the dummy's joint black.

- **Drumie in the app**: in the empty Library ("No songs yet. I'm ready to take some hits."),
  when no song is open, falling over when a link can't be analyzed, and on every challenge result:
  he celebrates an A or S, sweats when you're off the beat ("You're 45 ms behind me."), and falls
  over below 60%.
- **The test target** is the favicon and spins while a song is analyzed (still with reduced motion).
- New `drumie` command; `drum-practice` keeps working. Reinstall once (`pip install -e .`).
- A link that can't be downloaded (Spotify, etc.) prints one line in the terminal instead of a traceback.
- Brand board (logo, icon, mascot poses, colors) designed first as a mockup; the mascot is drawn in
  code (`musicpractice/ui/drumie.js`), four poses from joint angles.

## v0.32.0 (2026-10-12): A new interface in sections, with a Library

**Sections instead of one long page.** The app now opens on a **Library** (add a song by link or
file with a progress bar, open, analyze again, delete with two clicks, search, disk use), and the
open song has one section per job in a sidebar: **Learn** (kit + grid or steps), **Play**
(highway), **Train** (calibration, challenges, history), **Fix transcription** (the grid full
width, the kit out of the way), **Mixer** (a channel per track, presets, solo/mute), **Settings**
(kit connection, a pad map table with Learn buttons, timing, storage limit, keys) and **Export**
(downloads and the practice track). One play bar at the bottom works in every section, and the
music keeps playing while you switch. The address shows the section (`#/play`), so the browser's
back button works and a reload reopens the last song where you were. On laptops (up to 1500 px)
the sidebar becomes a narrow rail so the grid keeps two bars per line; on a phone it becomes a
row of tabs and nothing scrolls sideways.

**Its own server.** Gradio is replaced by a small FastAPI server for the drums app
(`drum-practice`): the page is plain HTML, saves go through a JSON API instead of a hidden Gradio
text box, analyses run in a queue with progress you can leave and come back to, and audio is only
served from the songs folder. The older one-page Gradio interface stays as `music-practice` for
the other instruments. **Reinstall once** after updating (`pip install -e .`).

**Copy this bar to every repeat** (Fix transcription): fix one bar and every other bar of the same
groove (same pattern, same length) gets exactly the same hits, as hand edits that survive
re-analysis and can be reset.

**Training history**: every challenge you play, newest first, with tempo, score, grade and timing.

**Robustness**: song files are written atomically (a library listed during an analysis no longer
reads a half-written file), and an unreadable song no longer breaks the Library.

## v0.31.0 (2026-10-12): Sync review, readable timing, Training panel

**Sync review.** Three causes of notes and music not lining up, measured and fixed:
- ADTOF puts every hit one frame early: on MDB Drums the median distance from its hits to the hand-annotated onsets is -10.0 ms for every piece (1,009 hits on 6 tracks with the app's own transcriber). Hits are now moved 10 ms later; songs already analyzed are moved once, with their grid, when opened (no model run).
- After calibrating, the judging was right but the picture still ran ahead by the audio delay the browser doesn't report (USB interface): the calibrated latency now also moves the notes ("Notes follow the calibration", on by default).
- Right after play or a jump the highway's clock predicted ahead of a media clock that hadn't started moving: the first notes read late. It now follows the media clock for the first 250 ms. A hit exactly where you press play also flashes now.
- Checked and fine: the Opus copies decode with no offset (0 samples, ffmpeg and Chromium).

**Readable timing in the highway.** Bands around the hit line drawn to scale (Perfect 30 ms bright green, Good 60 ms, OK 100 ms), a slot at the line in the shape of each note that lights up when a note is in the Perfect window, and a line through every note marking its exact time.

**Training panel** (new button next to Highway). *Calibration* moved here. *Challenges*: nine basic rhythms (quarter notes, 8th-note hi-hat, rock beats, four on the floor, kick on the & of 2, 16ths, paradiddle, groove + fill) at 60 to 140 BPM, with a metronome: one bar of clicks, then 4 bars falling like the highway. At the end: score (Perfect 100, Good 80, OK 50, Miss 0 per note, minus 2% per extra hit), grade S/A/B/C/D with stars, timing (rushing/dragging), steadiness, accuracy per piece, a suggestion for the next tempo, and the best score per rhythm and tempo.

## v0.30.0 (2026-10-12): Calibrate with rudiments, one-tap volumes

**Calibrate with a rudiment.** Pick quarter notes, single strokes (8ths), single stroke roll, double stroke roll or single paradiddle, and 70 / 100 / 120 BPM. One bar of clicks to get ready (big 4-3-2-1), then 8 beats of the rudiment on any pad: the sticking scrolls with the current stroke lit, and an early/late meter shows a dot per hit (blue right hand, orange left). The latency is the median distance from each hit to its stroke, so a few flubbed strokes don't move it; the result shows the spread (how much to trust it) and whether the left hand sits behind the right. Fewer than half the strokes matched: nothing changes.

**One-tap volumes.** Every mixer strip has 0 / 25 / 50 / 75 / 100 buttons; a new "Song" master strip scales everything. In the highway: "Drums in the song" and "Song volume" buttons, a "Bar back" button, and a **play/pause pad**: learn any pad you don't use and play / pause without putting the sticks down. New "Hear" presets: play along with the original drums at 25, 50 or 75%.

## v0.29.0 (2026-10-12): Highway in sync, with impact

**Notes follow the music you hear.** The highway used the player's media position, but the sound reaches your ears later by the output latency (Windows audio plus an audio interface): the notes crossed the line before you heard them, and on-time hits were judged late. The highway now runs on a clock moved back by the output latency the browser reports (32 ms in the test browser), and hits are judged on the same clock. A new **Note delay** control shifts only the picture, for latency the browser can't see (Bluetooth, TV).

**Bigger and louder hits.** Taller highway (78% of the screen), wider notes, thicker hit line and pads. Each hit bursts on its pad (shockwave ring, sparks, a glow up the lane), PERFECT / GOOD / OK in big text that pops in (with "early 23 ms" under it), MISS in red, a combo counter that bumps on every hit, and banners at 10, 25, 50 and 100 in a row. One text per lane, so fast hi-hats don't pile up. Reduced-motion settings drop the sparks and the pop-in.

## v0.28.0 (2026-10-05): Disk space

**Disk space: 25 GB -> 1.7 GB** on a real workspace of 56 songs.

- After analysis, every WAV is replaced by its Opus copy (what the player already played) and scratch audio is deleted: ~450 MB -> ~30 MB per song.
- Disk limit (default 5 GB, `--max-gb`): past it, the audio of the songs opened longest ago is removed. Transcription, sensitivity and hand edits stay; opening the song again downloads and separates it again without re-transcribing.
- `drum-practice --clean` shrinks songs analyzed by older versions. Gradio's temp folder is cleared hourly.

**Measured:** re-transcribing from Opus instead of WAV changes nothing that matters (ADTOF on MDB Drums, F1 within 0.004 per piece; toms within 0.013).

## v0.27.0 (2026-10-03): Pulses on the playhead

**Grid marks pulse when the playhead crosses them.** They used to pulse at the hit's audio time; a hit off the 16th grid (8% of hits are more than 40 ms off, even on hand-annotated beats) pulsed before or after the line reached its mark, so the line seemed to pass notes that didn't react. The kit still flashes on the audio time.

**Lighter animations.** Pulses use Web Animations instead of class toggling with a forced page layout on every hit.

## v0.26.1 (2026-10-03): Visible folded panels

**Folded panels are easy to find.** The mixer remembers being folded, so a folded mixer looked gone. Its title is now a header bar with a yellow chevron and show/hide; the upload and accordion titles are brighter with accent-colored arrows.

## v0.26.0 (2026-10-03): Beat figures

**Beat figures.** Every beat of the bar gets a letter, so repeated beats are visible at a glance: plain rock reads "A B C B" (A = kick + hi-hat, B = snare + hi-hat, C = kick on the beat and the "&"). A chip per figure shows a drawing of its four 16ths and how often it is played; clicking it lights every beat that plays it.

**Measured** (MDB Drums rock/pop/funk, grouping from the transcription vs from the hand annotations, adjusted Rand index): 0.84 on the drum stem, 0.72 on the full mix. Cut 0.3 chosen over 0.4 because 0.4 merges "kick on the beat" with "kick on the beat and the &", a difference the drummer has to play.

## v0.25.0 (2026-10-02): One look for the whole page

**One look for the whole page.** The form around the player now uses the player's own dark theme with one yellow accent; the song title, BPM, bars and length sit at the top of the player.

**Fewer fields.** In drums-only mode the instrument picker, checkboxes, duplicate audio players and explanatory text are gone: paste a link, press Analyze.

## v0.24.0 (2026-10-02): First public version

First public version: paste a song link (or upload a file) and get a drum lesson to play on an electronic kit.

**What it does**
- Separates the drums (Demucs htdemucs_6s, drums re-separated with the htdemucs_ft drum model) and transcribes every hit with ADTOF.
- Moves the bar grid onto the drummer (timing, beat 1, half-speed detection), so the same groove is drawn the same way.
- Finds grooves, fills and practice steps; synced player with kit diagram, drum grid, Steps view, mixer and a MIDI highway for e-kits.
- Everything runs locally.

**Measured** (MDB Drums, 23 real recordings, F1): kick 0.93, snare 0.77, hi-hat 0.86, cymbals 0.86, toms 0.46. Bar lines on the right beat 88% of the time (65% with the beat tracker alone).

Versions v1 to v23 were developed before this repository existed; their history, decisions and measurements are in `docs/PROJETO.md`.

## Before this repository (v1 to v23)

Developed before the code was on GitHub; v0.24.0 contains all of it.

- v1-v3: base app: link in, separation, analysis, transcription, tab, practice tracks, export; runs on WSL.
- v4: one-page player synced with the music.
- v5-v7: drums: kit diagram, same colors in the grid, ADTOF transcription.
- v8: bigger two-line grid; one player at a time; whole-piece flash animations.
- v9: resizable kit/grid layout, live mixer per instrument.
- v10: Guitar Hero style highway for MIDI e-kits.
- v11: transcription redone from the isolated track once it exists.
- v12: "Fix transcription" panel: sensitivity per piece, editing in the grid.
- v13: playhead synced with the marks; typeable slider values.
- v14: continuous staff per line.
- v15: MIDI monitor in the highway.
- v16: "three hands" rule (no unplayable hits).
- v17: smooth follow scroll.
- v18: repeated patterns (grooves, fills, one-offs) with colors and loop per pattern.
- v19: tolerant pattern matching (weights + clustering, ARI 0.47 -> 0.73) and the Steps view.
- v20: grip to choose how many lines ahead to see.
- v21: bar grid reaches the end of the song (fade-outs and quiet endings had no transcription).
- v22: bar grid follows the drummer (timing, beat 1, half speed); repeatable Demucs.
- v23: a kick heard twice (as a weak snare/hi-hat echo) becomes one hit.
