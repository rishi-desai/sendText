# sendText — OSR XML Telegram Sender

Interactive CLI tool for composing, managing, and sending XML telegrams to an OSR (Order Storage and Retrieval) system via omniORB/CORBA. Supports one-shot non-interactive modes and a full-featured batch queue.

---

## Requirements

- Python 3.6+
- `omniORB` Python bindings (`omniORB`, `CosNaming`, `OSR`)
- Access to the OSR NameService (CORBA)
- The script must be **run on a host that can reach the OSR NameService**

---

## Setup

The script is self-contained. Telegram files and config are stored in a `telegrams/` directory created automatically **next to the script**:

```
scripts/
├── sendText          ← the script
└── telegrams/
    ├── my-order.xml
    ├── transport-flowrack.xml
    └── config/
        ├── sim.json
        ├── container_naming.json
        ├── sourceLoop.txt
        └── targetLoop.txt
```

### OSR Target

Set the target OSR via environment variable or flag:

```bash
sendText                       # uses $OSR_ID
sendText --osr-id osr1         # or explicit flag
```

---

## Usage

```
sendText [--osr-id OSR_ID] [OPTIONS] [FILE]
```

### Modes

| Command                     | Description                                                       |
| --------------------------- | ----------------------------------------------------------------- |
| `sendText`                  | **Interactive loop** — pick from saved telegrams, send, loop back |
| `sendText --pick` / `-p`    | Pick once from saved list, send, exit                             |
| `sendText <name.xml>`       | Send a saved telegram by name (resolved from `telegrams/`)        |
| `sendText <file.xml>`       | Send any XML file by full or relative path                        |
| `sendText --last`           | Send the most recently saved telegram                             |
| `cat order.xml \| sendText` | Send from stdin                                                   |
| `sendText --list-saved`     | List all saved telegrams and exit                                 |
| `sendText --delete <name>`  | Delete a saved telegram and exit                                  |

### Flags

| Flag             | Description                                        |
| ---------------- | -------------------------------------------------- |
| `--osr-id <id>`  | OSR target (overrides `$OSR_ID`)                   |
| `--editor <cmd>` | Editor to use (overrides `$EDITOR`, default: `vi`) |
| `-y` / `--yes`   | Skip confirmation prompt — send immediately        |
| `--no-sim`       | Skip simulator insert even when sim is enabled     |

---

## Interactive Picker

When launched in interactive mode (`sendText`), a numbered list of saved telegrams is shown. Each entry displays filename, modification time, size, and a parsed summary of the order type, order number, and container number.

Each entry shows the filename, modification time, size, and a colour-coded order type summary. Recognised order types:

| Display label  | XML tag                                              | Notes                     |
| -------------- | ---------------------------------------------------- | ------------------------- |
| `TRANSPORT GI` | `transport_order` with `<product>` child             | Goods-in transport        |
| `TRANSPORT`    | `transport_order` without products                   | Standard transport        |
| `PICK`         | `pick_order`                                         | Pick order                |
| `INVENTORY`    | `inventory_order`                                    | Inventory                 |
| `GOODS IN`     | `goods_in_order`                                     | Goods-in from pickstation |
| `GOODS ADD`    | `goods_in_order` with a child element text `renewal` | Renewal/replenishment     |
| `DISPATCH`     | `dispatch_order`                                     | Dispatch                  |

Unrecognised tags fall back to a generic XML attribute display.

```
  Saved telegrams  (newest first)
  ────────────────────────────────────────────────────────────
    1)  transport-rack-location.xml   2026-06-04 00:50:24   1.1 KB   TRANSPORT GI    ord: RISHI-ORD-001  cont: RADTOTE08
    2)  transport-flowrack.xml        2026-06-01 22:58:14    416 B   TRANSPORT       ord: RISHI-FLOW-001  cont: RADTOTE01
    3)  transport-no-lines.xml        2026-06-01 22:55:11    394 B   TRANSPORT       ord: RISHI-ORD-001  cont: RADTOTE01
```

### Picker Commands

| Input           | Action                                          |
| --------------- | ----------------------------------------------- |
| `<n>`           | Send telegram number n                          |
| `v <n>`         | Preview XML of telegram n                       |
| `e <n>`         | Edit telegram n in-place (opens `$EDITOR`)      |
| `d <n>`         | Delete telegram n (with confirmation)           |
| `d <n> <m> ...` | Delete multiple telegrams by number             |
| `d <n>-<m>`     | Delete a range of telegrams (e.g. `d 2-5`)      |
| `m <n>`         | Rename telegram n                               |
| `dup <n>`       | Duplicate telegram n (timestamped copy)         |
| `n`             | Create a new telegram in editor                 |
| `/<text>`       | Filter list by filename or XML content          |
| `r`             | Refresh list (also clears active search filter) |
| `Q`             | Open the **Queue menu**                         |
| `sim`           | Open the **Simulator config menu**              |
| `q`             | Quit                                            |

#### Search

Type `/` followed by text to filter the list:

```
> /rack          # shows only telegrams whose filename or XML contains "rack"
> /RADTOTE08     # filter by container number
> r              # refresh clears the filter
```

---

## Queue Menu

The queue allows you to build a batch of orders and send them all in sequence with an optional inter-send delay. Open it with `Q` from the picker.

Each queue entry holds a template file, a count, and starting order/container numbers. When sent, numbers are auto-incremented across the batch (e.g. ORD-001 → ORD-002 → ORD-003).

