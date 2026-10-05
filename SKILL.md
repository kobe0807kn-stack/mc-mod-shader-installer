---
name: mc-mod-shader-installer
description: Locate a Minecraft installation automatically and install mods or shader packs into it, with download verification, conflict pre-checks and one-command rollback. Use when a user asks to install / download / add Minecraft mods (模组) or shaders (光影 / shaderpacks), says "装模组", "下个 mod", "装光影", "install a shader", "add this mod to my game", or hands over a Modrinth / CurseForge / mcmod.cn link or a game version number for Minecraft. Also use when a user does not know where their .minecraft folder is.
license: MIT
---

# Minecraft Mod & Shader Installer

A portable skill for any agent. **Single file, no dependencies beyond Python 3.8+ (stdlib only) or equivalent shell tools.**

---

## 0. The contract — read this first

Your job in one sentence:

> **Ask the user exactly two things, find their game folder yourself, then download + verify + install.**

**You ask ONLY these two things:**

| # | Question | Why |
| - | -------- | --- |
| 1 | **Game version number** (e.g. `1.21.1`, `26.2`) | Picks the right build and the right `versions/<x>` folder |
| 2 | **The resource link(s)** — Modrinth / CurseForge / mcmod.cn / direct URL, or just the mod/shader name | What to download |

**Never ask for a file path.** The whole point of this skill is that you find it. The user does not know it, and asking makes you look worse than a search bar.

**Never ask extra questions.** No "which launcher?", no "do you want a backup?", no "should I check for conflicts?". You do those automatically and report what you did. The only permitted follow-up is a **multiple-choice pick** when auto-detection genuinely finds several candidate installations (see §3.4).

---

## 1. Language

Detect the language of the user's first message and reply in it.

- User writes Chinese → answer in **中文**（见 Appendix A 的中文模板）
- Anything else → answer in **English**
- If the user asks for another language, use it. The message templates in §2 and §8 are translated; swap in the user's language as needed.

Keep a single language per conversation. Do not mix.

---

## 2. Step 1 — The two questions

Ask them together, in one message, as a short numbered list. Do not split into two turns.

### English template

```
I need two things and then I'll handle the rest:

1. Your Minecraft version number (e.g. 1.21.1, 26.2) — the one you actually play on.
2. The link(s) or name(s) of the mod(s) / shader pack(s) you want.

I'll find your game folder myself — you don't need to tell me the path.
```

### 中文模板

```
我只需要两样东西，剩下我来办：

1. 你的游戏版本号（比如 1.21.1、26.2）—— 你实际在玩的那个。
2. 你想要的模组 / 光影的链接或名字。

游戏文件夹我自己会找，你不用告诉我路径。
```

If the user already supplied one or both, **do not re-ask**. Just ask for what is missing.

If the user says "just find me a good one" instead of naming a resource, search for the top-downloaded option matching their version and present the top 3 with download counts, then proceed on their pick. That counts as question 2.

---

## 3. Step 2 — Find the game folder yourself

Run the detection script in **Appendix B.1**. It is cross-platform Python, stdlib only, and prints a JSON report. Read it, then reason about the result.

### 3.1 Where Minecraft lives

Scan **all** of these, not just the first hit. A user can have several.

**Windows**

```
%APPDATA%\.minecraft                                   → C:\Users\<u>\AppData\Roaming\.minecraft
%USERPROFILE%\.minecraft
%USERPROFILE%\curseforge\minecraft\Instances\<name>     → CurseForge app
%APPDATA%\ModrinthApp\profiles\<name>                   → Modrinth App
%APPDATA%\PrismLauncher\instances\<name>\minecraft      → Prism Launcher
%APPDATA%\MultiMC\instances\<name>\minecraft            → MultiMC
%APPDATA%\.minecraft\versions\<ver>\PCL                 → PCL2 selected version marker
```

Also: **the folder containing the launcher executable**. Chinese players very often run a portable PCL2 / HMCL / BakaXL from a custom folder, and `.minecraft` sits **next to the .exe**. If the user mentions PCL, HMCL, BakaXL, or 网易版, search for launcher executables first:

```
# find launchers, then look for .minecraft next to them
PCL*.exe, HMCL*.jar, HMCL*.exe, BakaXL*.exe, *.lnk pointing at them
```

