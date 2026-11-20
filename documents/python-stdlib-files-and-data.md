# Python reference for chip authors — files, data, and bytes

Written for this pack: a condensed, original reference to the
standard-library modules chips reach for when they read,
write, hash, pack, and parse. Python 3.12, stdlib only.

## json — reading and writing JSON

How do I read a JSON file in Python? Open it as text with an
explicit encoding and hand the file object to `json.load`:

```python
import json

with open(path, encoding="utf-8") as f:
    data = json.load(f)
```

- `json.load(f)` parses from a file object; `json.loads(s)`
  parses from a string. `json.dump(obj, f)` writes to a file;
  `json.dumps(obj)` returns a string.
- `json.dumps(obj, indent=2)` pretty-prints — the shape the
  bridge writes its own stores in. `ensure_ascii=False` keeps
  non-ASCII text as-is instead of `\uXXXX` escapes.
- JSON parses to plain Python: objects to `dict`, arrays to
  `list`, strings to `str`, numbers to `int`/`float`, `true`
  to `True`, `null` to `None`. Trust none of the shapes:
  check `isinstance(data, dict)` and read with `.get()` —
  a manifest is outside data until it proves itself.
- Bad JSON raises `json.JSONDecodeError` (a `ValueError`).
  Catch it and name the file in the refusal.

## pathlib and os — paths and files

- `from pathlib import Path`: `p = Path("chips") / name`;
  `p.read_text(encoding="utf-8")`, `p.write_text(text,
  encoding="utf-8")`, `p.read_bytes()`, `p.write_bytes(b)`;
  `p.exists()`, `p.is_file()`, `p.suffix`, `p.stem`,
  `p.parent`; `p.glob("*.png")` lists matches;
  `p.mkdir(parents=True, exist_ok=True)` makes directories.
- `os.path.join(a, b)`, `os.path.basename(p)`,
  `os.path.dirname(p)` — the bridge's own idiom; either
  library is fine, mixing them carelessly is not.
- `os.environ` is a dict of the environment: read with
  `os.environ.get("NAVI_MODEL", "navi")`. A chip receives the
  operator's whole environment — read what the API promises,
  leave the rest alone.
- `os.listdir(d)` lists a folder (the bridge guards it with
  try/except and treats failure as an empty folder).
- Temp space: `tempfile.mkdtemp(prefix="navi-chip-")` makes a
  private directory; clean it with
  `shutil.rmtree(tmp, ignore_errors=True)` in a `finally` —
  the exact discipline `execute_chip` itself uses.

## re — regular expressions

- `re.search(pat, s)` finds anywhere; `re.match` anchors at
  the start; `re.fullmatch` anchors both ends; `re.findall`
  returns all matches; `re.sub(pat, repl, s)` rewrites.
- Groups: `m.group(1)`, named groups `(?P<name>...)` read as
  `m.group("name")`. Flags: `re.I` case-insensitive, `re.M`
  multiline `^`/`$`, `re.S` dot-matches-newline.
- Compile once when a pattern runs in a loop:
  `PAT = re.compile(r"^\s*def\s+run\s*\(")`. Raw strings
  (`r"..."`) for every pattern — the Studio's own structure
  checks are four compiled ideas exactly like this.
- Escape user text before embedding it in a pattern:
  `re.escape(text)`.

## datetime — times and stamps

- `datetime.datetime.now()` is naive local time;
  `datetime.datetime.now(datetime.timezone.utc)` is aware
  UTC. Never mix naive and aware in a comparison — it raises.
- Parse ISO with `datetime.datetime.fromisoformat(s)`;
  render with `.isoformat()` or `.strftime("%Y-%m-%d %H:%M")`;
  parse a known format with `strptime(s, fmt)`.
- `datetime.timedelta(days=1, seconds=30)` does arithmetic;
  `datetime.date.today()` for dates; `time.time()` is the
  float epoch when a plain number is what's wanted.

## hashlib — digests

How do I compute a sha256 in Python? Hash the **bytes** with
`hashlib.sha256`, read the digest as hex:

```python
import hashlib

digest = hashlib.sha256(data).hexdigest()
```

- `data` must be `bytes` — encode text first:
  `hashlib.sha256(text.encode("utf-8")).hexdigest()`.
- Hash a file in chunks so a big file never loads whole:

```python
h = hashlib.sha256()
with open(path, "rb") as f:
    for block in iter(lambda: f.read(65536), b""):
        h.update(block)
digest = h.hexdigest()
```

- `hashlib.md5`, `hashlib.sha1`, `hashlib.sha512` share the
  shape; `h.digest()` returns raw bytes when hex isn't the
  need. For keyed integrity use `hmac.new(key, msg,
  hashlib.sha256)` — plain hashes are fingerprints, not
  signatures.

## zipfile — archives

- Read: `with zipfile.ZipFile(path) as z:` then
  `z.namelist()`, `z.read("root.py")` (bytes — decode for
  text), `z.getinfo(name).file_size`.
- Write: `zipfile.ZipFile(out, "w", zipfile.ZIP_DEFLATED)`,
  `z.write(file, arcname)` from disk or `z.writestr(arcname,
  text_or_bytes)` from memory — the bridge's chip export is
  built on `writestr`.
- Treat archive member names as hostile paths: a member with
  `..` or a leading `/` must never be written to disk as-is.

## struct — binary layouts, and PNG chunks

`struct.pack(fmt, ...)` builds bytes; `struct.unpack(fmt,
data)` parses them. Format strings state endianness first:
`>` big-endian, `<` little-endian, then codes — `I` unsigned
32-bit, `H` 16-bit, `Q` 64-bit, `B` a byte, `Ns` N raw bytes.
`struct.calcsize(fmt)` is the byte count. Network and file
formats are big-endian until proven otherwise.

**What Python do I use to write a PNG tEXt chunk?** `struct`
for the length, `binascii.crc32` for the checksum — this is
exactly how the shipped bridge builds every chunk
(`_png_chunk` and `build_cartridge_png` in bridge.py):

```python
import binascii, struct

def png_chunk(kind, data):
    out = struct.pack(">I", len(data)) + kind + data
    out += struct.pack(">I", binascii.crc32(kind + data) & 0xFFFFFFFF)
    return out

text_data = b"navi-chip\x00" + json_bytes      # keyword, NUL, payload
chunk = png_chunk(b"tEXt", text_data)
```

A PNG is the 8-byte signature `b"\x89PNG\r\n\x1a\n"` followed
by chunks of *length, type, data, CRC*. The bridge's forge
splices its tEXt chunk in right after IHDR (which ends at
byte `8 + 12 + 13`). Reading a chunk back is the same walk
`parse_cartridge_png` does:

```python
pos = 8
while pos + 12 <= len(data):
    (length,) = struct.unpack(">I", data[pos:pos + 4])
    kind = data[pos + 4:pos + 8]
    chunk_data = data[pos + 8:pos + 8 + length]
    # ... inspect kind / chunk_data ...
    pos += 8 + length + 4
```

Guard every step: a file shorter than its declared chunk is
not an error to raise on — it is a `None` to return, the way
the shipped parser treats a damaged cartridge.

## sqlite3 — a small real database

- `import sqlite3`; `con = sqlite3.connect(path)`;
  `cur = con.cursor()`; `cur.execute(sql, params)`;
  `con.commit()`; close with `with` or `con.close()`.
- **Parameters are never string formatting.** Write
  `cur.execute("SELECT * FROM t WHERE id = ?", (ident,))` —
  the driver quotes safely; an f-string in SQL is an
  injection waiting for its input.
- `cur.fetchall()` returns tuples; `con.row_factory =
  sqlite3.Row` makes rows readable by column name.
- Create tables defensively:
  `CREATE TABLE IF NOT EXISTS ...`. A chip's database lives
  where the chip says it does — name the path in the chip's
  description, because the sandbox is wiped after every run:
  persistent state must live in a real folder (the working
  directory or a path the operator passes in), never in the
  sandbox.