### Telegram management (same as picker)

| Command   | Action                                                    |
| --------- | --------------------------------------------------------- |
| `v <n>`   | Preview telegram n                                        |
| `e <n>`   | Edit telegram n in-place                                  |
| `d <n..>` | Delete telegram(s) — supports ranges and multiple indices |
| `dup <n>` | Duplicate telegram n                                      |
| `m <n>`   | Rename telegram n                                         |
| `n`       | Create new telegram (optionally add to queue immediately) |
| `/<text>` | Search filter                                             |

### Queue operations

| Command          | Action                                                     |
| ---------------- | ---------------------------------------------------------- |
| `a <n> [count]`  | Add telegram n to queue (count times, default 1)           |
| `dq <n>`         | Remove queue entry n                                       |
| `xq <n> <count>` | Change the count of queue entry n                          |
| `vq <n>`         | Preview expanded order/container numbers for queue entry n |

### Queue actions

| Command      | Action                                             |
| ------------ | -------------------------------------------------- |
| `send` / `s` | Send all queued orders (asks for confirmation)     |
| `clear`      | Clear the queue                                    |
| `delay <s>`  | Set the inter-send delay in seconds (default: 2.0) |
| `cfg`        | Open container naming config                       |
| `sim`        | Open simulator config                              |
| `r`          | Refresh telegram list                              |
| `back` / `b` | Return to picker                                   |

### Auto-increment

When a queue entry has `count > 1`, the order number and container number are incremented automatically for each item. After sending, all bases advance so the next send continues from where the last one left off.

Example: entry with base `ORD-001` / `TOTE001` and count 3 will send:
- `ORD-001` / `TOTE001`
- `ORD-002` / `TOTE002`
- `ORD-003` / `TOTE003`

---

## Container Naming (`cfg`)

The naming config sets auto-advance prefixes for container numbers when adding to the queue. This prevents collisions when you have multiple queue entries.

| Field         | Used for                          |
| ------------- | --------------------------------- |
| Source prefix | transport GI and transport orders |
| Target prefix | pick orders                       |

Set via `cfg` in the queue menu. Examples: `RADTOTE001`, `T00001`, `C00001`.

---

## Simulator Integration

When the simulator is enabled, `sendText` automatically inserts the container into the simulator after each successful send.

| Order type                                       | Sim insert behaviour                                                                 |
| ------------------------------------------------ | ------------------------------------------------------------------------------------ |
| `TRANSPORT GI` / `TRANSPORT`                     | Inserts only if container is **not already present** in the simulator                |
| `PICK`                                           | Always inserts                                                                       |
| `INVENTORY`, `GOODS IN`, `GOODS ADD`, `DISPATCH` | No sim insert by default — add to `_SIM_INSERT_TYPES` at top of the script to enable |

Before each interactive session, `sendText` starts a background thread that runs `simulator --config simosrX.properties -c` and caches the full container list. The presence check reads from this in-memory cache instantly — no blocking wait during sends. The cache refreshes automatically every 30 seconds for the lifetime of the process.

Use `--no-sim` to skip sim insert for the current send.

### Simulator Config Menu (`sim`)

Accessible via `sim` in the picker or queue menu.

| Command         | Action                                         |
| --------------- | ---------------------------------------------- |
| `toggle`        | Enable / disable sim insert                    |
| `user <name>`   | Set the simulator OS user (default: `sandbox`) |
| `src <elem>`    | Set default source element                     |
| `tgt <elem>`    | Set default target element                     |
| `addsrc <elem>` | Add an element to the source round-robin list  |
| `addtgt <elem>` | Add an element to the target round-robin list  |
| `delsrc <n>`    | Remove element n from source list              |
| `deltgt <n>`    | Remove element n from target list              |
| `back`          | Return                                         |

#### Element Round-Robin

If `sourceLoop.txt` or `targetLoop.txt` contain multiple elements, the simulator cycles through them in order across sends. This is useful when inserting into different conveyors or locations in sequence.

---

## Config Files

All config files live in `telegrams/config/` next to the script.

| File                    | Description                                                      |
| ----------------------- | ---------------------------------------------------------------- |
| `sim.json`              | Simulator enabled flag, sim user, default source/target elements |
| `container_naming.json` | Auto-advance source/target container prefixes                    |
| `sourceLoop.txt`        | Newline-separated list of source elements (round-robin)          |
| `targetLoop.txt`        | Newline-separated list of target elements (round-robin)          |

---

## Telegram Storage

Saved telegrams live in `telegrams/` next to the script.

- When you send new XML (from editor, stdin, or file), it is saved automatically with a timestamped filename: `telegram_20260604_123456_789012.xml`
- If an identical XML already exists in `telegrams/`, no duplicate file is created — the existing file is reused
- You can rename files to something descriptive (e.g. `transport-flowrack.xml`) using `m <n>`

---

## Examples

```bash
# Interactive session on osr1
sendText

# Send the last saved telegram without confirmation
sendText --last -y

# Send a saved telegram by name (resolves from telegrams/)
sendText transport-flowrack.xml

# Send any XML file by path
sendText /tmp/my-order.xml

# Send from a file, skip sim insert
sendText --no-sim my-order.xml

# Send from stdin
echo '<host2osr>...</host2osr>' | sendText

# List all saved telegrams
sendText --list-saved

# Delete a saved telegram
sendText --delete telegram_20260601_225814.xml
```