Then check `<launcher_dir>\.minecraft` and `<launcher_dir>\minecraft`.

**macOS**

```
~/Library/Application Support/minecraft
~/Library/Application Support/PrismLauncher/instances/<name>/minecraft
~/Library/Application Support/ModrinthApp/profiles/<name>
~/Library/Application Support/CurseForge/Instances/<name>
~/Library/Application Support/minecraft/versions/<ver>/PCL
```

**Linux**

```
~/.minecraft
~/.var/app/com.mojang.Minecraft/.minecraft              → Flatpak
~/.local/share/PrismLauncher/instances/<name>/minecraft
~/.local/share/multimc/instances/<name>/minecraft
~/.local/share/ModrinthApp/profiles/<name>
~/.local/share/curseforge/minecraft/Instances/<name>
```

A directory counts as a Minecraft root if it contains **at least one** of: `versions/`, `mods/`, `shaderpacks/`, `options.txt`, `launcher_profiles.json`.

### 3.2 Pick the version folder

Match the user's version number against `<root>/versions/*`:

1. **Exact match** on the folder name → done.
2. **Normalised match**: strip non-alphanumerics and lowercase, then compare. `26.2-NeoForge_26.2.0.75` normalises to `262neoforge262075`, which contains `262` → match.
3. **Prefix match**: folder name starts with the version, e.g. user says `26.2`, folder is `26.2-NeoForge_26.2.0.75`.
4. **Fuzzy**: folder name contains the version string.
5. Still nothing → list the folders you *did* find and let the user pick. This is allowed; it's a choice, not a path.

### 3.3 Decide whether version isolation is on

**This is the single most common reason a manual install "doesn't work".**

If version isolation is on, `mods/` and `shaderpacks/` live **inside** the version folder, and anything dropped in the root `<root>/mods/` is silently ignored.

| Launcher | How to check |
| -------- | ------------ |
| **PCL2** | Read `versions/<ver>/PCL/Setup.ini`. If it contains `VersionArgumentIndieV2:True` (or `VersionArgumentIndie:1`) → **isolation ON** |
| **HMCL** | Read `<root>/hmcl.json` → `versionIsolation` / per-instance settings; or simply: does `versions/<ver>/mods` exist? |
| **Prism / MultiMC / Modrinth App / CurseForge** | Always instance-scoped. Mods go in the instance's own `minecraft/mods` |
| **Official launcher** | No isolation. `mods/` and `shaderpacks/` at root |
| **Universal fallback** | If `versions/<ver>/mods` **exists**, isolation is effectively on → use it |

### 3.4 Resolve the target folders

```
if isolation is ON:
    MODS_DIR       = <root>/versions/<ver>/mods
    SHADER_DIR     = <root>/versions/<ver>/shaderpacks
else:
    MODS_DIR       = <root>/mods
    SHADER_DIR     = <root>/shaderpacks
```

Create them if missing (`mods/` for mods, `shaderpacks/` for shaders). Creating an empty folder is safe and reversible.

**If more than one candidate installation exists**, show a short numbered list — launcher name, version folder, and which one is most recently used (newest `logs/latest.log` mtime) — and ask the user to pick a number. Then continue. That is the only permitted second question.

### 3.5 Detect the mod loader

You need this to filter downloads. Read `versions/<ver>/<ver>.json` and look for these strings:

| Found in the json | Loader |
| ----------------- | ------ |
| `net.neoforged` | `neoforge` |
| `net.minecraftforge` | `forge` |
| `net.fabricmc` | `fabric` |
| `org.quiltmc` | `quilt` |
| none of the above | `vanilla` — **mods will not work; tell the user they need a mod loader** |

Shaders are loader-agnostic — they work on any loader, but need a shader loader mod (Iris / Oculus) or OptiFine. See §5.4.

---

## 4. Step 3 — Download

### 4.1 Source priority

1. **Modrinth** — always try first. Public API, no key, no ads, no redirects, and it gives you SHA-1 + SHA-512 so you can prove the file is authentic.
2. **CurseForge** — only if Modrinth has no matching build. The API needs a key you probably do not have, so open the file page in a browser and let the user click Download, or ask them to paste the direct CDN URL.
3. **Direct URL** the user supplied — download it, but you have no official hash. Say so explicitly in your report.
4. **mcmod.cn** — a Chinese wiki, **not** a file host. Use it only to map a Chinese name to the real project name, then go to Modrinth / CurseForge.

