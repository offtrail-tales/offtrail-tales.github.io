# Project File Format (`.otproject`)

Spec version 0.1 (draft), 2026-10-23.

## 1. Purpose and scope

A project file (`.otproject`) holds the source of one tale: its `tale.json`, its ink files and its media files. The tale editor ([editor.md](editor.md)) writes it with Save and reads it back with Open, the buttons under `.otproject` in its bar, so that an author can go on editing later, in another browser or on another computer. Unzipped, it is a tale folder that `npm run pack-tale` builds.

Players get a tale as a `.tale` file instead ([story package spec](story-package-spec.md), 9.4). A `.tale` holds the compiled story and no ink text, so it cannot be edited. The project file is the tale's source.

This document defines:

- the file name, extension and media type (4),
- the ZIP container and its entries (5),
- the contents: `tale.json`, the story files and the media files (6),
- what the editor writes (7) and reads (8),
- how pack-tale builds an unzipped project file (9),
- security (10) and versioning (11).

It describes the format as the editor writes and reads it today. "The editor writes" and "Open does" describe that implementation. Sentences with MUST, MUST NOT, SHOULD, SHOULD NOT or MAY, used as described in RFC 2119, are rules for any writer or reader.

The story package spec defines the tale itself. This document calls it "the package spec" and cites its sections by number, such as "package spec 6.1".

## 2. Terms

| Term | Meaning |
| --- | --- |
| Project file | An `.otproject` file. |
| Tale, package ID, asset, asset ID | As in package spec 2 and 3. |
| Source form | A tale as its authors keep it: `tale.json`, `story/` and `media/` (package spec 4). A project file holds this form. |
| Project form | The package spec's name for a tale folder that makes its media from masters, with `assets.json` (package spec 4.1). A project file does not hold this form (3.1). |
| Distribution form | A built tale: `tale.json` with the generated fields, the compiled story and the media (package spec 6.2). A `.tale` file holds this form. |
| Tale root | The folder in the archive that holds `tale.json`. |
| Writer, reader | A program that makes a project file (such as Save), or one that opens it (such as Open). |

## 3. Forms

| File | Holds | Made by | Used by |
| --- | --- | --- | --- |
| `.otproject` | the source form (package spec 4) | the editor's Save, or `zip` (9.2) | the editor's Open, and pack-tale once unzipped (9.1) |
| `.tale` | the distribution form (package spec 9.4) | the editor's Publish .tale, or pack-tale | the app's Import tale |

Publish and pack-tale turn the source form into a `.tale`. Nothing turns a `.tale` back into a project file, as it holds no ink text. Open refuses a `.tale` and says to open the tale's project file instead (8.1).

### 3.1 Not the project form

The names clash. The package spec's "project form" (4.1) is a tale folder with `assets.json`, `assets.lock.json` and a mounted `assets/` folder of masters, from which pack-tale renders the media. A project file does not hold that form. It holds the source form: the media files themselves, already in the formats of package spec 5.2.

- A writer MUST NOT put `assets.json`, `assets.lock.json`, `assets/` or masters into a project file.
- A reader MUST refuse a project file with `assets.json` in the tale root. Open refuses it: "This tale makes its media from masters (assets.json). The editor needs the media itself: pack the tale, or unzip a source-form copy of it."

A project-form tale becomes a project file through its staged source form (9.2). It comes back with `npm run editor -- open`, which takes the editor's media as new masters (9.3).

## 4. File name and media type

- The file name ends in `.otproject`. The editor names a download by the tale's title and the local time, `<title>.<YYYY-MM-DD-HHMMSS>.otproject`, with `_` for spaces and without the signs that a Mac or Windows refuses in a name, so that each save keeps the ones before. `npm run editor save` names its file `<id>.otproject`. A reader MUST take the ID from `tale.json`, not from the file name.
- The extension `.otproject` is the editor's own, like the app's `.otsave` for saved games.
- The editor gives the file the media type `application/octet-stream`. A host that serves project files SHOULD use the same type, so that browsers do not unpack them as ZIP archives.

