# Story Package Format

Spec version 0.1 (draft), 2026-10-18.

## 1. Purpose and scope

A story package ("tale") holds one story for the offtrail-tales engine: the Ink text and the media it uses. Packages ship separately from the app. The app downloads them, keeps them for offline play, and updates them. Players can also import a package from a single file.

This spec defines:

- the source form (what authors keep in the repo), and its project form, which makes the media from masters,
- the distribution form (what the build publishes and the app downloads),
- the single-file form, `.tale` (what players share and import),
- the manifest, `tale.json`,
- the tags that link story text to media,
- runtime, download, import, validation and security rules.

Out of scope for v0.1: several languages in one package, engine UI and layout, the save-game format. See section 13.

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as described in RFC 2119.

## 2. Terms

| Term | Meaning |
| --- | --- |
| Package, tale | One story with its media, identified by a package ID. |
| Slot | A top-level class of content in a package. v0.1 has two: `story` and `media`. |
| Asset | One media item, addressed by an asset ID. One asset can have several files (alternative encodings of the same content). |
| Master | The original file an asset is made from. Masters live outside git, for example in a synced folder, and are never part of a package. |
| Build | The tool that turns the source form into the distribution form. |
| Engine, app | The runtime that plays packages. |
| Core assets | The engine's own assets (GUI backgrounds, UI art). They are not part of any package. |

## 3. Identifiers

- Package IDs and asset IDs MUST match `^[a-z0-9]+(-[a-z0-9]+)*$` (lowercase kebab-case) and MUST be at most 64 characters.
- `none` and `default` are reserved and MUST NOT be used as asset IDs (see section 7). So is `constructor`, a name every JavaScript object has already.
- The package folder name MUST equal the package ID.
- An asset ID is the file name without its extension: `media/img/forest-clearing.avif` has the ID `forest-clearing`.
- Asset IDs MUST be unique in the package across all media types. `img/forest.avif` and `audio/forest.m4a` in one package is an error.
- File and folder names, including extensions, MUST be lowercase.

## 4. Source form

```text
stories/<id>/
├── tale.json          # manifest: hand-written fields only (6.1)
├── story/
│   ├── main.ink       # entry point, required
│   └── ...            # more .ink files, subfolders allowed
├── media/             # optional
│   ├── img/           # .avif, optional .webp fallback
│   ├── audio/         # .m4a
│   └── video/         # .mp4
└── tests/             # optional: chain tests, never packed
```

- `story/main.ink` MUST exist. It is the entry point.
- `story/` MUST contain only `.ink` files. Subfolders are allowed.
- A package MAY have no media at all. Most packages are expected to have at least their own background image, but this is not required.
- `media/` MAY contain only the folders `img/`, `audio/` and `video/`. Each is optional. Files MUST sit directly in these folders, without subfolders, so that asset IDs stay unambiguous.
- The package root MUST NOT contain anything other than `tale.json`, `story/`, `media/` and `tests/`. Names that start with `.`, such as `.DS_Store`, are ignored, here and in `story/` and `media/`. Tools keep their files in `.build/` (Appendix A).
- `tests/` MAY hold chain tests: `.chain` files, without subfolders. They are for the tools (Appendix A), not part of the package: the build never packs them.
- Masters MUST NOT be in the package.

### 4.1 Project form

In the project form, a tale keeps its text in git and makes its media from masters that live anywhere, for example in a synced folder. It replaces `media/` with a list of masters and render settings. One build renders the masters, stages the source form and builds that. A folder without `assets.json` is plain source form and builds as described above.

```text
<id>/
├── tale.json          # as in the source form (6.1)
├── story/             # as in the source form
├── tests/             # as in the source form
├── assets.json        # hand-written: asset ID -> master and render settings
├── assets.lock.json   # written by the build: hashes and tool versions
├── assets/            # the mount: a folder, or a symlink to one
└── .build/            # cache and staging
```

- The project root MUST NOT contain anything other than `tale.json`, `story/`, `tests/`, `assets.json`, `assets.lock.json` and `assets/`. Dotfiles are ignored. A `media/` folder next to `assets.json` is an error.
- `assets/` MUST be a folder, or a symlink to one. It holds the masters. To mount a folder: `ln -s <folder with masters> <id>/assets`.
- `assets.json`, `assets.lock.json`, `tale.json`, `story/` and `tests/` belong in git. `assets` and `.build/` do not: the tale's repository SHOULD ignore them, or a version from git (6.2) always ends in `-dirty`.

`assets.json` is a UTF-8 JSON object. Each key is an asset ID (section 3). Each value is an object:

| Field | Type | Required | Meaning |
| --- | --- | --- | --- |
| `from` | string | yes | Path of the master inside `assets/`, `/` separated. It MUST NOT be absolute or contain a `..` segment, and the file MUST exist. Symlinks are followed. |
| `gain_db` | number | no | `audio` only. Changes the volume by this many decibels, for example `-6`. |
| `start` | number | no | `video` only. Seconds to skip at the start of the master, 0 or more. |
| `duration` | number | no | `video` only. Seconds to keep, more than 0. |

Unknown fields are a build error, so typos do not pass silently.

```json
{
  "no-exp-kitchen": { "from": "img/no-exp-kitchen.png" },
  "noir-crime-tension": { "from": "music/noir/jorisvermeer-noir-crime-tension-535274.mp3", "gain_db": -6 },
  "intro-clip": { "from": "video/raw/intro.mov", "start": 26, "duration": 8 }
}
```