### 4.2 Modrinth API calls

Send a `User-Agent` header on every request or you get 403. Add ~0.7 s between calls and retry on 429 / 5xx.

**Search (mods)**

```
GET https://api.modrinth.com/v2/search
  ?query=<keywords>
  &limit=10
  &index=downloads
  &facets=[["categories:<loader>"],["versions:<mc_version>"],["project_type:mod"]]
```

**Search (shaders)**

```
  &facets=[["versions:<mc_version>"],["project_type:shader"]]
```

> Shaders have no loader facet — do not filter shaders by `neoforge`/`fabric`.

**Get builds**

```
GET https://api.modrinth.com/v2/project/<id-or-slug>/version
  ?loaders=["<loader>"]
  &game_versions=["<mc_version>"]
```

For shaders omit `loaders`.

The response's `files[]` each carry `url`, `filename`, `size`, and `hashes.sha1` / `hashes.sha512`. `dependencies[]` lists required prerequisites.

**No matching build for the user's version?** Do not force it. Say plainly that no build exists for that version, and offer the nearest supported version plus 2–3 alternatives. Never install a build built for a different major version — that is the #1 cause of a black screen at launch.

### 4.3 Quality floor

Do not recommend junk. When auto-picking for a user, require **both**: downloads ≥ 10,000 and a release within the last 24 months. Below that, show it only if the user named it themselves.

### 4.4 Save location

Download to a scratch folder, not straight into the game:

```
<root>/_mod_tool/staging/       ← downloads land here, verified before they move
<root>/_mod_tool/backup/        ← snapshots for rollback
<root>/_mod_tool/replaced/      ← originals displaced by an upgrade
<root>/_mod_tool/manifest.json  ← the ledger
```

Placing `_mod_tool/` inside the game root but **outside** the launcher's scanned folders keeps it out of the way.

---

## 5. Step 4 — Verify

Run every check. If any fails, **discard the file and report** — do not install.

### 5.1 Integrity

1. File begins with the ZIP magic bytes `PK\x03\x04`.
2. Computed **SHA-1 and SHA-512 both equal** the values the API published.
   - If they differ, the file is corrupt or tampered with. Delete it. Do not retry silently more than twice.

### 5.2 Is it the right kind of file?

| Target | Required inside the archive |
| ------ | --------------------------- |
| NeoForge mod | `META-INF/neoforge.mods.toml` |
| Forge mod | `META-INF/mods.toml` |
| Fabric mod | `fabric.mod.json` |
| Quilt mod | `quilt.mod.json` |
| Shader pack | `shaders/` directory, usually with `shaders/shaders.properties` |

A file with the wrong marker for the target loader is a **mismatch** — stop and tell the user.

### 5.3 Compatibility

From the mod's `mods.toml`, read the `[[dependencies.<modId>]]` entries:

- `modId = "minecraft"` → `versionRange` must cover the user's MC version
- `modId = "neoforge"` / `"forge"` → `versionRange` must cover the installed loader version
- `type = "required"` entries naming other mods → **install those too**, and tell the user what you added

Version ranges are messy in the wild. Parse the leading numeric part with `^\d+(\.\d+)*` and **zero-pad both sides to equal length before comparing** — otherwise `26.2` sorts below `26.2.0.0`, which is wrong. Real examples you must handle: `[26.2-,26.3)`, `[26.2.0.0-beta,)`, `[1.20.1,1.21)`.

### 5.4 Shaders need a shader loader

A shader pack is inert on its own. Check that one of these is installed:

- **Iris** (Fabric / NeoForge) — preferred
- **Oculus** (Forge)
- **OptiFine**

If none is present, say so and offer to install Iris as part of the same operation. Do not silently install it — shaders change how the game looks and costs FPS, so it deserves a mention. (This is not an extra question: state it as "I'm also adding Iris so the shader actually loads" and proceed unless the user objects.)

### 5.5 Conflict pre-check (mods only)

