# wechat-to-lark

A Claude Code skill that automates translating WeChat articles (mp.weixin.qq.com) to Vietnamese and publishing them as beautifully formatted Lark documents — complete with images, callouts, and QA verification.

## What it does

Given a WeChat article URL, this skill runs a 5-phase pipeline:

```
Phase 1: EXTRACT  → Open URL via CDP browser, scroll, extract text + images
Phase 2: MAP      → Build image-context map (which image goes where)
Phase 3: TRANSLATE → Translate Chinese → natural Vietnamese
Phase 4: CREATE   → Create Lark doc with images via lark-cli
Phase 5: QA       → Cross-check translation vs original
```

## Key technical solutions

- **WeChat image extraction**: Uses regex on `innerHTML` instead of DOM selectors (WeChat's nested sections make `querySelectorAll("img")` return 0 results)
- **Image position mapping**: `[[IMG_N]]` markers bridge extraction → translation → document creation
- **Shell safety**: All CDP eval calls use heredoc (`--data-binary @- << 'EVALEOF'`) to avoid regex escape issues
- **Chunked creation**: Long articles are split at heading boundaries for reliable Lark doc creation

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
├── SKILL.md                              # Main pipeline definition
├── README.md                             # This file
└── references/
    ├── wechat-extraction.md              # WeChat DOM extraction scripts
    ├── translation-guidelines.md         # Vietnamese translation style guide
    ├── lark-formatting.md                # Lark document formatting templates
    └── qa-checklist.md                   # QA verification process
```

## License

MIT
