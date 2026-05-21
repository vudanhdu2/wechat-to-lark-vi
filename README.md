# wechat-to-lark (v2)

A Claude Code skill that automates translating WeChat articles (mp.weixin.qq.com) to Vietnamese and publishing them as Lark documents — complete with images, embedded videos, callouts, and QA verification.

## What it does

Given a WeChat article URL, the pipeline runs:

```
Phase 1:  EXTRACT  → text + images + videos (via CDP browser + regex)
Phase 2:  MAP      → image-context map + video-context map
Phase 3:  TRANSLATE → Chinese → natural Vietnamese
Phase 4:  CREATE   → Lark doc text-only (v2 API, markdown)
Phase 4a: IMAGES   → fetch block IDs, insert via XML <img> block_insert_after
Phase 4b: VIDEOS   → download .mp4 files, insert via +media-insert preview player
Phase 5:  QA       → cross-check counts, positions, missing content
```

## Key technical solutions

- **WeChat image extraction**: Regex on `innerHTML` (DOM selectors return 0 results due to WeChat's nested sections)
- **WeChat video extraction**: Regex for `data-mpvid` + `<video src="...mpvideo.qpic.cn...">` patterns
- **Image position mapping**: `[[IMG_N]]` markers bridge extraction → translation → block-level insertion
- **Two-pass insertion (CRITICAL)**: `<image>` markdown is silently stripped by lark-cli v2 API. Images must be inserted in a second pass via XML `<img>` + `block_insert_after` against fetched block IDs
- **Video embedding**: Download videos locally, then upload via `+media-insert --file-view preview` for inline player. Files >20MB auto-use multipart upload
- **Shell safety**: All CDP eval calls use heredoc (`--data-binary @- << 'EVALEOF'`)
- **lark-cli v2 API**: All `docs` commands use `--api-version v2` with `--content "@file"` and `--doc-format markdown|xml`

## Prerequisites

- [web-access](https://github.com/eze-is/web-access) skill (CDP browser automation via Chrome)
- [lark-cli](https://www.npmjs.com/package/@larksuite/cli) (`npm i -g @larksuite/cli`) with `lark-doc` skill
- Node.js 22+
- Chrome with remote debugging enabled

## Installation

### Option 1: Clone to project skills

```bash
cd your-project
mkdir -p .claude/skills
git clone https://github.com/abc8888xyz/weixintolark .claude/skills/wechat-to-lark
```

### Option 2: Clone to global skills

```bash
git clone https://github.com/abc8888xyz/weixintolark ~/.claude/skills/wechat-to-lark
```

## Usage

Just provide a WeChat URL and ask Claude to translate:

```
https://mp.weixin.qq.com/s/xxxxx
dịch bài này
```

Claude will automatically:
1. Open the article in your Chrome browser
2. Extract all text and images
3. Translate to natural Vietnamese
4. Create a formatted Lark document with images
5. QA-check the result

## File structure

```
wechat-to-lark/
├── SKILL.md                              # Main pipeline definition (v2.0.0)
├── README.md                             # This file
└── references/
    ├── wechat-extraction.md              # WeChat DOM extraction (text/images/videos)
    ├── translation-guidelines.md         # Vietnamese translation style guide
    ├── lark-formatting.md                # Lark doc formatting + XML image insertion (v2 API)
    ├── video-handling.md                 # Video download + +media-insert workflow
    └── qa-checklist.md                   # QA verification (images + videos + headings)
```

## Changelog

### v2.1.0 (2026-05-21)

- **NEW: Format preservation as hard constraint** — Script 2 now converts `<strong>`, `<b>`, `<span style="font-weight:bold">`, and `<span style="color:cam/đỏ">` to markdown `**bold**` BEFORE stripping HTML
- **NEW: translation-guidelines.md MANDATORY section** — translator must keep same count of `**...**` markers between source and target, with self-check formula
- **NEW: QA Bước 2b** — bold count parity check between source markers and Lark `<b>` tags, with auto-fix path via `block_replace`
- **WHY**: WeChat uses bold + orange/red highlight extensively for key insights. Without preservation, translations on other machines came out flat (no emphasis), breaking quality consistency across installs.

### v2.0.0 (2026-05-21)

- **NEW**: Full video support — extract, download, embed as inline player
- **BREAKING**: Migrated to lark-cli v2 API (`--api-version v2`, `--content "@file"`, `--doc-format markdown`)
- **FIX**: Images now inserted in 2-pass (Phase 4 text-only, then Phase 4a XML `<img>` via `block_insert_after`) — the markdown `<image>` extension is silently stripped in v2
- **FIX**: `/new` CDP endpoint changed to POST (v2.5.3+)
- **FIX**: lark-cli `--file` requires relative paths inside current directory

### v1.0.0

- Initial release: text + image extraction, translation, Lark doc creation

## License

MIT