Read every jar already in `MODS_DIR` and collect their `modId`s — **including files ending in `.jar.disabled`**, which are still installable content. `zipfile` reads them regardless of extension.

- **Same filename already present** → this is a reinstall/upgrade, not a conflict. Back the old one up and replace.
- **Different filename, same `modId`** → a genuine duplicate. Stop and ask before installing.
- **Functional overlap** — the user already has a minimap, a recipe viewer, a performance mod, or a shader loader, and the new file does the same thing → flag it and ask.
- **File larger than 50 MB** → flag it and ask.
- Two mods already in the folder sharing a `modId` is a pre-existing problem. Report it; do not fix it unasked.

---

## 6. Step 5 — Install

### 6.1 Snapshot first

Before the **first** install of a session, copy the whole `MODS_DIR` (and `SHADER_DIR` if you will touch it) into `_mod_tool/backup/<timestamp>/`, recording name + size + SHA-1 for every file. This is what makes rollback total rather than approximate. It is cheap — usually well under 200 MB.

### 6.2 Install

1. If a file of the same name exists, **move it to `_mod_tool/replaced/<timestamp>__<name>` first**. Never overwrite in place.
2. Copy the verified file into `MODS_DIR` (or `SHADER_DIR`).
3. Re-hash the installed copy and confirm it matches what you verified in staging.

### 6.3 Record

Append to `_mod_tool/manifest.json`:

```json
{
  "name": "display name",
  "mod_ids": ["..."],
  "project_id": "...",
  "version": "...",
  "source": "modrinth",
  "url": "https://cdn.modrinth.com/...",
  "filename": "...",
  "sha1": "...",
  "sha512": "...",
  "size": 1234567,
  "target": "/absolute/path/to/mods/x.jar",
  "time": "ISO-8601",
  "action": "added | replaced",
  "replaced_original": "/path or null",
  "reason": "user requested | dependency of X"
}
```

A human-readable `_mod_tool/install-log.md` table alongside it is worth the extra write.

---

## 7. Step 6 — Report

Keep it short. The user cares about three things: did it work, where did it go, how do I undo it.

### English

```
Done.

Game folder found:  <path>
Version:            <mc version> + <loader> <loader version>
Installed:
  - <filename>  (<size>)  →  <MODS_DIR or SHADER_DIR>
  - <filename>  (<size>)   [auto-added dependency]
Verified:           SHA-1 + SHA-512 match the official values
Your existing files: untouched (N files, 0 modified)

Undo: <rollback command>
Restart the launcher before launching the game, so it rescans the folder.
```

### 中文

```
搞定。

找到的游戏目录：<路径>
版本：          <MC 版本> + <加载器> <加载器版本>
已安装：
  - <文件名>  (<大小>)  →  <mods 或 shaderpacks 目录>
  - <文件名>  (<大小>)   [自动补齐的前置]
校验：          SHA-1 和 SHA-512 与官方公布值逐字节一致
你原有的文件：  未改动（共 N 个，0 个被修改）

撤回：<撤回命令>
启动游戏前请先重启启动器，让它重新扫描文件夹。
```

If anything was skipped, blocked or mismatched, say so explicitly with the reason. Never report success for a partial install.

---

## 8. Rollback

Three levels, always available:

| Level | Command | What it does | Risk |
|---|---|---|---|
| **L1 — last** | `rollback --last` | Delete the most recent install; restore any displaced original. Skip a file whose hash has changed since install — the user edited it | none |
| **L2 — session** | `rollback --session` | Undo everything this session installed. Touches only your own files | none |
| **L3 — restore** | `rollback --restore <snapshot>` | Restore a whole folder from a snapshot | **back up the current state first** |

For L3, copy the live folder to `backup/pre-rollback_<timestamp>/` **before** overwriting. That way the rollback itself can be rolled back. Do not skip this.

---

## 9. Hard rules

1. **Never delete anything the user did not ask you to delete.** Mods, shaders, saves, configs, resource packs — all of it stays.
2. **Never touch** `saves/`, `config/`, `options.txt`, `servers.dat`, or the launcher's own files.
3. **Never overwrite.** Move the original aside, then place the new file.
4. **Never install an unverified file.** No hash match, no install.
5. **Never force a wrong version.** If there is no build for the user's version, say so.
6. **Never touch a second installation** you found while scanning, unless the user picked it.
7. **Report honestly.** Skipped, blocked, mismatched — all of it gets named.
8. **Ask at most two things**, plus at most one multiple-choice pick if detection is ambiguous.