The extension of the master, in any case, gives the asset type and what the build does with it:

| Master | Type | Media file (5.2) |
| --- | --- | --- |
| `.png`, `.jpg`, `.jpeg` | `img` | rendered to `.avif` and `.webp` |
| `.avif`, `.webp` | `img` | copied as it is |
| `.wav`, `.mp3`, `.aif`, `.aiff`, `.flac` | `audio` | rendered to `.m4a` |
| `.m4a` | `audio` | copied as it is, or rendered when it has `gain_db` |
| `.mov`, `.m4v` | `video` | rendered to `.mp4` |
| `.mp4` | `video` | copied as it is, or rendered when it has `start` or `duration` |

Any other extension is an error. The media file is `media/<type>/<asset ID>.<extension>`, and all checks of the source form apply to it. An `img` asset from a `.webp` master alone has no `.avif` file, so it fails.

Render commands. Parts in brackets are used only when the setting is there:

```text
avifenc -q 60 -s 6 <master> <file>.avif
cwebp -quiet -q 80 <master> -o <file>.webp
ffmpeg -nostdin -i <master> -vn -map_metadata -1 [-af volume=<gain_db>dB] -ar 48000 -c:a aac_at -b:a 128k
       -movflags +faststart -fflags +bitexact <file>.m4a
ffmpeg -nostdin [-ss <start>] -i <master> [-t <duration>] -map_metadata -1 -c:v libx264 -threads 8
       -pix_fmt yuv420p -crf 23 -preset medium -ar 48000 -c:a aac_at -b:a 128k -movflags +faststart
       -fflags +bitexact <file>.mp4
```

- The image commands are the ones `tools/import-assets.sh` uses.
- `-map_metadata -1` and `+bitexact` keep dates and tool names out of the files, so the same master renders to the same bytes.
- x264 otherwise takes its thread count from the number of CPU cores, and the count changes its output. `-threads 8` fixes the count.
- `aac_at` is the AAC encoder of macOS, so audio and video renders need a Mac.
- `aac_at` takes at most 48 kHz, and studio masters are often 96 kHz, so `-ar 48000` resamples all sound to 48 kHz.
- The tools come from Homebrew (`brew install ffmpeg libavif webp`). A missing tool stops the build.

Cache. A rendered file is kept in `.build/cache/<key>.<extension>`. `<key>` is the SHA-256 of the master's SHA-256, the render command with its settings, and the tool version. A file in the cache is used as it is. The tool version is the first line of `ffmpeg -version`, `avifenc --version` or `cwebp -version`. As `aac_at` is part of macOS, an `ffmpeg` render also counts the macOS version, `macos` in the lock (`sw_vers -productVersion` and `-buildVersion`), so an OS update gives new cache keys and a lock warning. Deleting `.build/` is always safe.

Lock. The build writes `assets.lock.json`. Authors commit it and do not edit it. An example, with hash values shortened:

```json
{
  "assets": {
    "no-exp-kitchen": {
      "bytes": 2315123,
      "from": "img/no-exp-kitchen.png",
      "outputs": {
        "media/img/no-exp-kitchen.avif": "c41f…",
        "media/img/no-exp-kitchen.webp": "9a07…"
      },
      "sha256": "5be2…"
    }
  },
  "format": "0.1",
  "tools": {
    "avifenc": "Version: 1.4.2 (dav1d [dec]:1.5.4, aom [enc/dec]:3.14.1)",
    "cwebp": "1.6.0"
  }
}
```

- `format` is the spec version. `tools` holds the version of each tool the renders use. `assets` holds, for each asset, its `from`, the `sha256` and `bytes` of the master, and in `outputs` the `sha256` of each media file.
- Keys are sorted, the indent is 2 spaces, and the file ends with a newline, so the same lock always has the same bytes.
- The build MUST fail when the lock is missing, when an asset is not in it, when an entry has another `from` or master `sha256`, or when it has an asset that `assets.json` does not. It lists every problem, plus the way out: `npm run pack-tale -- <folder> --update-lock`, then commit `assets.lock.json`.
- Masters are not in git, but the lock is. A master that changes therefore changes the tale only through a commit, and a version from git (6.2) changes with it.
- A tool version that differs from the lock, or a media file whose `sha256` differs from `outputs`, is a warning.

Build order:

1. Check the project root and `assets.json`.
2. Hash every master and check it against the lock. `--update-lock` skips this check.
3. Render each master that needs it into the cache, or take the file from the cache.
4. With `--update-lock`, write the lock.
5. Delete and re-create `.build/stage/<id>/`. Copy `tale.json` and `story/` into it, and the media files into `media/`. This is the source form.
6. Build the stage folder as in section 10, then play it as `pack-tale` does (Appendix A). A missing `version` comes from git for the project folder, not the stage folder (6.2).

Command line:

```text
npm run pack-tale -- <folder> [output folder] [--update-lock]
```

The flag may come in any position. The output is as for the source form, plus one line that counts the media files rendered, taken from the cache, and copied. `--update-lock` on a folder without `assets.json` is an error.

## 5. Slots

### 5.1 Story slot