The extension is not `.zip` because Safari unzips a downloaded `.zip` by itself, with its setting "Open 'safe' files after downloading", which is on by default. The author would then have a folder, and Open takes a file, not a folder. Safari keeps a downloaded `.otproject` as it is.

Open still takes a `.zip` (8.1), so a ZIP of a tale folder that another tool made opens too.

## 5. Container

### 5.1 Layout

A project file is a ZIP archive. Every entry sits in one folder named by the package ID:

```text
<id>/
├── tale.json          # the source tale.json (6.1)
├── story/
│   ├── main.ink       # entry point, required
│   └── ...            # every other story file, subfolders allowed (6.2)
├── media/             # only when the tale has media (6.3)
│   ├── img/           # .avif, and optionally .webp with the same ID
│   ├── audio/         # .m4a
│   └── video/         # .mp4
└── tests/             # only when the tale has chain tests (6.4)
```

- A writer MUST put every entry in one top folder, named exactly as the `id` in `tale.json` (package spec 3). Unzipped, that folder is the tale folder (9.1).
- `<id>/tale.json` and `<id>/story/main.ink` MUST be there.
- The archive MUST NOT hold anything that the source form does not allow (package spec 4).

The entries that the editor writes for a tale with two story files, a picture with a WebP fallback, a sound, a video and a chain test, in its order:

```text
the-gate/tale.json                 deflated
the-gate/story/main.ink            deflated
the-gate/story/acts/two.ink        deflated
the-gate/media/audio/birds.m4a     stored
the-gate/media/img/gate.avif       stored
the-gate/media/img/gate.webp       stored
the-gate/media/video/intro.mp4     stored
the-gate/tests/core.chain          deflated
```

### 5.2 Entries

