# split-stems

Pull any song apart into its parts — vocals, drums, bass, guitar, piano — as
files that drop straight onto the grid in Ableton, Logic, or FL.

Runs entirely on your own machine. Nothing uploads, nothing costs money per use,
and it works offline once the models are downloaded.

Everything you need is on this page.

---

## What you get

Point it at a folder of songs. Every song comes back as its own folder:

```
~/Music/stems/
  01 Happiness [120bpm]/
      vocals_120bpm.wav
      instrumental_120bpm.wav
      drums_120bpm.wav
      bass_120bpm.wav
      guitar_120bpm.wav
      piano_120bpm.wav
      other_120bpm.wav
```

With `--max` you also get the drum kit taken apart — `kick`, `snare`, `toms`,
`hh`, `ride`, `crash` as separate files. That is a one-shot library out of any
record you own.

Roughly **45 seconds of processing per minute of music**, or about 50 seconds
per minute with `--max`, on an Apple Silicon Mac.

---

## Install

Mac with Apple Silicon (M1 or later). About 10 minutes, most of it downloading.

### 1. Prerequisites

If you don't have Homebrew, install it first from [brew.sh](https://brew.sh).

```bash
brew install python@3.11 ffmpeg
```

Python **3.11 specifically** — the audio libraries this depends on don't publish
builds for every newer version, and a newer Python will fail with confusing
errors about missing wheels.

### 2. The engine

```bash
python3.11 -m venv ~/.venv-stems
~/.venv-stems/bin/pip install "audio-separator[cpu]==0.44.5" librosa
```

This downloads a few GB. It is a one-time cost.

### 3. The command

```bash
mkdir -p ~/bin
curl -fsSL https://raw.githubusercontent.com/hareeshnagaraj/split-stems/main/split-stems -o ~/bin/split-stems
chmod +x ~/bin/split-stems
```

### 4. Wire it up

```bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
echo 'export STEMS_VENV="$HOME/.venv-stems"' >> ~/.zshrc
source ~/.zshrc
```

### 5. Check it

```bash
split-stems
```

You should see a usage line. If you get `command not found`, close the terminal
and open a new one.

---

## Use it

```bash
split-stems ~/Music/some-folder-of-songs
```

Results land in `~/Music/stems`. Takes wav, mp3, m4a, aif, aiff,
and flac.

```bash
split-stems <folder> <somewhere-else>    # different output folder, this once
split-stems <folder> --max               # also split the drum kit apart
split-stems <folder> --flac              # keep lossless copies as well as WAV
```

Set `STEMS_OUT` in your `~/.zshrc` if you want a different permanent home.

The first run downloads the separation models (a few hundred MB) and will be
slower than the rest. By default they land in `/tmp`, which macOS clears now
and then; set `STEMS_MODELS` to a folder to download them only once:

```bash
echo 'export STEMS_MODELS="$HOME/.stems-models"' >> ~/.zshrc
```

---

## Getting them onto the grid

1. Set your project tempo to the BPM in the folder name
2. Drag in **all** the stems for that track together
3. Select them all and warp them as one selection

Every stem of a track is exactly the same length, so once one clip sits on the
grid, they all do. Nudge one and they move together.

**Why the tempo is in the filename.** Compressed formats carry no tempo
information, so a DAW guesses — and it guesses differently for a sparse bass
stem than for a dense drum stem. The stems then drift against each other and you
end up rewarping every clip by hand. Writing the tempo into the name and
exporting WAV removes the guess.

---

## What it's bad at

Worth knowing before you spend an afternoon on it.

- **BPM detection is a starting point, not a guarantee.** It's confident on
  steady four-on-the-floor and unreliable on rubato, live drumming, and anything
  with a tempo change. Beat trackers also lock onto whichever pulse is
  strongest — often the half-time one — so a 120 BPM track can read as 60. The
  script corrects for that by preferring 90–160, but check anything that feels
  half or double speed.
- **Non-Western instrumentation gets folded together.** The model was trained
  mostly on Western pop and rock, so tabla, dhol, oud and similar tend to land
  in `other` rather than getting their own stem. Drums and bass still separate
  cleanly.
- **Dense mixes bleed.** Heavily layered or loudness-maximised masters separate
  worse than open arrangements. This is a limit of the models, not the settings.
- **Vocals under heavy effects** — reverb tails and delays often follow the
  vocal into its stem, or smear into `other`.

---

## If something breaks

**`command not found: split-stems`** — `~/bin` isn't on your PATH. Open a new
terminal, or run step 4 again.

**`missing: .../audio-separator`** — the engine isn't installed or `STEMS_VENV`
points at the wrong place. Check `echo $STEMS_VENV` names a folder that exists.

**`ffmpeg not found`** — `brew install ffmpeg`.

**Install fails on wheels or build errors** — you're almost certainly not on
Python 3.11. Check with `~/.venv-stems/bin/python --version`, and rebuild the
venv with `python3.11` explicitly if not.

**It seems frozen** — nothing is written until a whole pass finishes, so a long
track can look stuck for minutes. Confirm it's alive with
`pgrep -fl audio-separator`.

---

## Under the hood

| Pass | Model | Produces |
|---|---|---|
| 1 | BS-Roformer (`ep_368`) | vocals, instrumental |
| 2 | htdemucs_6s | drums, bass, guitar, piano, other |
| 3 | MDX23C-DrumSep | kick, snare, toms, hh, ride, crash (`--max` only) |

Models come from the [audio-separator](https://github.com/nomadkaraoke/python-audio-separator)
project and download themselves on first use. Separation runs on the GPU via
Metal, so **don't run two copies at once** — they'll fight over the same device
and both get slower.