- Ink source files, UTF-8. `INCLUDE` paths are relative to `story/` and MUST NOT point outside it.
- The build compiles `story/main.ink` (inkjs compiler or inklecate) into one ink JSON file, `story/main.ink.json`.
- The distribution form contains only the compiled file. `.ink` sources are not shipped.
- Stories MUST NOT declare `EXTERNAL` functions. v0.1 engines bind none.
- Global tags (tags at the top of `main.ink`) are not used in v0.1. Package metadata lives in `tale.json`.
- Story text may contain the inline formatting tags `<b>`, `<i>`, `<em>` and `<strong>`. The engine keeps these and renders them as styled elements. It removes all other standard HTML tags (such as `<div>` and `<script>`), which Inky may show as formatting, and keeps the text between them. Anything else in angle brackets, such as `<Enter>`, stays as written.
- Speech is written as a name of up to three words, a colon, a space and the words: `Guard: You're late.` An app MAY show such a line as speech, with the name as a label once over that speaker's lines in a row. The story text does not change.

### 5.2 Media slot

| Folder | Asset type | Required format | Optional extra | Size per file |
| --- | --- | --- | --- | --- |
| `img/` | `img` | AVIF (`.avif`) | WebP (`.webp`) fallback | SHOULD: long edge at most 2560 px, at most 500 KB |
| `audio/` | `audio` | AAC-LC in MP4 (`.m4a`) | none | MUST: at most 10 MB |
| `video/` | `video` | H.264 video + AAC audio in MP4 (`.mp4`) | none | MUST: at most 200 MB |

- Every `img` asset MUST have an `.avif` file. A `.webp` file with the same ID MAY be added as a fallback.
- Transparency (for example character tiles) uses the alpha channel of AVIF or WebP.
- Other formats (PNG, JPEG, SVG, MP3, ...) MUST NOT be in `media/`. SVG is excluded because it can carry scripts.
- The build warns when a picture goes over its size, and fails when a sound or a video does. The tale editor refuses such a sound or video. At the editor's bitrates, 10 MB is about 10 minutes of sound (128 kb/s), and 200 MB about 5 minutes of video (5 Mb/s, at most 1920 px wide).
- A tale's media SHOULD take at most 1 GB in all. The build and the editor warn past that: a big tale takes long to import, and takes much of a player's disk space.
- A sound effect (`sfx`, section 7) is decoded fully into memory, so keep it short: long sounds belong in `music`.

## 6. Manifest: `tale.json`

UTF-8 JSON object.

### 6.1 Hand-written fields (source form)

| Field | Type | Required | Meaning |
| --- | --- | --- | --- |
| `format` | string | yes | Spec version the package follows, as `"major.minor"`. `"0.1"` for this spec. |
| `id` | string | yes | Package ID. Equals the folder name. |
| `version` | string | no | Package version, SemVer (`"1.2.0"`). Bump it on every published change. If it is left out, the build makes one from git (6.2). |
| `title` | string | yes | Display title. |
| `language` | string | yes | BCP 47 language tag of the story text, for example `"en"`, `"nl"`, `"sv"`. |
| `authors` | string[] | yes | At least one entry. |
| `description` | string | no | Short text for the catalog. |
| `cover` | string | no | Asset ID of an `img` asset used as cover. |
| `license` | string | no | Licence of the package as a whole: an SPDX identifier, a `LicenseRef-<name>` identifier, or `"proprietary"`. |
| `credits` | object[] | no | Credits for media not made by the authors. See below. |
| `theme` | object | no | The tale's own colours, and a picture behind the story. See below. |

Each `credits` entry:

| Field | Type | Required | Meaning |
| --- | --- | --- | --- |
| `assets` | string[] | yes | Asset IDs this entry covers. |
| `by` | string | yes | Creator. |
| `license` | string | yes | SPDX identifier, a `LicenseRef-<name>` identifier, or `"proprietary"`. |
| `source` | string | no | URL of the original. |

A licence that has no SPDX identifier is named in the SPDX form for other licences, as REUSE uses it: `LicenseRef-` followed by a name for the licence.

Every asset not made by the package authors SHOULD be covered by a `credits` entry.

The `theme` object:

| Field | Type | Required | Meaning |
| --- | --- | --- | --- |
| `background` | string | yes | Behind the story, and the surface of bars and windows. |
| `text` | string | yes | Story text. |
| `muted` | string | yes | Secondary text, such as the authors on the title card. |
| `highlight` | string | yes | Choices, speaker names, the chosen option, titles and headings. |
| `border` | string | yes | Borders of buttons and windows, picture frames, and the light fill of buttons. |
| `separator` | string | yes | The lines that set off the chosen option. |
| `picture` | string | no | Asset ID of an `img` asset shown behind the story. |

Each colour is `"#rrggbb"`. The engine wears the theme from the tale's title card until the player leaves the tale. Then the player's own theme comes back. A tale without `theme` plays in the player's theme. `theme` comes last in `tale.json`, with its fields in the order of the table. Unknown fields in `theme` are a build error. [docs/tale-themes.md](tale-themes.md) says what each colour paints in this app.

The source `tale.json` MUST NOT contain the fields `story` or `media`: the build generates them. Unknown fields in the source manifest are a build error, so typos do not pass silently.

### 6.2 Generated fields (distribution form)

The build adds:

- `version`, if the source manifest has none: `<date>-<hash>`, from the commit that the git repository holding the package folder is at (`HEAD`). `<date>` is the commit date as `YYYY-MM-DD`, and `<hash>` is the short commit hash. If the package folder has changes that are not committed, `-dirty` is appended. Examples: `"2026-10-18-a1b2c3d"`, `"2026-10-18-a1b2c3d-dirty"`. The distribution form always has a `version`: SemVer from the author, or this form.
- `story`: object `{ "path", "bytes", "sha256" }` for `story/main.ink.json`.
- `media`: object keyed by asset ID. Each value is `{ "type", "files" }`:
  - `type`: `"img"`, `"audio"` or `"video"`.
  - `files`: alternative encodings in preference order (AVIF before WebP). Each entry is `{ "path", "bytes", "sha256" }`, plus `width` and `height` in pixels for `img` and `video`, and `duration` in seconds for `audio` and `video`. The dimension and duration fields are optional in v0.1. The build SHOULD add them when it can read them.

Rules for generated values:

- `sha256` is the lowercase hex SHA-256 of the file bytes.
- `path` is relative to the package root, uses `/`, and MUST NOT contain `..` or start with `/`.

### 6.3 Example (distribution form)

Hash values are shortened.

```json
{
  "format": "0.1",
  "id": "sample-tale",
  "version": "0.1.0",
  "title": "Sample Tale",
  "language": "en",
  "authors": ["R. Karlsson Linner"],
  "description": "A short walk off the trail.",
  "cover": "forest-clearing",
  "license": "proprietary",
  "credits": [
    { "assets": ["owl"], "by": "Example Sound Co.", "license": "CC-BY-4.0", "source": "https://example.org/owl" }
  ],
  "story": { "path": "story/main.ink.json", "bytes": 18234, "sha256": "4f1c…" },
  "media": {
    "forest-clearing": {
      "type": "img",
      "files": [
        { "path": "media/img/forest-clearing.avif", "bytes": 163220, "sha256": "9a02…", "width": 1536, "height": 1024 },
        { "path": "media/img/forest-clearing.webp", "bytes": 208114, "sha256": "c7e9…", "width": 1536, "height": 1024 }
      ]
    },
    "night-theme": {
      "type": "audio",
      "files": [{ "path": "media/audio/night-theme.m4a", "bytes": 1804511, "sha256": "11bd…", "duration": 112.4 }]
    },
    "owl": {
      "type": "audio",
      "files": [{ "path": "media/audio/owl.m4a", "bytes": 40211, "sha256": "e83a…", "duration": 1.8 }]
    }
  }
}
```

## 7. Tags: linking story to media

The story refers to media only through Ink tags, and only by asset ID.

Grammar: the tag text (after `#`, trimmed) has the form `<key>: <arg> [<arg> ...]`. The key matches `^[a-z][a-z0-9-]*$`. Arguments are separated by whitespace.

| Tag | Argument | Effect | Lasts |
| --- | --- | --- | --- |
| `bg: <id>` | `img` asset | Sets the scene background. | Until the next `bg` tag. |
| `bg: none` | | Clears the background. | |
| `img: <id> [<frame>]` | `img` asset, then optionally a frame | Shows the image with the current line (illustration, character tile), in a frame. | This line only. |
| `music: <id>` | `audio` asset | Starts the track, looping, and replaces the current track. | Until the next `music` tag. |
| `music: none` | | Stops the music. | |
| `music: default` | | Plays the engine's default music (section 8). | |
| `sfx: <id>` | `audio` asset | Plays the sound once, from start to end. The story cannot stop it. | Until the sound ends, or the player leaves the tale. |
| `video: <id>` | `video` asset | Plays the video once. The story continues when it ends or the player skips it. | |
| `map: <id>` | `img` asset | Sets the map: a button over the story opens the picture, which the player can zoom and move. | Until the next `map` tag. |
| `map: none` | | Takes the map button away. | |
| `wait: <n>s` | a time, such as `17s` | Holds back the rest of the passage, its choices included, for that many seconds. The player cannot skip it. | That many seconds. |
| `input: <variable> [keypad]` | a global `VAR` that starts as text, then optionally `keypad` | Only on a choice, inside its `[brackets]`. Taking the choice asks the player for text, which goes into the variable. With `keypad`, the window shows the app's keypad picture. | Until the ink changes the variable. |

Rules:

- **When tags run.** A passage is everything from one set of choices to the next: the text under the chosen option, and any knots the story moves on to before it offers choices again. The engine shows a whole passage at once. It handles the tags of each line (inkjs `currentTags`) in order, before it shows that line, so all tags in a passage take effect within a moment of each other, even tags many lines apart. To space them out, use `wait`. Tags at the start of a knot belong to its first line.
- **`wait`.** The one exception to the rule above: a `wait` starts after its line is shown. Then it holds back the rest of the passage and its choices. Use it to let a sound play out before the player can move on, or for a dramatic pause. It counts seconds, not the sound: to let a sound play out, set it to the sound's length plus about a second for loading. The player cannot skip a wait, so keep waits short and rare.
- **`sfx` or `music`.** `sfx` plays a sound once, from start to end. The story cannot stop it or fade it, and several sounds can play at the same time. `music` loops until the next `music` tag, and only one track plays at a time. Use `sfx` for a one-off sound, such as a door or an ad that plays once. Use `music` for a sound that keeps going, or that the story must be able to stop, such as a radio left on.
- **Music changes.** A `music` tag MAY end with how the music changes: `cut` switches at once, for example at a scene change, and a time such as `8s` fades the old track out and the new one in over that many seconds. Without it, the engine crossfades over its own default time of a few seconds. This works with `none` and `default` too: `music: none 5s` fades to silence. A `music` tag that names the track already playing changes nothing: the track plays on. Opening a tale is a scene change: in the first passage after a game starts or a save loads, a `music` tag without a time cuts, and so does the music a save restores. A tale that sets no music there plays on with the default music (section 8).
- **Frames.** An `img` tag MAY end with how the picture is framed, as in `img: kitchen print`: `plain` (rounded corners and a shadow), `line` (a thin line), `mat` (a dark mat inside a thin line), `double` (two thin lines), `glow` (a soft glow) or `print` (a photo print on pale paper, slightly tilted). Lines and glow take the colour of the theme: the tale's own (6.1), else the player's. Without a frame, the engine uses its own default. Any other word is an error.
- **Picture size.** The engine shows an `img` picture at half its pixel size, as a 2x picture, so it stays sharp on high-density screens. It is never wider than the story text. Make a picture twice the size it should appear: one 400 px wide shows 200 px wide.
- **`input`.** The one tag that goes on a choice. Write it inside the brackets: `+ [Enter the code #input: door_code]`. Ink gives a tag there to the choice only. A tag after the brackets, or on a choice without brackets, also lands on the line ink prints for the choice, and there `input` is an error. The variable MUST be a global `VAR` that starts as text, such as `VAR door_code = ""`. The engine shows a window with a text field, Cancel, Clear and Submit. Clear empties the field. Submit puts the text into the variable, without spaces around it, and takes the choice. Empty text is allowed. Cancel, or closing the window another way, takes no turn. The engine never interprets the text: the ink decides what it means, for example `{door_code == "1984": -> open}`. Ink compares text exactly, case included. `keypad` after the variable, as in `#input: door_code keypad`, shows the app's keypad picture in the window: its digit keys type, ✱ is Clear and # is Submit. The field and its buttons stay, so the window also works without the picture. The keypad is the app's own picture, not an asset of the tale, and any other word in its place is an error. Media tags act only on lines: inside a choice's brackets they do nothing.
- **Maps.** A tale MAY have any number of maps, or none. Each is an `img` asset, and a `map` tag names the one that fits the story's place, such as a town map at the start of a town's scenes. While a map is set, the engine SHOULD show a button that opens it over the story, where the player can zoom in, move it around and close it. A map is not a picture in the story: for that, use `img`. The engine shows a map at most at half its pixel size, as an `img` picture. Make a map twice the size it should appear, or larger, so the player can zoom in on its details.
- The asset type MUST match the tag: `bg`, `img` and `map` take `img` assets, `music` and `sfx` take `audio` assets, `video` takes `video` assets.
- Tags without `:` are free for author notes. The engine ignores them.
- The engine MUST ignore tags with unknown keys. The build warns about them.
- Ink save state does not include tags. The engine MUST store the effect of every tag that lasts beyond its passage (now `bg`, `music` and `map`) in its own save data, and restore it when a save is loaded. Tags that act only within their passage (`img`, `sfx`, `video`, `wait`) need nothing, as a loaded save replays its passage (8). Neither does `input`: its text is in an ink variable, which the ink state holds.

Example:

```ink
=== clearing ===
# bg: forest-clearing
# music: night-theme
The trail ends at a clearing. # sfx: owl
* [Follow the owl] -> owl_path
* [Make camp] -> camp
```

Example with a sound that plays out before the choices, and a scene change:

```ink
=== kitchen ===
# music: none 5s
The radio is halfway through an ad break. # sfx: radio-ad # wait: 17s
+ [Keep listening]
    The ad ends on a jingle.
+ [Turn it off]
    The kitchen goes quiet.
- -> street

=== street ===
# bg: street-night
# music: rain-theme cut
Outside, the rain has started.
* [Walk to the bus stop] -> bus
```

- `music: none 5s` fades the music that was playing out over 5 seconds.
- `radio-ad` is a 16-second sound that plays once, so it is an `sfx`. The `wait: 17s` holds back the choices until it has played out, with a second to spare for loading.
- After either choice, the line under it and `street` up to its choice form one passage: they appear together, and `street`'s tags run at the same moment. `cut` starts the rain theme at once, for the new scene. Without `cut`, it would fade in over the engine's default time.

## 8. Runtime behaviour

- **Story first.** A tale is playable as soon as `tale.json` and the story file are downloaded and verified. Media MAY still be downloading.
- **Missing media is not an error.** If an asset is not available (not downloaded yet, failed, or no file in a format the engine can decode), the engine MUST continue the story without it. It SHOULD NOT block the player or show an error. It MAY show a placeholder.
- **Default background.** While no `bg` is set, or the set `bg` is missing, the engine shows a default background: the theme's `picture`, else the theme's `background` colour (6.1). For a tale without a theme, it is the engine's own (a core asset).
- **Default music.** Until the first `music` tag, and after `music: default`, the engine plays its own default music (a core asset). After `music: none` there is silence.
- **Sound effects end with the tale.** When the player leaves a tale, for the main menu or another game, the engine MUST stop its sound effects, including ones still loading.
- **A loaded save replays its passage.** A save restores the start of the passage it was made in, with its `bg`, `music` and `map`. A save made before maps existed loads with no map. The engine shows that passage again, so its `img`, `sfx`, `video` and `wait` tags run again.
- **File choice.** The engine uses the first file in `files` that it can decode.
- **A story that runs on stops.** A turn that gives no choice and no ending within 10,000 lines, or one line that takes more than a second, stops the story with an error. Diverts that go in a circle do this, and so does a knot named `__proto__`, which inkjs cannot run. Without the limits, such a story would freeze the page.
- **Format check.** See section 12.