| Property | Rule | The editor writes |
| --- | --- | --- |
| Names | UTF-8, with `/` between folders, and unique. A name MUST NOT start with `/`, contain `\`, or have a segment that is empty, `.` or `..`. Names are lowercase (package spec 3). | The paths of 5.1. The UTF-8 flag (bit 11) is set on a name with a character outside ASCII. |
| Folder entries | Writers SHOULD NOT write them. Readers MUST ignore them. | None. |
| Dotfiles | Writers SHOULD NOT write them. Readers MUST ignore every entry with a name segment that starts with `.`, as the build ignores dotfiles. | None. |
| Compression | Each entry MUST be stored (method 0) or deflated (method 8), and MUST NOT be encrypted. | Text (`tale.json`, the `.ink` files and the `.chain` files) deflated at level 6, even where this makes an entry bigger. Media stored, as it is compressed already. |
| Date | Readers MUST NOT rely on it. | 1980-01-02 00:00:00 on every entry, as a DOS date only, so an unchanged project saves to the same bytes. |
| Order | Readers MUST NOT rely on it. | `tale.json`, then the story files (`main.ink` first, then in the order of the editor's tabs), then the media files: assets by ID, and each asset's files best first (AVIF before WebP). Then the chain tests, by name. |
| Other fields | | Version 2.0, MS-DOS, as made by and as needed. No file mode, no extra fields, no comments, no data descriptors. ZIP64 records only past the limits of 5.3, with version 4.5. |

### 5.3 Size

Save writes the text from memory, and each media file straight from where the browser keeps it. It reads a media file once, in parts, for its CRC. So memory use stays small for a project of any size.

The editor writes ZIP64 records (APPNOTE 4.5) only where a plain ZIP cannot hold a value:

- An entry of 0xffffffff bytes (4 GiB) or more gets a ZIP64 field with both sizes in its local header.
- An entry with that size, or with its local header at an offset of 0xffffffff or more, gets a ZIP64 field with all three values in its central record. The record then holds 0xffffffff for each.
- An archive with 65535 entries or more, or a central directory that is that big or starts that far, gets a ZIP64 end record and locator.

A smaller project has none, so it saves to the same bytes as before ZIP64 came in. Readers MUST read ZIP64. Open does (8.2).

## 6. Contents

### 6.1 `tale.json`

`<id>/tale.json` is the source manifest of package spec 6.1, with the hand-written fields only. It MUST follow package spec 6.1: no `story` or `media` field, no unknown field, and a `version`, if there is one, in SemVer.

The editor writes it from the tale's details:

| Field | The editor writes |
| --- | --- |
| `format` | `"0.1"`, the package spec version (11). |
| `id` | The ID in the details. If that is empty, the ID comes from the title: lowercase ASCII words joined by `-`, or `my-tale` if the title gives none. |
| `version` | The version in the details, but only a SemVer one, such as `"1.0.0"`. Otherwise the field is left out. |
| `title`, `language` | As in the details, trimmed. |
| `authors` | As in the details, each trimmed, empty names left out. |
| `description`, `cover`, `license` | Trimmed, and left out when empty. `cover` is the asset ID of a picture. |
| `credits` | The credits in the author's order, each listing only assets the tale has. An entry left without assets is left out, and so is an empty list. |
| `theme` | The tale's theme, last, with its fields in the order of package spec 6.1, whatever order they came in. Left out when the tale has none. |

```json
{
  "format": "0.1",
  "id": "the-gate",
  "version": "1.0.0",
  "title": "The Gate",
  "language": "en",
  "authors": [
    "A. Author"
  ],
  "cover": "gate",
  "credits": [
    {
      "assets": [
        "birds"
      ],
      "by": "A. Bird",
      "license": "CC0-1.0"
    }
  ]
}
```

- The text is UTF-8 JSON with no byte order mark: the fields in the order above, an indent of 2 spaces, and a newline at the end. Readers MUST NOT rely on the order or the spacing.
- The editor never writes a version that pack-tale made from git (package spec 6.2), such as `"2026-10-22-3a1088c"`. Only a built tale holds one. The worked example has one, from the app's `no-exp.tale` (editor.md, "The worked example"). After Save and Open, Publish therefore makes a new version from the date and a hash of the tale (editor.md, "Publish").
- The editor does not check `tale.json` before it writes it, as a project may be a draft. A tale whose details were never filled in has no authors, so the editor writes `"authors": []`. That breaks package spec 6.1: pack-tale refuses the unzipped folder, and Publish asks for the details first. Open takes the file, and its notes say what the details must fix (8.3).

### 6.2 Story files

- Each story file is `<id>/story/<path>`, where `<path>` is the path that `INCLUDE` lines use (package spec 5.1). `INCLUDE acts/two.ink` names `story/acts/two.ink`.
- `story/main.ink` is the entry point. It MUST be there.
- `story/` MUST hold only `.ink` files (package spec 4).
- The editor writes every story file it holds, as UTF-8 with no byte order mark. That includes a file that no `INCLUDE` names. Such a file is not part of the story: the build ignores it, and the editor's checker reports it.
- The story may have errors. Save does not check the ink, as a project is work in progress. Publish and pack-tale refuse a story with errors.
- The editor names a new story file with `a` to `z`, digits, `_` and `-` only. A file that came in through Open keeps its name (8.4).

### 6.3 Media files

Each media file is `<id>/media/<type>/<asset ID>.<extension>`, in a format of package spec 5.2:

| Folder | Files |
| --- | --- |
| `img/` | `.avif`, and optionally a `.webp` fallback with the same asset ID |
| `audio/` | `.m4a` (AAC) |
| `video/` | `.mp4` (H.264 and AAC) |

- The asset ID is the file name without its extension (package spec 3). Story tags name the file by it.
- Every `img` asset MUST have an `.avif` file, and asset IDs MUST be unique across the three folders (package spec 3 and 5.2).
- The bytes are the file as the author added it, or as the editor converted it (editor.md, "Media"). They are the bytes that Publish puts into the `.tale`.
- The editor gives a file its asset ID from the name it was added under: lowercase, accents removed, each run of other characters made one `-`, at most 64 characters. `Forest Clearing.avif` becomes `forest-clearing`. `none` and `default` get `media-` in front, and a name with no letter or digit becomes `media-file`. If a file of another type has the ID already, the new file gets a number: `forest-2`.
- Credits, the maker and licence of a file, are in `tale.json` (6.1).

A project file holds no compiled story and no media index. The build makes them (package spec 6.2).

### 6.4 Chain tests

The source form MAY hold chain tests in `tests/` (package spec 4, and docs/building-tales.md, "Chain tests"): `.chain` files, without subfolders. Each is `<id>/tests/<name>.chain`.

- Open keeps each chain test as text, by its name (8.4). Save writes them back as they came. The editor does not show, change or run them; pack-tale plays them. So a tale that goes through the editor keeps its chain tests.
- A story changed in the editor may no longer pass a chain test. The editor does not tell, as Publish plays no chains. pack-tale does.

## 7. Writing

Save:

1. takes the tale from this browser: the details, the story files, and each media file by its SHA-256 (editor.md, "Where the work is kept"),
2. makes `tale.json` from the details (6.1),
3. writes the archive (5), with each media file straight from the browser's storage, and downloads it as `<title>.<YYYY-MM-DD-HHMMSS>.otproject` (4).

- If the bytes of a media file are missing from this browser, Save stops with "`<path>` is missing from this browser. Add the file again." It downloads nothing.
- It checks neither the ink (6.2) nor `tale.json` (6.1).

## 8. Reading

### 8.1 Which files Open takes

Open, and a single file dropped on the page that is not media, go by the end of the file name, in any case:

| Name ends in | Open |
| --- | --- |
| `.tale` | refuses it, as a published tale holds no ink text, and says to open the tale's project file instead |
| `.otproject` or `.zip` | reads it as a project file (8.2 to 8.5) |
| `.ink` | takes it as the whole story, in `main.ink`; the details and the media stay |
| anything else | asks whether to open it as ink |

Before it replaces the tale in the editor, Open asks the author.

### 8.2 Finding the tale

Open reads the archive in place: a stored entry is a part of the file, and a deflated entry is unpacked when Open reads it. Before it reads any entry, it adds up the sizes that the entries declare. It refuses the file when the deflated entries would unpack to more than 2 GiB, or more than 50 times the size of the file plus 16 MiB, or when the stored entries come to more than the file itself (section 10). Then it:

1. ignores folder entries, and every entry with a name segment that starts with `.`: `.DS_Store`, `.build/`, the `._` files in the `__MACOSX/` folder that Finder's Compress adds, and any `..` segment,
2. takes the shortest remaining name that is exactly `tale.json`, at the top or after a `/`; what comes before it is the tale root, which is empty or one or more folders,
3. leaves out every entry outside the tale root, and its notes say how many.

So Open takes a tale with or without a folder around it. The folder name does not have to equal the ID: Open takes the ID from `tale.json`.

- A reader MUST take the tale root from a file named exactly `tale.json`. A name such as `mytale.json` is not one.
- An archive SHOULD hold one tale. With two at the same depth, Open takes the first in the archive and leaves out the other, as files outside the tale root.

### 8.3 Checks

Open refuses the file when one of these checks fails, in this order. It shows `"<file name>" could not be opened.` and the message:

| Problem | Message |
| --- | --- |
| Not a ZIP archive; an entry that is encrypted, has a method other than 0 or 8, or is stored with two sizes; or an entry that does not unpack to its size | This is not a ZIP file. |
| The compressed entries would unpack to more than 2 GiB (8.2) | Its compressed files would unpack to more than 2 GB, more than the editor opens. Zip the tale folder with its media stored as they are: zip -r -n .avif:.webp:.m4a:.mp4 \<file> \<folder>. |
| The entries would unpack too far (8.2) | This file would unpack to far more than its own size, so the editor does not open it. |
| No `tale.json` | The ZIP holds no tale.json, so it is not a tale's source. |
| `assets.json` in the tale root (3.1) | This tale makes its media from masters (assets.json). The editor needs the media itself: pack the tale, or unzip a source-form copy of it. |
| `tale.json` is not JSON | tale.json is not valid JSON. |
| `tale.json` is not a JSON object | tale.json is not a JSON object. |
| `tale.json` has `story` or `media`, as a built tale's does | This tale.json lists a built story and media: it comes from a .tale, which holds no ink text. Open the tale's project instead. |
| `format` is not `"0.1"` | tale.json is for format …, which this editor cannot read (it reads "0.1"). |
| No `story/main.ink` in the tale root | The ZIP has no story/main.ink. |

A `tale.json` that passes these checks but breaks other rules of package spec 6.1 opens: a project may be a draft, such as a new tale with no author yet. Open runs the build's own check for the source form (`checkManifest`), and its notes list each problem, which the details must fix before Publish. A field of the wrong kind, or one the editor does not know, is left out and named in the notes. So is a `theme` with any fault, whole: half a theme is no theme, and the editor's Theme tab can set a new one. A version that pack-tale made, which a source `tale.json` may not hold, opens as the tale's version.

- A reader MUST refuse a `tale.json` that is not a JSON object, that has `story` or `media`, or whose `format` it does not support (package spec 12).
- A reader MAY open a `tale.json` that breaks other rules of package spec 6.1, as a draft. It MUST then tell the user each problem, and a writer of a `.tale` MUST NOT pass the problems on (package spec 10).
- A reader MUST refuse a file without `story/main.ink`.

### 8.4 What Open keeps

Open goes through the other entries in the tale root, by their path there:

| Entry | Open |
| --- | --- |
| `story/<path>` ending in `.ink`, at any depth | keeps it as a story file, by its path under `story/`, in lowercase |
| `media/<type>/<file>`, where `<type>` is `img`, `audio` or `video`, and `<file>` ends in a lowercase extension of 6.3 for that type | adds it as a media file, as the Media tab does (6.3) |
| `tests/<name>.chain`, with no `/`, `\` or `:` in `<name>` | keeps it as a chain test, by its name (6.4) |
| anything else in `tests/` | leaves it out, with a note: "`<path>` was left out: tests/ holds only .chain files, without subfolders, "\" or ":"." The build refuses such a file (package spec 4), and on Windows `\` and `:` lead out of a folder. |
| anything else | leaves it out, with a note: "`<path>` was left out: a tale's source holds only tale.json, story/, media/ and tests/." |

A tale folder that pack-tale builds passes Open unchanged. Open is lenient in some ways that such a folder never meets:

- It gives a story file whose name is not lowercase, such as `Act1.ink` from an Inky project, its lowercase name, as the build takes only lowercase names (package spec 3) and the editor cannot rename a file. An `INCLUDE` line that names a story file in another case then names it in lowercase, as a Mac finds the file either way. The notes report each rename, and how many `INCLUDE` lines changed. A file whose lowercase name another file has is left out, with a note.
- It gives each media file an asset ID as the Media tab does (6.3), and its notes report each change: "`<path>` is named "`<id>`" in the editor: "`<name>`" names another file or is not a valid asset ID. Tags must use the new name." `Big Tree.avif` becomes `big-tree`, and `none.avif` becomes `media-none`. `forest.avif` beside `forest.m4a` becomes `forest-2`, so a `bg: forest` tag then names a sound until it changes.
- It takes an `img` asset that has only a `.webp` file. Publish refuses it.
- It does not check that `cover`, `credits` and the theme's `picture` name assets that the tale has. Save leaves out credits for missing assets (6.1). Publish refuses a cover or a theme picture that is not a picture.
- It does not check that the bytes of a media file match its extension. The build does not check this either.
- Of two entries with the same name, it takes the last.
- It decodes text as UTF-8. A byte that is not UTF-8 becomes U+FFFD, and a byte order mark is dropped.

A reader SHOULD tell the user about each entry that it leaves out or changes. Open does so for the entries it leaves out and for the media and story files it renames. It does not report the other leniencies above: none of them gives a tale that the build refuses, as Publish refuses what the build refuses (section 12, question 7).

### 8.5 After Open

Open replaces the tale in this browser. The details come from `tale.json`, with the version as it is there. The story files and the media files come from the archive, and Open stores each media file by its SHA-256 (editor.md, "Where the work is kept"). The story files keep the order in which the archive lists them, after `main.ink`. A file that the editor wrote lists them in the order of its tabs (5.2), so the tabs come back as they were. The editor then works from the stored copies, so the opened file may move or go. If the browser cannot keep the media files, as in a private window, they last until the page closes, and only while the opened file stays as it is. The editor says so. Then the editor shows the notes, if there are any.

## 9. Building with pack-tale

### 9.1 From project file to `.tale`

```text
unzip the-gate.otproject
npm run pack-tale -- the-gate
```

- `unzip` makes the folder `the-gate/`, in the source form. pack-tale builds it as any tale folder (package spec 10 and Appendix A) and writes `the-gate.tale`.
- `tale.json` needs a SemVer `version`, or the folder has to sit in a git repository with a commit (package spec 6.2). As long as the folder is not committed, that version ends in `-dirty`. With neither, pack-tale stops: `tale.json has no "version", and the tale is not in a git repository with a commit to take one from.`
- With a SemVer version, Open and Publish in the editor make the same `.tale` as pack-tale on the unzipped folder, byte for byte.
- The unzipped files carry the date of the entries, 1980-01-02.

Names outside ASCII need care. The editor sets the UTF-8 flag on them, but the `unzip` of macOS (UnZip 6.00) ignores the flag and garbles the name. `ditto -x -k the-gate.otproject .` keeps the name. `tar -xf` keeps it too, but in decomposed form (NFD), and the build then cannot find a file that an `INCLUDE` line names in composed form. The names that the editor makes itself are ASCII (6.2, 6.3).

`npm run editor -- open <project file> [output folder]` does the same without these pitfalls: it reads the file as Open does, and writes the names as they are. It writes only the files the editor changed, `tests/` included, and works for a project-form folder too (9.3, and editor.md, "From the terminal").

### 9.2 From a tale folder to a project file

```text
zip -r -n .avif:.webp:.m4a:.mp4 the-gate.otproject the-gate
```

- `-n` stores the media as they are: they are compressed already, and Open unpacks at most 2 GB of compressed files (8.2).

- This works for a tale folder in the source form. `zip` adds folder entries, and any `.DS_Store` file. Open ignores both (8.2).
- A project-form tale (package spec 4.1) has no `media/`. After a pack-tale run, `.build/stage/<id>/` in the tale folder holds its source form (package spec 4.1, build step 5). Zip that folder in the same way.

`npm run editor -- save <tale folder> [output folder]` writes the file that Save would write for the tale, byte for byte, without folder entries or dotfiles. For a project-form tale, it makes the media from the masters first, as pack-tale does, and takes `tests/` from the tale folder.

### 9.3 Back into a project-form tale

A project-form tale has masters, not media files (package spec 4.1). When the editor changed its media, `npm run editor -- open` takes them as new masters:

- Each asset with a media file that the editor added or changed gets a new master: the asset's best file (AVIF before WebP), written as `assets/<img|audio|video>/<asset ID>.<extension>`. Its `assets.json` entry then names that file, without render settings. pack-tale copies such a master as it is, so the tale gets the editor's bytes.
- `open` never writes over or deletes a master. When another file has that name, the master gets a number: `gate-2.avif`. A file there with the same bytes is used as it is.
- An asset that the editor removed leaves `assets.json`. Its master stays.
- From an AVIF master, pack-tale makes the AVIF alone. A WebP fallback beside it is left out, and `open` says so.
- Then `open` updates `assets.lock.json` with the steps of `--update-lock`. Before it writes anything, it checks the masters against the lock, as pack-tale does, so a master that changed elsewhere stops it.
- `assets.json` gets one asset on each line. Commit it and the lock with the tale.

Media changes merge by file, as the other files do (9.1): a media file that changed in the folder and in the editor stops `open`, unless `--force`.

## 10. Security

The safety statements in this section use ASD-STE100 Simplified Technical English.

A project file is data:

- A project file contains only JSON, ink text, chain tests and media files. A chain test is text. It contains no program for the editor.
- Open does not run code from the file. It reads the ZIP archive, reads `tale.json` as JSON, reads the story files and the chain tests as text, and keeps the media files as bytes. The editor does not run the chain tests.
- Open does not write files to your disk. It keeps the project in the storage of the browser.
- Then the editor compiles and plays the ink, as it does for text that you type. Ink runs in the inkjs runtime, without EXTERNAL functions (package spec 11).
- The editor shows all text as text, not as HTML. The game screen keeps only the tags `b`, `i`, `em` and `strong` (package spec 5.1).
- The editor puts media files only into picture, sound and video elements.
- In a build, the security policy of the page stops all scripts that are not the app's own files (docs/security.md). The dev server has no policy.

At worst, a bad project file can do these things:

- It can use too much memory or time. Then the page stops or becomes slow. Open reads a stored entry in place, as a part of the file. It unpacks a deflated entry only when it keeps that entry, and it stops when the entry unpacks to more or less than its declared size. Before it reads an entry, it adds up the sizes that the entries declare. It refuses the file if the deflated entries declare more than 2 GiB, or more than 50 times the size of the file plus 16 MiB. It also refuses the file if the stored entries declare more than the size of the file, because many entries can point to the same bytes. The app's import of a `.tale` file refuses compressed entries. Open takes them, as the editor writes its text compressed.
- It can fill the storage of the site. The app uses the same storage.
- It can replace the tale in the editor. Open asks you before it does this.
- A bad media file can use a defect in a media decoder of the browser. Media files on all web pages have this risk.

A bad project file cannot read or change the files on your computer. It cannot run its own scripts in the page.

Rules:

- Do not use pack-tale to build a project file from another person. The build is not a sandbox. A bad input can run code on your computer (docs/building-tales.md, "Safety"). Open the file in the editor, or ask that person for the `.tale` file.
- A tool that unpacks a project file MUST NOT write outside its target folder. It MUST skip or refuse a name that starts with `/` or has a `..` segment. Open never writes the entries to disk, and it ignores every name with a segment that starts with `.` (8.2).
- `npm run editor -- open` writes the tale of a project file into the folder `<output folder>/<id>/`. The ID comes from `tale.json` in the file. If the ID is not a valid package ID (package spec 3), `open` stops and writes nothing. A valid ID has no `/` and no `.`, so the folder is always in the output folder. `open` writes only `tale.json`, the story files, the media files and the chain tests that Open keeps (8.4). Story files keep their paths under `story/`, without a segment that starts with `.` (8.2). Media files get the names that Open gives them (6.3). For a tale in the project form, `open` writes new master files into the tale's `assets/` folder, in `assets/img/`, `assets/audio/` or `assets/video/`, by asset ID (9.3). It does not write over a file there, and it does not delete a file there. The `assets/` folder can be a link to a folder elsewhere. `open` then writes into that folder.

## 11. Versioning

- A project file has no format field and no version of its own: no marker entry and no ZIP comment.
- Its `tale.json` tells a reader what it holds. `format` is the version of the package spec that the tale follows, `"0.1"` today. A reader MUST refuse a file whose `format` it does not support (package spec 12). Open does this with the build's own check (8.3).
- A change to the source form in the package spec changes what a project file holds. The package spec records each change in its changelog, and its section 12 says when a change breaks old readers.
- So a project file is marked only by its extension, its one top folder and its `tale.json`. A ZIP of any tale folder in the source form opens the same way.
- This document has a version of its own, for its changelog. No file records it.

## 12. Open questions

None is open. The answers stay here, with their reasons.

1. **A format field.** Resolved (2026-10-23): no marker. `tale.json`'s `format` says what a project file holds (section 11), and a ZIP of any tale folder opens the same way, which `zip` and Finder's Compress rely on (9.2). The one piece of editor state that a marker could have held, a version that pack-tale made, does not go into the file (question 4). A marker could not go into the tale folder anyway: the source form allows nothing else there (package spec 4).
2. **Keeping `tests/`.** Resolved (2026-10-23): Open keeps the chain tests and Save writes them back as they came (6.4). The editor does not show or run them.
3. **The project form.** Resolved (2026-10-23): a project file holds no masters (3.1). Masters can be big and live elsewhere, and the editor's conversions need not give the bytes of pack-tale's renders. `npm run editor -- save` makes the media from the masters as pack-tale does, and `open` takes the editor's media back as new masters (9.3).
4. **A version that pack-tale made.** Resolved (2026-10-23): the project file does not keep it. Save leaves it out, as `tale.json` holds SemVer only (6.1). After Save and Open, Publish makes a new version: a reopened project is the author's own tale. The worked example, opened from New, still publishes as `inst/no-exp.tale` byte for byte.
5. **Big projects.** Resolved (2026-10-23): Save writes media straight from the browser's storage and Open reads the archive in place, so memory use stays small. Past 4 GiB, Save writes ZIP64 records (5.3, 8.2).
6. **Names outside ASCII.** Resolved (2026-10-23): no new rule. Package and asset IDs are ASCII (package spec 3), and so is every name that the editor makes (6.2, 6.3). A story file or chain test may keep a lowercase name outside ASCII, as the package spec allows. Such a name never reaches a player: a `.tale` holds the compiled story alone. To unzip such a project by hand, use `ditto`, or `npm run editor -- open`, which writes names as they are (9.1).
7. **Strict reading.** Resolved (2026-10-23): Open stays lenient, so a file that it can read opens and no work is lost. What Open changes or leaves out, its notes report (8.4). What the build refuses and Open cannot change, Publish refuses, as pack-tale does: a draft's details, a cover or theme picture that is not a picture, a picture without AVIF. Story file names with capitals were the one gap, as Publish packs the compiled story and never sees them: Open gives them their lowercase names (8.4).

## Appendix A: Code (non-normative)

| File | Does |
| --- | --- |
| `editor/src/project-file.ts` | `saveProjectFile` (Save), `openProjectFile` (Open), `detailsFrom` (the details from `tale.json`) |
| `editor/src/project-zip.ts` | `writeProjectZip`: the archive as fflate's `zipSync` wrote it, media straight from their Blobs, ZIP64 past the limits of 5.3 |
| `editor/src/publish.ts` | `sourceManifest`: `tale.json` from the details, SemVer versions only |
| `editor/src/media.ts` | `addMedia` and `assetIdFromName`: asset IDs from file names; `creditsFor`: credits for the assets the tale has |
| `editor/src/project.ts` | The `Project` and `Details` types, and the ID from the title |
| The engine's `tale.ts` | `checkManifest`: the `tale.json` rules, shared with the build and the app |
| `editor/src/project-file.test.ts` | Save gives a folder that the build takes; Open reads it back unchanged |
| `editor/cli/` | `npm run editor`: `save` and `open` on the command line (9.1, 9.2) |

## Changelog

- 0.1 (2026-10-23): first draft.
- 0.1 (2026-10-23): Open takes a draft's `tale.json` and lists its problems, finds `tale.json` by its exact name, reports renamed media, `tests/` and files outside the tale root, and refuses a file whose entries declare far more than its size (8.2 to 8.4, 10). Save project stops before it would pass the limits of a plain ZIP (5.3).
- 0.1 (2026-10-23): `npm run editor -- save` and `open` write and read a project file from a tale folder (9.1, 9.2). `open` refuses an ID that is not a valid package ID (10).
- 0.1 (2026-10-23): Open keeps the chain tests in `tests/`, and Save project writes them back after the media (5.1, 5.2, 6.4, 8.4). Question 2 resolved.
- 0.1 (2026-10-23): Save project writes each media file straight from the browser's storage, and ZIP64 records past 4 GiB. Open reads the archive in place, and its size checks count what it unpacks (5.2, 5.3, 7, 8.2, 8.3, 8.5, 10). Question 5 resolved.
- 0.1 (2026-10-23): `npm run editor -- open` takes the editor's media into a project-form tale as new masters, and updates its lock (3.1, 9.3, 10). Question 3 resolved.
- 0.1 (2026-10-23): Open gives story files with capitals their lowercase names, and `INCLUDE` lines follow (8.4). Questions 1, 6 and 7 resolved: no format marker, no ASCII rule, and Publish refuses what the build refuses. No question is open.
- 0.1 (2026-10-25): Open keeps the order in which the archive lists the story files, so the editor's tabs come back in their order (8.5). The editor's Save project is now Save, under `.otproject` in its bar.
- 0.1 (2026-10-25): the editor names a download by the title and the time, not by the ID (4, 7). Open… is Open.
- 0.1 (2026-10-25): `tale.json` may hold a `theme` (package spec 6.1). The editor writes it last, and Open keeps it, or leaves out a theme with a fault, with a note (6.1, 8.3, 8.4).