---

## Appendix A — 中文完整流程（可直接给中文玩家看）

### A.1 你需要提供什么

只有两样：

1. **游戏版本号** —— 你实际在玩的那个，比如 `1.21.1`、`26.2`
2. **模组 / 光影的链接或名字**

**不用告诉我游戏文件夹在哪，我自己找。**

### A.2 我会自动做什么

1. **找游戏目录**。我会扫描这些位置，不需要你提供路径：
   - Windows：`%APPDATA%\.minecraft`、`C:\Users\你的名字\.minecraft`、CurseForge / Modrinth App / Prism Launcher / MultiMC 的实例目录，以及**启动器 exe 旁边的 `.minecraft`**（PCL2 / HMCL / BakaXL 常这么放）
   - macOS：`~/Library/Application Support/minecraft` 及各家启动器实例目录
   - Linux：`~/.minecraft`、Flatpak 的 `~/.var/app/com.mojang.Minecraft/.minecraft`、Prism / MultiMC 实例目录
2. **找到你的版本文件夹**，并判断是否开了**版本隔离**。
   - 开了隔离（PCL 的 `VersionArgumentIndieV2:True`），模组要放进 `versions\<版本名>\mods\`，放根目录的 `mods\` 是**不生效**的 —— 这是最常见的新手坑。
   - 没开隔离，放根目录 `mods\` / `shaderpacks\`。
3. **确认加载器**（NeoForge / Forge / Fabric / Quilt），只下载匹配的版本。
4. **下载并校验**。优先 Modrinth 官方源，下载后比对官方公布的 SHA-1 和 SHA-512，逐字节一致才算通过。
5. **检查冲突**。看看你要装的模组和你已有的模组有没有 modId 撞车、功能重复、缺前置。
6. **装进去并记账**。同名文件先移走再放新的，绝不直接覆盖。
7. **出问题能撤回**。分三级：撤最近一个 / 撤本次全部 / 整个目录还原。整目录还原前会先把当前状态再备份一份。

### A.3 光影特别说明

光影包本身不会生效，需要有**光影加载器**：

- **Iris**（Fabric / NeoForge）—— 推荐
- **Oculus**（Forge）
- **OptiFine**

没装的话我会一并帮你装上 Iris。

光影包是个 `.zip`，放进 `shaderpacks\` 文件夹，然后在游戏里 **选项 → 视频设置 → 光影** 里选。

### A.4 装完要做什么

**先重启启动器，再启动游戏。** 启动器只在启动时扫描 mods 文件夹，不重启看不到新模组。

### A.5 我不碰的东西

存档（`saves\`）、配置（`config\`）、`options.txt`、启动器本身 —— 一个字都不改。
你原有的模组也一个都不会动，只增不改。

---

## Appendix B — Copy-paste tooling

### B.1 Game folder auto-detection

Cross-platform, Python 3.8+, stdlib only. Prints JSON.

```python
#!/usr/bin/env python3
"""Locate Minecraft installations and resolve mods/ + shaderpacks/ targets."""
import json, os, re, sys
from pathlib import Path

HOME = Path.home()
SYSTEM = sys.platform  # 'win32' | 'darwin' | 'linux'

MARKERS = ("versions", "mods", "shaderpacks", "options.txt", "launcher_profiles.json")


def _bounded_walk(base: Path, max_depth=3):
    """Yield files under `base` without descending deeper than max_depth.
    Keeps the launcher hunt fast - a naive recursive glob over C:\\ can take minutes."""
    base = base.resolve()
    for dirpath, dirnames, filenames in os.walk(base):
        depth = len(Path(dirpath).relative_to(base).parts)
        if depth >= max_depth:
            dirnames[:] = []
        else:
            dirnames[:] = [d for d in dirnames
                           if d.lower() not in ("windows", "program files", "program files (x86)",
                                               "$recycle.bin", "system volume information",
                                               "appdata", "node_modules", ".git")]
        for fn in filenames:
            yield Path(dirpath) / fn


