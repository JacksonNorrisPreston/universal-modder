---
title: "Cracking the post-Nov-2026 Steam meta encryption and extracting models"
game: "Umamusume: Pretty Derby"
kind: game
game_version: "Steam Global appid 3224770, install dated 2026-11-03"
platform: proton
engine: unity-il2cpp
route: asset-only
tools: ["UnityPy 1.24.2", "apsw-sqlite3mc", "Python 3.14 (pure-python chacha20 codec)"]
anti_cheat: "none"
status: working
verified: "2026-11-10"
agents:
  - "OpenCode (GLM 5.3 Prime)"
humans:
  - "jackson"
date: 2026-11-10
links: []
tags: [umamusume, unity, il2cpp, sqlite3mc, chacha20, asset-extraction, gltf]
---

# Cracking the post-Nov-2026 Steam meta encryption and extracting models

> Character 3D models extracted from the Global Steam release of Umamusume: Pretty Derby to
> rigged, textured GLB files, asset-only route on Linux without root. The encrypted `meta`
> manifest (SQLite3MultipleCiphers chacha20) was cracked by recovering the PBKDF2-derived codec
> key from the running game's memory; all 19,671 pages decrypt with valid Poly1305 tags.

## Setup

- Game: Umamusume: Pretty Derby, Steam Global, install dated **2026-11-03** (the update that broke
  every community tool), Unity **2022.3.62f2 IL2CPP**, no anti-cheat.
- Host: Manjaro Linux x64, **no root, no gcc/cmake**, game runs under GE-Proton11-7 via Steam.
- Python 3.14 with `UnityPy==1.24.2`, `Pillow`, `apsw-sqlite3mc` installed via
  `pip --target` (venvs are unusable in this harness: `python3` resolves to the OpenCode
  AppImage; `sys.executable` is rewritten even for `/usr/bin/python3.14`).

## Route and why

`asset-only`: the goal was extraction, not modding, so no loader or hook. Alternatives
considered: IL2CPP dumping to find the key in code (Il2CppDumper auto-mode fails on this build;
unnecessary once the memory-scan route worked), and Windows GUI tools (UmaViewer, uma-db-stuff) —
all broken on this build and Windows-only anyway.

## How the game's data works

- Assets live in `UmamusumePrettyDerby_Data/Persistent/`: `dat/` holds 172,162 hash-named Unity
  AssetBundles; `meta` is an 80 MB **encrypted SQLite** manifest; `master/master.mdb` and
  `d/SaveData.db` are plaintext SQLite.
- `meta` is SQLite3MultipleCiphers **chacha20** (sqleet variant): file[0:16] = salt, bytes 16–23 are
  plaintext SQLite header fields (page size 4096, reserved 32), encryption starts at offset 24
  (`CIPHER_PAGE1_OFFSET`). Each page ends with a 16-byte nonce + 16-byte Poly1305 tag in the
  reserved space. `counter = LE32(nonce[12:16]) ^ page_number`; the Poly1305 one-time key is
  `chacha20_block(key, nonce12, counter)[0:32]`; the body keystream starts at `counter+1`. KDF is
  PBKDF2-HMAC-SHA256 with **kdf_iter 64007**.
- The game wraps sqlite3mc in `LibNative.Sqlite3` (native plugin
  `UmamusumePrettyDerby_Data/Plugins/x86_64/libnative.dll`, exports `sqlite3_key`,
  `sqlite3mc_config`, CODEC_TYPE enum incl. CHACHA20/SQLCIPHER).
- Decrypted `meta` tables: `a` = assets (`n` name, `h` hash → `dat/<h[:2]>/<h>`, `e` per-asset
  key, `l` size), `r` = dependency graph (`f`, `t`, `d` depth — load a prefab bundle with all its
  `r`-deps so UnityPy resolves cross-CAB references), `c` category, `i` version.
- Bundle bodies are XOR-encrypted from byte 256; the XOR key comes from the `e` column via the
  community-known AB_KEY scheme (kairusds/umamusu-utils `_derive_asset_key`, 11-byte
  `53 2B 46 31 E4 A7 B9 47 3E 7C FB` → 88-byte key). The AB scheme did NOT change in the update.
- Body materials have **null `_MainTex`**; the game composites diffuse variants at runtime by name
  convention (`tex_<code>_{00}_{shape}_{bust}_{sock}_diff` under `<family>/textures/`). Head/face
  materials reference textures directly via cross-bundle CAB refs.
- Character names: `master.mdb` `text_data` category 6 (index = chara id; 1001 = Special Week);
  dress names: category 4 (index = chara_id*100 + dress). Ids below 1000 are generic/NPC bodies.

## The key recovery (the novel part)