## 9. Distribution and download

### 9.1 Server layout

```text
<base>/catalog.json
<base>/<id>/tale.json
<base>/<id>/story/main.ink.json
<base>/<id>/media/img/<asset>.avif
<base>/<id>/media/audio/<asset>.m4a
...
```

`<base>` is an HTTPS URL on the static host that serves the web app. No separate storage service is needed.

`catalog.json` lists the available tales:

```json
{
  "format": "0.1",
  "tales": [
    {
      "id": "sample-tale",
      "version": "0.1.0",
      "title": "Sample Tale",
      "language": "en",
      "manifest": "sample-tale/tale.json",
      "storyBytes": 18234,
      "mediaBytes": 2015945
    }
  ]
}
```

`storyBytes` and `mediaBytes` let the app show download sizes before the download starts. `mediaBytes` counts the first file of each asset.

Hosts SHOULD serve all files with revalidation (`Cache-Control: no-cache` plus `ETag`), so that updates are seen and unchanged files cost only a `304` response.

### 9.2 Install

1. Read `catalog.json` and show tales with their download sizes.
2. Fetch `tale.json` and check `format` (section 12).
3. Download the story file and verify its `sha256`. On a mismatch, discard the file, report the error, and do not install.
4. The tale is now playable.
5. Download media in the background and verify each file. Retry failed files later. Until a file is verified, section 8 "missing media" applies.

### 9.3 Update

1. A `version` in the catalog that differs from the installed one starts an update. Fetch the new `tale.json`.
2. Download only files whose path is new or whose `sha256` changed.
3. Keep the old story file until the new one is verified. A tale is never half-updated.
4. After the new story file is verified, delete files that the new manifest no longer lists.

### 9.4 Single-file form (`.tale`)

A tale can travel as one file, for example to share it or to import it into the app.

- The file is a ZIP archive of the distribution form of one package. Its name SHOULD be `<id>.tale`. The engine MUST NOT rely on the file name.
- The archive MUST hold exactly these entries, at their package paths: `tale.json`, the story file, and every file in the `media` index. It MUST NOT hold folder entries, a folder that wraps the package, or any other file.
- Entry names MUST be unique.
- Every entry MUST be stored without compression (ZIP method 0) and MUST NOT be encrypted. Media is already compressed, and stored entries let the engine play files straight from the archive.
- An archive over 4 GB, or an entry over 4 GB, uses ZIP64. Plain ZIP sizes and offsets stop at 4 GB.
- `tale.json` is the distribution manifest (6.2), unchanged. Its `sha256` values cover every other entry.
- Hosts SHOULD serve `.tale` files as `application/octet-stream`, so that browsers do not unpack them as ZIP archives.

### 9.5 Import

To import a `.tale` file, the engine:

1. Checks the archive against 9.4, the manifest against sections 6 and 12, and all paths against section 11.
2. Verifies `bytes` and `sha256` of every file that `tale.json` lists.
3. Refuses a package whose ID is the ID of a tale in its catalog. If an imported tale with the same ID exists, the engine MUST ask the player before it replaces that tale, and SHOULD show both versions.
4. Installs the whole package or nothing. On any failure, it reports the reason and keeps the installed tales as they were.
5. Keeps one copy per tale ID: a replaced tale's old copy is deleted. Packages can be large, so the engine SHOULD show how much space each imported tale takes, and MUST let the player remove an imported tale.

## 10. Build and validation

The build:

1. Validates the source `tale.json` (6.1).
2. Compiles `story/main.ink` to `story/main.ink.json`.
3. Scans `media/` and generates the `media` index with sizes, hashes, and dimensions or durations where it can read them.
4. Checks all tags against the media index.
5. Writes the distribution form to `dist/tales/<id>/` and updates `dist/tales/catalog.json`.
6. Writes the single-file form to `dist/tales/<id>.tale` (9.4).

The build MUST fail on:

- a missing or invalid required manifest field, an unknown field, or `story`/`media` in the source manifest,
- a missing `version` when the package folder is not in a git repository with a commit,
- a package ID that does not match the folder name,
- an Ink compile error, or an `EXTERNAL` declaration,
- a tag that refers to an asset ID that does not exist, or to an asset of the wrong type,
- an `input` tag whose variable the story does not declare with `VAR`, whose `VAR` does not start as text, or with a word other than `keypad` after the variable,
- an `img` tag with a word other than a frame after the asset ID (7),
- `cover` or a `credits` entry that refers to an asset ID that does not exist,
- a `theme` colour that is missing or not `"#rrggbb"`, an unknown field in `theme`, or a theme `picture` that is not an `img` asset,
- an invalid or duplicate asset ID, or the ID `none` or `default`,
- an `img` asset without an `.avif` file,
- a file type not allowed by 5.2, a subfolder inside `media/<type>/`, or any file outside the allowed layout (section 4).

For the project form (4.1), the build MUST also fail on:

- anything in the project root other than 4.1 allows, a `media/` folder next to `assets.json`, or a missing `assets/`,
- `assets.json` that is not a JSON object, an invalid asset ID or the ID `none` or `default`, or an unknown field in an entry,
- a `from` that is missing, absolute, has a `..` segment, uses `\`, or names no file in `assets/`,
- a master extension that 4.1 does not list, a setting on the wrong asset type, or a setting with a value it does not allow,
- a lock that is missing, lacks an asset, has another `from` or master `sha256`, or has an asset that `assets.json` does not,
- a render tool that is not installed, or a render that fails.

The build SHOULD warn on:

- media that no tag, `cover`, `credits` entry or theme `picture` refers to,
- files over the size guideline in 5.2,
- tags with unknown keys.

## 11. Security

- Packages are data. They contain no code the engine runs, apart from Ink, which runs in the inkjs runtime without `EXTERNAL` functions.
- Story text is never set as raw HTML. Allowed inline formatting tags (`b`, `i`, `em`, `strong`) are parsed through a strict allowlist into elements that hold text nodes.
- SVG is not allowed.
- All paths are checked: relative, inside the package, no `..`.
- Every downloaded or imported file is verified against its `sha256` before use.
- A `.tale` file holds only the files that its `tale.json` lists (9.4).
- A package cannot refer to core assets or to anything outside itself.
- Theme colours go into the page only as `"#rrggbb"` values. The engine checks each value again before it uses it. A theme cannot bring CSS, fonts or addresses.

## 12. Spec versioning

- `format` holds the spec version as `"major.minor"`.
- While the major version is 0, any minor version may break compatibility. The engine MUST reject a package whose `format` it does not support exactly.
- From 1.0 on: the engine MUST reject a package with an unknown major version. It MUST play a package with the same major and a higher minor version, ignoring unknown tags and manifest fields.
- Every spec change is recorded in the changelog below.

## 13. Open questions

1. **Text-only play.** Resolved: media is optional, both in the package (section 4) and at runtime (section 8).
2. **Single-file sharing.** Resolved: the `.tale` file (9.4) and its import (9.5). `npm run pack-tale` makes a `.tale` file from a source folder (Appendix A); the ink is compiled there, never in the engine.
3. **Translations.** v0.1 is one language per package. Alternatives: one package per language, or a translations slot.
4. **Character tiles.** `img` for now, or a dedicated `portrait:` tag tied to the speaker?
5. **Orientation variants.** The core backgrounds have desktop and mobile versions. Should a package `bg` support variants that the engine picks by screen orientation?
6. **Saves across updates.** An Ink save may not load after the story changes. Keep the old story version until the player finishes, or migrate saves?
7. **Size limits.** Resolved for now (2026-10-26): a sound may have 10 MB and a video 200 MB, and a tale's media over 1 GB warn (5.2). They start small and can grow with real content. First data point: a 6.4-minute music track is 6.1 MB as 128 kbps AAC.
8. **A tale's own keypad.** `input: <variable> keypad` shows the app's keypad picture (7). A tale cannot bring its own keypad picture yet: that needs a key map, each key's position on the picture and what it types or does, and where the map lives is open (`tale.json`, or tags).

## Appendix A: Masters and tooling (non-normative)

Where things are kept:

| Thing | Location | In git? |
| --- | --- | --- |
| Ink, `tale.json`, `assets.json`, `assets.lock.json` | tale project folder | yes |
| Masters (PNG, WAV, MP3, MOV) | any folder, mounted at `<tale>/assets` | no |
| Rendered media, staging, the tale editor's hand-offs | `<tale>/.build/` | no |
| `.tale` files | `dist/` | no |
| App core assets | app `src/assets/`, small, clean licences | yes |
| Engine test tale `the-intercept` | app `stories/` | yes |
| Tales built into the app | app `inst/<id>.tale`, the built version only; their development happens in the tale's own repo | yes |

Tools:

- `tools/import-assets.sh` converts image masters to AVIF and WebP. Audio and video use `ffmpeg` (Homebrew: `/opt/homebrew/bin/ffmpeg`).
- The build tool (section 10) runs in `npm run build` and in the dev server, for the source form in `stories/` and the `.tale` files in `inst/`. It checks each `inst/` file like an import (9.5 steps 1 and 2) and ships it as it is.
- `npm run pack-tale -- <folder>` builds one tale folder with the same checks and packs it into `<id>.tale` (9.4), for import (9.5). A project-form folder (4.1) has its masters rendered first. It writes to the folder it is run from, or to a folder given as a second argument. `*.tale` is gitignored, as a packed tale may be unpublished. The exception is in the app repo: `inst/<id>.tale` is committed. Before it writes the file, it plays the tale through the engine 50 times with seeded random choices, most runs with one save and restore. A choice that asks for text (`input`) gets one of the strings in the story's ink logic, such as the `"1984"` a variable is compared with, or the `""` it starts as, so the branches behind right and wrong answers both get played. An ink error, a turn that runs on without end (8), no ending within 1000 turns, a media tag that names a missing asset or the wrong type, or a save that restores differently stops it with the seed and the problem, and no file is written. Then it plays the chain tests in the tale's `tests/`. Each is a transcript of one path, which must come out the same. A difference stops it with the first line that differs. `--update-chains` rewrites chains that differ instead. `docs/building-tales.md` describes the format.
- `npm run editor` (`editor/cli/`) runs the tale editor's checks and its map on a tale folder, and passes the tale to the editor and back as a project file (`docs/otproject-spec.md`). It keeps its hand-offs in `.build/editor/` in the tale folder (`docs/editor.md`).

## Changelog

- 0.1 (2026-10-18): first draft.
- 0.1 (2026-10-18): add the single-file form, `.tale` (9.4), and import (9.5). `tale.json` does not change, so `format` stays `"0.1"`.
- 0.1 (2026-10-18): allow `LicenseRef-<name>` licences (6.1). Import keeps one copy per tale, shows its size and lets the player remove it (9.5).
- 0.1 (2026-10-18): `version` is optional in the source form. Without it, the build writes `<commit date>-<short hash>`, plus `-dirty` for uncommitted changes (6.2). An update starts on any different `version`, as the two forms do not compare (9.3). `format` stays `"0.1"`.
- 0.1 (2026-10-18): the engine removes standard HTML tags from story text, and an app may show `Name: line` as speech (5.1). `format` stays `"0.1"`.
- 0.1 (2026-10-18): a `music` tag may end with `cut` or a fade time such as `8s`; without it, the engine crossfades (7). New tag `wait: <n>s` holds back the rest of a passage (7). Clearer rules for when tags run and for `sfx` against `music`, with a second example (7). Sound effects end with the tale, and a loaded save replays its passage (8). Saves store every tag effect that lasts beyond its passage, not only `bg` and `music` (7). `format` stays `"0.1"`.
- 0.1 (2026-10-19): in the first passage after a game starts or a save loads, music changes without a time cut (7). `format` stays `"0.1"`.
- 0.1 (2026-10-19): new tag `input: <variable>`. On a choice, inside its brackets, it asks the player for text and puts it into an ink variable (7). The build checks the variable (10), and pack-tale types the story's own strings (Appendix A). `format` stays `"0.1"`: an engine without `input` shows an ordinary choice.
- 0.1 (2026-10-19): `input: <variable> keypad` shows the app's keypad picture in the text window (7). `format` stays `"0.1"`.
- 0.1 (2026-10-20): an `img` tag may end with a frame: `plain`, `line`, `mat`, `double`, `glow` or `print`. Without one, the engine uses its default (7). The build fails on any other word (10). `format` stays `"0.1"`: an engine without frames ignores an `img` tag that names one.
- 0.1 (2026-10-20): add the project form (4.1): `assets.json` names a master in a mounted `assets/` folder for each asset, and `assets.lock.json` pins the masters by hash. `pack-tale` renders the masters, stages the source form and builds it, and `--update-lock` accepts changed masters. The build fails on the project-form problems in section 10. Sound effects are decoded into memory, so keep them short (5.2). Appendix A lists where things are kept. `format` stays `"0.1"`: the distribution form does not change.
- 0.1 (2026-10-20): an archive over 4 GB, or an entry over 4 GB, uses ZIP64 (9.4). `format` stays `"0.1"`: a smaller archive is plain ZIP, as before.
- 0.1 (2026-10-22): the engine keeps `<b>`, `<i>`, `<em>` and `<strong>` as inline formatting and renders them as styled elements (5.1, 11). All other HTML tags are still stripped. `format` stays `"0.1"`.
- 0.1 (2026-10-22): the engine shows an `img` picture at half its pixel size, as a 2x picture (7). A picture at least twice as wide as the story text looks the same as before. `format` stays `"0.1"`.
- 0.1 (2026-10-22): a tale folder MAY have `tests/` with chain tests (4, 4.1). The build never packs it. `format` stays `"0.1"`: the package does not change.
- 0.1 (2026-10-23): the text window of an `input` choice also has Cancel, which takes no turn, as closing the window does (7). `format` stays `"0.1"`: tales do not change.
- 0.1 (2026-10-23): a turn that gives no choice or ending within 10,000 lines, or a line that takes more than a second, stops the story with an error, and the build's playtest reports it (8, Appendix A). Before, such a story froze the page and the build. `format` stays `"0.1"`: tales that play today do not change.
- 0.1 (2026-10-23): names that start with `.` are ignored in a source-form folder too, as the build already did (4). Appendix A lists `npm run editor`. `format` stays `"0.1"`: the package does not change.
- 0.1 (2026-10-25): new optional field `theme`: six colours, and a picture, that the engine wears while the tale plays (6.1). Without a `bg`, the default background is the theme's picture or colour (8). The build checks the theme (10), and the engine checks its colours again (11). `format` stays `"0.1"`, as the spec is a draft: an engine without themes refuses a tale with `theme`, as an unknown field.
- 0.1 (2026-10-25): new tag `map: <id>`, and `map: none`. While a map is set, a button over the story opens it, to zoom and move (7). Saves store the map (7, 8). `format` stays `"0.1"`: an engine without maps ignores the tag, and a save without a map loads with none.
- 0.1 (2026-10-26): a sound may have at most 10 MB and a video at most 200 MB, and the build fails on a bigger one. A tale's media over 1 GB warn (5.2). `format` stays `"0.1"`: the package does not change.
- 0.1 (2026-10-26): `constructor` is a reserved asset ID too (3). `format` stays `"0.1"`.
- 0.1 (2026-11-01): fix: on a phone, switching apps could make the sound stutter and squeak on the way back. The engine now pauses music, sounds and video while the page is hidden, and the sound returns from silence. `format` stays `"0.1"`: tales do not change.