def find_launchers():
    """Locate portable launcher executables at shallow depth.

    Match on a prefix OR an alias: PCL2's real filename is
    'Plain Craft Launcher 2.exe', not 'PCL*.exe'.
    """
    aliases = ("pcl", "hmcl", "bakaxl", "plain craft launcher", "plaincraftlauncher",
               "bakaxl", "xmcl", "launcher")
    bases = [HOME]
    for extra in ("Desktop", "Downloads", "Documents", "桌面", "下载", "文档"):
        p = HOME / extra
        if p.is_dir():
            bases.append(p)
    if SYSTEM == "win32":
        for drive in ("C:", "D:", "E:"):
            d = Path(drive + os.sep)
            if d.is_dir():
                bases.append(d)
    found = []
    for base in bases:
        try:
            for f in _bounded_walk(base, 3):
                if f.suffix.lower() not in (".exe", ".jar"):
                    continue
                stem = f.stem.lower().replace("_", " ").replace("-", " ")
                if any(stem.startswith(a) or a in stem for a in aliases):
                    found.append(f)
                    if len(found) >= 20:
                        return found
        except (OSError, PermissionError):
            continue
    return found


def candidate_roots():
    out = []
    if SYSTEM == "win32":
        appdata = os.environ.get("APPDATA")
        if appdata:
            a = Path(appdata)
            out += [a / ".minecraft"]
            out += sorted((a / "ModrinthApp" / "profiles").glob("*")) if (a / "ModrinthApp" / "profiles").is_dir() else []
            out += sorted((a / "PrismLauncher" / "instances").glob("*/minecraft")) if (a / "PrismLauncher" / "instances").is_dir() else []
            out += sorted((a / "MultiMC" / "instances").glob("*/minecraft")) if (a / "MultiMC" / "instances").is_dir() else []
        for env in ("USERPROFILE", "HOME"):
            p = os.environ.get(env)
            if p:
                out.append(Path(p) / ".minecraft")
        cf = HOME / "curseforge" / "minecraft" / "Instances"
        if cf.is_dir():
            out += sorted(cf.glob("*"))
        for exe in find_launchers():
            out.append(exe.parent / ".minecraft")
            out.append(exe.parent / "minecraft")
    elif SYSTEM == "darwin":
        s = HOME / "Library" / "Application Support"
        out += [s / "minecraft"]
        for launcher, sub in (("PrismLauncher", "instances"), ("ModrinthApp", "profiles"),
                              ("CurseForge", "Instances"), ("MultiMC", "instances")):
            base = s / launcher / sub
            if base.is_dir():
                out += sorted(base.glob("*"))
                out += sorted(base.glob("*/minecraft"))
    else:
        out += [HOME / ".minecraft",
                HOME / ".var" / "app" / "com.mojang.Minecraft" / ".minecraft"]
        share = HOME / ".local" / "share"
        for launcher, sub in (("PrismLauncher", "instances"), ("multimc", "instances"),
                              ("ModrinthApp", "profiles"), ("curseforge", "minecraft/Instances")):
            base = share / launcher / sub
            if base.is_dir():
                out += sorted(base.glob("*"))
                out += sorted(base.glob("*/minecraft"))
    seen, uniq = set(), []
    for p in out:
        try:
            k = str(p.resolve()).lower()
        except OSError:
            continue
        if k not in seen:
            seen.add(k)
            uniq.append(p)
    return uniq


def is_root(p: Path):
    try:
        return p.is_dir() and any((p / m).exists() for m in MARKERS)
    except OSError:
        return False


def norm(s: str):
    return re.sub(r"[^a-z0-9]", "", s.lower())


def match_version(root: Path, wanted: str):
    vdir = root / "versions"
    if not vdir.is_dir():
        return None
    folders = [d.name for d in sorted(vdir.iterdir()) if d.is_dir()]
    if wanted in folders:
        return wanted
    nw = norm(wanted)
    for f in folders:
        if norm(f) == nw:
            return f
    for f in folders:
        if norm(f).startswith(nw) or norm(f).endswith(nw):
            return f
    for f in folders:
        if nw in norm(f):
            return f
    return None


