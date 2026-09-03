# Omega Database

Symbol mappings for Minecraft: where a name lives in a given build, per mod
loader and per side.

```
mappings/
  schema.json                        JSON Schema; every file validates against it
  index.json                         what exists, generated from the files
  {loader}/{side}/{version}.json.zst vanilla | fabric | forge | neoforge
                                     client | server
```

Release versions only. No snapshots, no pre-releases, no betas.

## One key, four answers

Every file is keyed by the **deobfuscated Mojang name**. A method key carries its
descriptor, because overloads share a name:

```
net/minecraft/client/Minecraft#tick()V
```

The same key resolves differently depending on which file is loaded:

| loader | 1.16.5 | 1.21.4 | 26.2 |
|---|---|---|---|
| vanilla | `djz.q` | `flk.t` | `Minecraft.tick` |
| fabric | `class_310.method_1574` | `class_310.method_1574` | `Minecraft.tick` |
| forge | `Minecraft.func_71407_l` | `Minecraft.m_91398_` | `Minecraft.tick` |
| neoforge | — | `Minecraft.tick` | `Minecraft.tick` |

That is the point of the database: consuming code names a symbol once and never
branches on version or loader.

## Namespaces

| loader | runtime namespace | source |
|---|---|---|
| vanilla | obfuscated, as shipped | Mojang ProGuard mappings |
| fabric | intermediary | FabricMC intermediary |
| forge | SRG, two eras | MCPConfig `joined.tsrg` |
| neoforge | Mojang official | identity, see below |

**fabric** runs on intermediary, not Yarn — Yarn is a development layer and never
appears in a running game. From 26.1 on, intermediary is empty: without
obfuscation no intermediate namespace is needed.

**forge** changed namespaces mid-range. Up to 1.16 classes carry the old MCP
names and members are `func_71407_l`; from 1.17 classes carry Mojang's names and
members are `m_91398_`. The two class namespaces are not interchangeable —
measured on 1.14.4, 4385 of 4860 class names differ.

**neoforge** dropped SRG entirely. Its NeoForm `joined.tsrg` maps every entry to
itself (8857 of 8858 classes on 1.21.4), so the runtime names are Mojang's. These
files are identity mappings, written for uniformity rather than for translation.
NeoForm starts at 1.20.2; older releases have no NeoForge.

## Coverage

| range | releases | state |
|---|---|---|
| 1.12 – 1.14.3 | 10 | obfuscated, no official mappings — **not dumped** |
| 1.14.4 – 1.21.11 | 39 | obfuscated, official mappings |
| 26.1 – 26.2 | 4 | not obfuscated |

Every loader namespace is built on top of Mojang's mappings, so the ten releases
without them are missing everywhere. Filling them needs MCP or Yarn as a name
source. Dumping them from the jar alone would put obfuscated names in the key
column, which reads like a mapping without being one.

NeoForge additionally covers only 1.20.2 and up.

## File format

```json
{
  "schema": 1,
  "minecraft": "1.21.4",
  "loader": "vanilla",
  "side": "client",
  "source": "mojang-official",
  "classes": {
    "net/minecraft/client/Minecraft": {
      "runtime": "flk",
      "methods": {
        "tick()V": { "runtime": "t", "descriptor": "()V", "pattern": "2A 59 B4 ?? ??" },
        "getInstance()Lnet/minecraft/client/Minecraft;": {
          "runtime": "Q", "descriptor": "()Lflk;", "static": true
        }
      },
      "fields": {
        "player": { "runtime": "t", "descriptor": "Lgkx;" }
      }
    }
  }
}
```

`descriptor` names the runtime types, which is what `GetMethodID` needs.

`static` comes from the class file's access flags, not from the mapping file:
ProGuard mappings do not record it, and JNI needs it to choose between
`GetMethodID` and `GetStaticMethodID`.

`pattern` holds the method body as wildcarded bytecode, with every operand that
reaches into the constant pool or a branch target masked as `??`. It verifies
that a runtime name still points at the right method, and finds the method again
when the name is wrong. Patterns are identical across loaders: remapping renames
constant pool entries but leaves the instruction sequence alone.

Abstract and native methods carry no pattern; they have no code to read.

## Why zstd

Measured on 1.21.4, because the intuition points the wrong way:

| encoding | raw | gzip | zstd |
|---|---|---|---|
| compact JSON | 23.2 MB | 2.43 MB | **1.64 MB** |
| MessagePack | 21.4 MB | 2.75 MB | 1.86 MB |
| MessagePack, patterns as raw bytes | 12.7 MB | 2.74 MB | 2.01 MB |
| plus a string table | 11.6 MB | 2.75 MB | 2.08 MB |

Every binary encoding halves the *uncompressed* size and enlarges the
*compressed* one: compression lives on exactly the redundancy those encodings
remove. MessagePack is also slower to parse with nlohmann, 110 ms against 91 ms.

Reading is unaffected by the compression — the largest file loads in 0.30 s.

## Validating

```bash
python -m pip install jsonschema zstandard
python - <<'PY'
import json, pathlib, zstandard
from jsonschema import Draft202012Validator
root = pathlib.Path("mappings")
validator = Draft202012Validator(json.loads((root / "schema.json").read_text()))
dctx = zstandard.ZstdDecompressor()
for f in sorted(root.glob("*/*/*.json.zst")):
    with f.open("rb") as raw, dctx.stream_reader(raw) as stream:
        errors = list(validator.iter_errors(json.load(stream)))
    print(f, "ok" if not errors else [e.message for e in errors])
PY
```

## Producing

Written by `omega_extractor` from the Omega2 repository:

```bash
uv run omega-mappings - mappings --loader all --patterns
```

Read at runtime by `Omega::Minecraft::SymbolProvider`, which prefers
`.json.zst`, then `.json.gz`, then plain `.json`.