The 2026-11-03 update replaced the meta DB key; all published keys (JP 32-byte
`6D5B6533…` XOR base; Global 12-byte `56 63 6B 63 42 72 37 76 65 70 41 62` XOR base) fail, and
UmaViewer issue #191 (Steam DBs "file is not a database") was still open. The key is not in
`global-metadata.dat` field defaults (only the AB key blob is, at a hex-named field) nor in string
literals.

Recovery: **launch the game and read its memory**. As the game opens `meta`, sqlite3mc allocates a
`ChaCha20Cipher` struct that holds the PBKDF2 output verbatim next to the file salt. Scan
`/proc/<pid>/mem` for the known 16-byte salt (meta file bytes 0–15); the struct appears in an
extension-data region as `{… ints …, kdf_iter=0x7fa0(64007), key_length=0x20, key[32], salt[16]}`.
Verify the candidate with the page-1 Poly1305 tag — that check is definitive.

Codec key for the 2026-11-03 Global build: `e1098eeaa3ae8a073377dc56cf07a1ff66f4c9cc8fd4654e8a288558aaffeeba`
(all 19,671 pages decrypt with valid tags). This is the *derived* key, not the passphrase.

Practical notes for the memory scan: `ptrace_scope=1` means you must be the parent (launch via
Proton yourself) or scan a Steam-launched instance while it runs (`/proc/PID/mem` reads worked
from the same user here). Launching Proton directly failed with exit 53 (prefix created by
GE-Proton11-7, Experimental refused to upgrade it) — launching through the user's Steam worked
and the game stayed up.

## Verification

- Poly1305 tag match on page 1 with the recovered key; then a full decrypt: **19,671/19,671 pages
  tag-valid**, resulting file passes `PRAGMA quick_check` and `meta.a`/`r` queries.
- A self-created sqlite3mc chacha20 DB (known passphrase) decrypted with the same Python codec,
  header fields matching a plaintext twin — proves the codec reimplementation.
- Bundle pipeline: UnityPy loads a decrypted bundle, objects parse (SkinnedMeshRenderer, 66–124
  bones, bindposes).
- GLB exporter: structural parse (nodes/meshes/skins/accessors), geometric-vs-vertex normal
  agreement 99%+ after winding fix, human-proportion bounding boxes (1.38 m body, 0.30 m head).
- Not verified: in-game visual comparison (no renderer on the extraction host); blendshape export
  (skipped this pass).

## Gotchas

1. **Community keys fail with `file is not a database` on the 2026-11-03 Steam build** → the meta
   key changed; recover it from game memory as above.
2. **`PRAGMA key` in apsw can't take raw bytes** (`X'..'` syntax error) and `cipher '3'` (numeric
   index) is rejected by sqlite3mc 2.6.1 → reimplement the codec in Python and use the Poly1305
   tag as your oracle, not header-plausibility heuristics (those give false negatives).
3. **Poly1305 implementation traps**: final partial block must be zero-padded to 16 BEFORE
   appending 0x01; clamp mask is `0x0FFFFFFC0FFFFFFC0FFFFFFC0FFFFFFF`; modulus is
   `(1<<130)-5`. RFC 8439 §A.3 expected tag for §A.1's key is `a8061dc1305136c6c22b8baf0c0127a9`.
4. **Unity→glTF winding must be REVERSED** (Unity front = CW, glTF front = CCW). Symptom:
   geometric normals anti-align with vertex normals (99% disagree). Emit `(a, c, b)`.
5. **Z-mirror conversion**: positions (x,y,-z), node quaternions (-x,-y,z,w), bindpose/inverse
   matrices conjugated `F·M·F`, textures flipped vertically with UV `v' = 1-v`.
6. **Multi-bundle loading in UnityPy**: pass bytes blobs and CAB registration misses bundles;
   write decrypted bundles to a temp dir and `UnityPy.load(dir)` — cross-CAB refs resolve.
7. **Reuse of one cursor for outer loop + inner lookups silently truncates the outer loop**
   (apsw) — materialize query results first. Symptom: dependency lists of 1 instead of 7.
8. **bindpose attribute names are `e00..e33`** in UnityPy 1.24 (`m00` fails silently → empty skin).
9. **Materials attach to the GameObject's Transform, not to the renderer component** — build a
   GameObject→Transform map or `transforms[k]` KeyErrors on the SMR's own key.
10. **Proton prefix mismatch kills direct launches** (exit 53, "invalid version"): use the exact
    Proton that created the prefix (check `compatdata/<appid>`), or scan the user's own
    Steam-launched game.
11. **Il2CppDumper 6.7.46 auto-mode fails on this build** (CodeRegistration not found); not needed
    for this task anyway.
12. Body diffuse textures need the naming-convention heuristic; do not expect `_MainTex` on body
    materials — heads have them, bodies don't.

## Artifacts (this machine, personal use)

`~/umamusume_models/` — GLBs per character/dress + `_tools/` (codec, exporter, decrypted meta,
key file) + README + MODLOG.