def isolation_on(root: Path, ver: str):
    """Returns (bool, reason)."""
    setup = root / "versions" / ver / "PCL" / "Setup.ini"
    if setup.is_file():
        try:
            txt = setup.read_text(encoding="utf-8", errors="replace")
            if "VersionArgumentIndieV2:True" in txt or "VersionArgumentIndie:1" in txt:
                return True, "PCL version isolation flag"
        except OSError:
            pass
    hmcl = root / "hmcl.json"
    if hmcl.is_file():
        try:
            if '"versionIsolation": true' in hmcl.read_text(encoding="utf-8", errors="replace"):
                return True, "HMCL version isolation flag"
        except OSError:
            pass
    if (root / "versions" / ver / "mods").is_dir():
        return True, "versions/<ver>/mods exists"
    # instance-scoped launchers
    if any(k in str(root).lower() for k in
           ("instances", "profiles", "curseforge", "prism", "multimc", "modrinthapp")):
        return True, "instance-scoped launcher"
    return False, "no isolation detected"


def detect_loader(root: Path, ver: str):
    j = root / "versions" / ver / f"{ver}.json"
    if not j.is_file():
        return "unknown"
    try:
        txt = j.read_text(encoding="utf-8", errors="replace")
    except OSError:
        return "unknown"
    for needle, loader in (("net.neoforged", "neoforge"), ("net.minecraftforge", "forge"),
                           ("net.fabricmc", "fabric"), ("org.quiltmc", "quilt")):
        if needle in txt:
            return loader
    return "vanilla"


def main():
    wanted = sys.argv[1] if len(sys.argv) > 1 else None
    roots = [p for p in candidate_roots() if is_root(p)]
    report = []
    for r in roots:
        entry = {"root": str(r), "versions": [], "isolated": None, "loader": None,
                 "mods_dir": None, "shaderpacks_dir": None, "matched_version": None,
                 "last_played": None}
        vdir = r / "versions"
        if vdir.is_dir():
            entry["versions"] = [d.name for d in sorted(vdir.iterdir()) if d.is_dir()]
        # "last played" is the newest of these; isolated instances keep logs inside the version folder
        stamps = []
        for cand in [r / "logs" / "latest.log"] + \
                    [vdir / d / "logs" / "latest.log" for d in entry["versions"]]:
            try:
                if cand.is_file():
                    stamps.append(cand.stat().st_mtime)
            except OSError:
                pass
        if stamps:
            entry["last_played"] = max(stamps)
        if wanted and entry["versions"]:
            ver = match_version(r, wanted)
            if ver:
                entry["matched_version"] = ver
                iso, why = isolation_on(r, ver)
                entry["isolated"] = iso
                entry["isolation_reason"] = why
                entry["loader"] = detect_loader(r, ver)
                base = (r / "versions" / ver) if iso else r
                entry["mods_dir"] = str(base / "mods")
                entry["shaderpacks_dir"] = str(base / "shaderpacks")
        report.append(entry)
    print(json.dumps(report, ensure_ascii=False, indent=2))


if __name__ == "__main__":
    main()
```

Run it as `python detect.py <version>` and read the JSON.

### B.2 Download + verify (Modrinth)

```python
import hashlib, json, urllib.parse, urllib.request
from pathlib import Path

UA = "mc-mod-shader-installer/1.0"
API = "https://api.modrinth.com/v2"


def api(path, retries=4):
    import time
    for i in range(retries):
        try:
            req = urllib.request.Request(API + path, headers={"User-Agent": UA})
            return json.loads(urllib.request.urlopen(req, timeout=40).read().decode())
        except urllib.error.HTTPError as e:
            if e.code in (429, 500, 502, 503):
                time.sleep(2 * (i + 1))
                continue
            raise
    raise RuntimeError("Modrinth API retries exhausted")


def hashes(path, algos=("sha1", "sha512")):
    h = {a: hashlib.new(a) for a in algos}
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(1 << 20), b""):
            for d in h.values():
                d.update(chunk)
    return {k: v.hexdigest() for k, v in h.items()}


def find_and_download(query, mc_version, loader, kind, dest: Path):
    """kind: 'mod' or 'shader'. Returns the verified path."""
    facets = [["versions:" + mc_version], ["project_type:" + kind]]
    if kind == "mod":
        facets.append(["categories:" + loader])
    url = (f"/search?query={urllib.parse.quote(query)}&limit=10&index=downloads"
           f"&facets={urllib.parse.quote(json.dumps(facets))}")
    hits = api(url).get("hits", [])
    if not hits:
        return None, "no results"
    project = hits[0]["slug"]

    q = f"/project/{project}/version?game_versions={urllib.parse.quote(json.dumps([mc_version]))}"
    if kind == "mod":
        q += f"&loaders={urllib.parse.quote(json.dumps([loader]))}"
    versions = api(q)
    if not versions:
        return None, f"{project} has no build for {mc_version}"

    v = versions[0]
    f0 = next((x for x in v["files"] if x.get("primary")), v["files"][0])
    dest.mkdir(parents=True, exist_ok=True)
    out = dest / f0["filename"]

    tmp = out.with_suffix(out.suffix + ".part")
    req = urllib.request.Request(f0["url"], headers={"User-Agent": UA})
    with urllib.request.urlopen(req, timeout=180) as r, open(tmp, "wb") as fh:
        while True:
            buf = r.read(1 << 18)
            if not buf:
                break
            fh.write(buf)
    tmp.replace(out)

    if out.read_bytes()[:4] != b"PK\x03\x04":
        out.unlink(missing_ok=True)
        return None, "not a valid zip/jar"
    got = hashes(out)
    if got["sha1"] != f0["hashes"].get("sha1") or got["sha512"] != f0["hashes"].get("sha512"):
        out.unlink(missing_ok=True)
        return None, "hash mismatch - file discarded"
    return out, {"version": v["version_number"], "url": f0["url"], **got,
                 "dependencies": v.get("dependencies", []), "project_id": v["project_id"]}
```

### B.3 Version-range check for mods.toml

```python
import re

_NUM = re.compile(r"^\d+(?:\.\d+)*")


def _tup(s):
    m = _NUM.match((s or "").strip())
    return tuple(int(x) for x in m.group(0).split(".")) if m else None


def range_covers(rng, version):
    """True / False / None(unknown). Handles [26.2-,26.3), [26.2.0.0-beta,), [1.20.1,1.21)."""
    if not rng or rng.strip() in ("*", "[*]", ""):
        return None
    r = rng.strip()
    open_lo, open_hi = r.startswith("("), r.endswith(")")
    body = r[1:-1] if len(r) >= 2 and r[0] in "[(" and r[-1] in ")]" else r
    parts = body.split(",")
    lo_s = parts[0].strip() if parts else ""
    hi_s = parts[1].strip() if len(parts) > 1 else ""

    cur = _tup(version)
    if cur is None:
        return None
    lo, hi = _tup(lo_s), _tup(hi_s)
    if lo is not None and hi is None and len(parts) == 1 and not open_lo and not open_hi:
        hi = lo
    if lo is None and hi is None:
        return None

    n = max([len(cur)] + [len(x) for x in (lo, hi) if x])
    pad = lambda a: a + (0,) * (n - len(a))  # noqa: E731
    c = pad(cur)
    if lo is not None and (c < pad(lo) or (open_lo and c == pad(lo))):
        return False
    if hi is not None and (c > pad(hi) or (open_hi and c == pad(hi))):
        return False
    return True
```

---

## Appendix C — Quick reference

| Thing | Where |
|---|---|
| Mods, isolated instance | `<root>/versions/<ver>/mods/` |
| Mods, no isolation | `<root>/mods/` |
| Shaders, isolated instance | `<root>/versions/<ver>/shaderpacks/` |
| Shaders, no isolation | `<root>/shaderpacks/` |
| Shader pack format | a single `.zip` |
| Shader loaders | Iris (Fabric/NeoForge), Oculus (Forge), OptiFine |
| Modrinth API base | `https://api.modrinth.com/v2` |
| Required header | `User-Agent` |
| Mod loaders | `neoforge`, `forge`, `fabric`, `quilt` |
| Modrinth project types | `mod`, `modpack`, `resourcepack`, `shader` |
| macOS root | `~/Library/Application Support/minecraft` |
| Linux root | `~/.minecraft` |
| Flatpak root | `~/.var/app/com.mojang.Minecraft/.minecraft` |

---

*MIT licensed. Contributions welcome — especially more launcher paths and more language templates.*
