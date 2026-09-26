# BM6 battery monitor — BLE protocol reference

Reverse-engineered reference for the **leagend BM6** Bluetooth LE 12 V battery monitor
(the same protocol is reported for BM2/BM200, Sealey BT2020 and intAct Battery Guard —
not checked by us). It covers the live measurement, the on-sensor **history** (the part
nobody documents), the **state-of-charge table** the app writes into the sensor, and the
traps we paid for.

Everything here was read off real sensors and cross-checked against the vendor app
`com.dc.bm6` (decompiled with apktool). Every claim says what confirms it; **"not
established" is a full entry, not a gap.**

---

## ⚠ Applicability

| | |
|---|---|
| Board | `BM6_V2.0`, PCB dated 20220609, FCC ID 2AU4A-191114 (one unit opened) |
| Firmware reply to `01` | `d1 55 01 01 08 00 04 00…` |
| Vendor app | "BM6" (`com.dc.bm6`) |
| Units observed | two: **sensor A** on a car, **sensor B** taken off a quad bike and powered from a bench supply |

Where a fact comes from only one of the two, it says so. Host side: ESP32 / ESP32-S3
with NimBLE-Arduino 2.x, and an Android phone for HCI logs.

## ⚠ One command is named "lock"

`d1 55 02 11` is `CMD_LOCK` in the app. We never sent it, so what it does and whether
`CMD_UNLOCK` undoes it are not verified. Do not sweep command codes blind.

---

## 1. Transport

| | |
|---|---|
| Service | `FFF0` |
| Write | `FFF3` |
| Notify | `FFF4` (the CCCD must be enabled) |
| Cipher | AES-128, key `"leagend"` + `FF FE` + `"0100009"` |
| MAC prefixes seen | `50:54:7b:`, `3c:ab:72:` |

The key is static and shared by all units (from the APK; also published earlier, see
[Prior work](#prior-work)):

```
6c 65 61 67 65 6e 64 ff fe 30 31 30 30 30 30 39
```

**Each 16-byte block is encrypted independently.** Formally it is CBC with a zero IV, but
there is no chaining between blocks — effectively ECB. Visible because identical
plaintext blocks (trailing zeros) give identical ciphertext. Decrypt long replies **block
by block**; running CBC over the whole buffer turns everything after the first block into
garbage.

Commands shorter than 16 bytes are zero-padded to one block.

**MTU.** A history packet is 160 bytes in one notification; it does not fit the default 23.
The phone negotiates 247; NimBLE-Arduino asks for 255 by default, so nothing to change.

**No data in advertising** — you have to connect. And connect **on the advertisement**,
not after a full scan: the sensor draws about 1.5 mA, advertises in bursts and goes quiet
quickly. Median gap between advertisements on sensor B: 6 s (38 packets in 5 min, max gap
38 s).

---

## 2. Who owns the sensor

**The sensor belongs to whoever HOLDS the connection.** While a link is held, the sensor
**does not advertise at all** — not "busy", silent — so a second central has nothing to
connect to.

Measured both ways (Sept 2026, sensor B):

- phone held the link → our device did not see the sensor for 20 min;
- our device held the link with the phone's app open next to it → 5382 frames, zero
  drops, the phone never got in;
- our device released the link → the phone connected **4 s** later (`CONNECTED` in
  `dumpsys bluetooth_manager`).

No firmware trick gets around this. When "the sensor disappeared", first ask whether the
app is open on a phone.

**After ONE `07` request the sensor streams measurements by itself, once per second**, for
as long as the link lives. Phone HCI log: 167 notifications per session with a median gap
of 1.00 s and only 10 writes from the app, all in the first three seconds. Our own
firmware sends only `07` (no unlock) and receives the same 1 Hz stream for hours. A
"connect, ask, disconnect" scheme throws this stream away.

The app keeps up to **four** sensors connected at once (`BleService`:
`isSingle || 连接size >= 4 → stopScan()`).

---

## 3. Command set

Names from `Constants.smali` of the app — field names, not guesses. The app's debug
strings label each reply.

| code | name in the app | request bytes | what |
|---|---|---|---|
| `01` | `CMD_VERSION` | `d15501` | sensor firmware version |
| `02` | `CMD_UNLOCK` | `d1550200` | **unlock sensor** |
| `02` | `CMD_LOCK` | `d1550211` | **lock sensor** — see warning above |
| `03` | `CMD_START_DATA` | `d15503` | cranking (starter) data |
| `04` | `CMD_CHARGE_TEST_FIRST/SECOND/THREE` | `d1550401` / `02` / `03` | charging-system test, three parts |
| `05` | `CMD_HISTORY_HEADER` | `d15505` | history |
| `06` | `CMD_HISTORY_END` | `d15506` | "end of history read" |
| `07` | `CMD_REAL_DATA` | `d15507` | live measurement |
| `08` | `CMD_LEAD_BATTERY` / `CMD_IRON_BATTERY` | `d1550801` / `d1550802` | battery type + SoC table |

`d15503ff00` means "accepted, no data". `d15507ff00` is the acknowledgement of a
measurement request; the measurements follow it (see byte 3 below).

What the app does on session open, in order: subscribe to `FFF4`, then `01`, `02`
(unlock), `08` as two packets, `05` twice, `03`, `07` — then it only listens.

---

## 4. Command `07` — live measurement

Reply is 16 bytes. **Offsets are in bytes.** Earlier write-ups give offsets in characters
of the hex string (voltage = characters 15–18); that is the same field — the low nibble
of byte 7 plus byte 8.

```
d1 55 07 | 00 | 19 | 02 | 64 | 05 56 | 00 00 00 00 | 02 | 00 00
          sign temp stat soc  voltage
```

| field | where | notes |
|---|---|---|
| temperature sign | byte 3 | `00` plus, `01` minus |
| temperature | byte 4 | °C |
| **status** | byte 5 | `0` normal, `2` charging, anything else "low charge" |
| state of charge | byte 6 | % (computed by the sensor, see §6) |
| voltage | (byte 7 & 0x0F) << 8 \| byte 8 | /100 V |

**Byte 3 = `ff`** is the acknowledgement, not a measurement: all fields are zero. Filter
it out, or it decodes as an honest-looking 0.00 V on a live battery.

**Byte 5 — status.** Cross-checked three ways: the branch on `RealTimeBean.status` in
`BatteryFragment` (`==0` normal, `==2` charging, else "low charge"); 644 bench readings
(`0x02` at 13.68…14.29 V, `0x01` at 11.99…12.26 V, `0x00` once at 13.08 V on the way up);
the app's card labels. Transition thresholds **not established** — the voltage was changed
in steps.

**Negative temperatures** were never observed on real hardware.

**Bytes 9–15.** Byte 13 is always `02`, meaning not established. The app's live-frame bean
also has `rapidAcce` and `rapidDece`; which bytes they map to was not checked.

**Upper nibble of byte 7** was zero at every observed voltage, so "12-bit voltage" vs
"16-bit big-endian" cannot be told apart from these readings. The 12-bit reading is
supported by the history record and the SoC table, which both pack voltage in 12 bits.
Between 6 and 20 V both give the same result.

---

## 5. Command `05` — history

The sensor logs a record **every 2 minutes** by itself and keeps weeks of them. The whole
trip of a car is visible: how long it stood, engine starts and stops, how fast the battery
drains while parked.

### Request

Nine bytes, zero-padded:

```
d1 55 05 | SS SS SS | EE EE EE
           start      end        record indices, 24-bit big-endian
```

Records **`[start, end)`** are returned — half-open, exactly `end − start` of them.
Checked: `[0,20)` → 20 records, `[5,10)` → five, `[10,10)` and `[20,10)` → none, and
`[10,20)` matched records 10…19 of a `[0,30)` read taken before and after it.

The sensor uses only the **low 16 bits** of each index: bytes 3 and 6 do not affect the
reply. The app fills all 24 bits anyway.

If `end` exceeds what has been accumulated, the reply is **cut at the accumulated count** —
that is how you learn how many records exist (not the memory size). Verified with `start`
inside the accumulated range; with `start` beyond it the sensor returns raw memory (trap 1
below). `start` exactly equal to the accumulated count: not established.

### The index is the time

**Record 0 is the newest, step 2 minutes**: record k was taken k·2 minutes ago. Records
carry no timestamp; the app also computes time by arithmetic from the moment of the read.

| source | evidence |
|---|---|
| measurement | 15 snapshots of 40 records, one per minute: the offset grew by 1 exactly every 2 minutes, 100 % match on all 14 transitions |
| APK | `0x1d4c0` = 120000 ms in both the request builder and the reply parser; also `0x2d0` = 720 records per day |

**A new record lands at index 0 even in the middle of a read**, and every index shifts by
one. Sensor A, pages of 8000 with a one-record overlap: the re-read record at the page seam
was found one position further in 4 reads out of 14. If you read in pages, check the seam
(see §5.6).

### Record — 4 bytes

| bits | field |
|---|---|
| 12 | voltage, /100 V |
| 8 | state of charge, % |
| 8 | temperature, °C |
| 4 | flags |

Example: `52 e6 41 80` → `0x52e` = **13.26 V**, `0x64` = **100 %**, `0x18` = **24 °C**,
flags 0.

**How a negative temperature is encoded in a record: not established.** The live frame
carries the sign in a separate byte; the record has no room for one. No negative values
seen (in 50467 records of sensor A the top bit of the temperature byte is never set).
Reading it as two's complement would be a guess.

**Flags: `2` — "start", `4` — "stop"** — as the app names them; the APK parser compares the
**low three bits** with the strings `010` and `100`. **The sensor sets them on a voltage
step; it knows nothing about the engine.** On the bench, with no engine: a drop from 13.98
to 12.0 V gave `4`, a rise to 14.2 V gave `2`; a drop from 13.48 to 12.79 V gave `4`; the
first record after power-up (0 to 13.48 V) gave `2`. On sensor A all 9 flags from 21 Jul
to 29 Aug are false: the car was not started after 15 Jul — these were charger
connections (battery kept at 13.45 V in storage) and capacity experiments (per the owner).
A real engine start is caught differently — by the fast cranking buffer, see `03`; history
flags and that buffer are separate things. Step thresholds not established.

**An all-zero record** is a real record in the middle of the data, not the end. It appears
at a sharp voltage change, right before a record flagged `4` (three times on the bench; on
sensor A all 4 zero records out of 50467 sit right before a `4`). Meaning not established.
The app skips them.

### Reply

1. Acknowledgement `d1 55 05 CC CC 00` — 16-bit record count.
2. Notifications of 160 bytes, up to 40 records each; **the last one is zero-padded**.
3. Terminator:

```
ff fe fe 00 | LL LL     LL LL = 4·RETURNED + 9, 16-bit big-endian
```

The **returned** records are counted, not the requested ones: `[0, 0xff00)` returned 696
records with 2793 = 4·696+9.

**Take the record count from the terminator, not from received bytes.** The zero padding
of the last packet is indistinguishable from real zero records: by bytes you get 2480
instead of 2467. We fell for it — "three reads in a row gave exactly 50120, the memory
must be full" was just 2097 and 2106 both padded up to 2120.

**The length field is 16 bits**: 4·records+9 stops fitting at **16382 records**. A single
request for the whole history of sensor A returned no terminator at all, and delivered
less than was accumulated (≈46200 of ≈50100); why, and what the sensor does between 16382
and that — not established. **Read deep history in pages** of a few thousand records; the
end is the page that returned fewer than requested.

**Empty reply** is special: marker `fe fe fe 00…` and terminator 12. With `start > end`
there is no marker and the terminator is 9. A third kind also happens — zero records with
terminator 9 on an ordinary request (trap 2); why without the marker, not established.

**No live frames during a history read, apparently.** Sensor A: during a 34 s read of
50467 records the link got 9 measurements in 43 s — as many as fit in the 9 s outside the
read; a second sensor on another link got 37 in the same time. A host that watches the
live stream should not treat this pause as a dead link.

Throughput: 739 records in 491 ms; about 1500 records/s on deep reads (50467 records in
28–39 s on ESP32 with one link held).

### Depth

**At the limit the sensor erases the oldest 1024 records at once** and keeps logging.
Reads of sensor A:

| when | records | oldest |
|---|---|---|
| 25 Sep 2026 01:48 | 50106 | 17 Jul 11:38 |
| 25 Sep 2026 13:50 | 50467 | 17 Jul 11:38 |
| 25 Sep 2026 17:58 | 50590 | 17 Jul 11:38 |
| 26 Sep 2026 16:58 | 50257 | 18 Jul 21:46 |

Below the limit the count grows by one record per 2 minutes and the oldest record stays
put (361 over 43272 s against 360.6 expected; the first records of both dumps matched by
content). Between 25 Sep 17:58 and 26 Sep 16:58 the oldest record moved forward by
34 h 08 min — exactly 1024 records — and the count became 50257 instead of the expected
51280; the 2-minute step in the dump has no gaps. 1024 × 4 bytes = 4 KB, a flash erase
block — a plausible explanation, not verified.

**The limit is between 50591 and 51281 records; hypothesis: 51200** (50 blocks of 1024,
71.1 days). It fits every read above: sensor A's history started on 17 Jul 11:38 and would
have reached 51200 records on 26 Sep around 14:16. Test: the count climbs to 51200 and
drops to 50176 — due on sensor A around 28 Sep 00:20. You cannot read more than 65535
records (91 days) anyway — the index is 16-bit. The app's sync modes "31 days" and
"72 days" (51840 records) fit.

**Weeks and months while the sensor keeps power** — per the owner, who has opened the app
after 1–2 months and got the whole history. Measured: about 71 days on sensor A before the
first erase.

**After a power loss the history starts over** — seen once: sensor B, disconnected on day
one and powered from a bench supply two days later, returned history only from the moment
of reconnection, although the app's database has the earlier days.

### Traps

1. **Beyond the accumulated count the sensor returns raw memory, not a refusal.** A
   request `[N, N+1)` with N past the end returns a "record": at 2000 and 5000 — plausible
   12.46 and 12.48 V, at 60000 — 39.69 V (impossible), at 65000 — zeros. Take the boundary
   from the terminator; never probe indices.
2. **A single empty reply does not mean empty history.** One request out of fifteen
   returned zero records with an honest terminator 9, on a live link (0 drops in an hour)
   and full history; the next returned 728 records.
3. **A truncated capture does not look truncated.** With a buffer of 12 notifications the
   reply was cut mid-stream, and "440 records" looked plausible.

### `06` does NOT stop the transfer

Sent in the middle of a transfer — at 110 and 242 ms of a 491 ms transfer — it did not
stop it: packets kept coming (13 and 22 after the command). On its own it returns nothing.
What it does: not established.

### 5.6 Reading the whole history — a recipe that works

What our firmware does (ESP32, NimBLE), verified on sensor A:

1. Hold the link; send pages `[from, from+8000)`: 4·8000+9 = 32009 fits the 16-bit length.
2. Decrypt each notification **per 16-byte block**; append records.
3. On the terminator, set the count to `from + (LL−9)/4` — this drops the zero padding.
   If fewer bytes arrived than the terminator promised, the page is broken.
4. Fewer records than requested → that was the last page.
5. **Start each next page one record early** (`from − 1`). The start is then always inside
   the accumulated range — the verified case — and the overlapping record is a seam check:
   if the re-read record does not match, but the previous one is found one position
   further, a new record was written mid-read; drop the duplicate at the seam. If the
   records on both sides of the seam are identical, a shift cannot be seen and the older
   part stays 2 minutes off.
6. Store the local time of the read; record k is `t − k·120 s`.

On ESP32 with **two** sensor links held at once, a full read took either ~34 s or ~60 s
depending on how the links happened to be set up (connection interval 15 ms in both cases).
Releasing the other link for the duration of the read: 8 set-ups, 28–39 s, no slow ones.

---

## 6. Command `08` — battery type and the SoC table

**The state of charge is computed by the sensor, from a table the app writes into it.**
The ready percentage arrives in both the live frame and the history record.

### Type code

`d1 55 08 CODE`, where CODE is built from three app settings. Request builder checked
against the "Edit device" screen:

| choice in the app | `type` | `leadType` | `socType` | CODE | table |
|---|---|---|---|---|---|
| 12 V lead-acid | 1 | 1 | — | 1 | no |
| AGM | 1 | 2 | — | 2 | no |
| Custom + smart algorithm | 1 | 3 | 1 | 3 | yes |
| Custom + voltage ⇔ capacity | 1 | 3 | 2 | 4 | yes |
| Lithium, first subtype | 2 | 1 | — | 5 | no |
| Lithium custom + smart | 2 | 2 | 1 | 6 | yes |
| Lithium custom + voltage | 2 | 2 | 2 | 7 | yes |

The table is sent only for codes **3, 4, 6, 7**. Otherwise the command is four bytes.
The two-packet form is sent only to sensor firmware **≥ 1.7f**.

In "voltage ⇔ capacity" mode the percentage is a direct function of voltage through the
table. In "smart algorithm" mode it is smoothed somehow — not analysed.

### Table

Eleven thresholds, from 100 % down to 0 % in 10 % steps; the sensor **interpolates
linearly** between them.

```
d1 55 08 CODE 01 | 4fb 4f6 4ec 4e2 4d8 4ce      first six
d1 55 08 CODE 02 | 4c4 4ba 4b0 4a6 49c          last five
```

Each threshold is **three hex digits, /100 V**. The app uses `Integer.toHexString`
**without zero padding**, so three digits only come out between 2.56 and 40.95 V; outside
that range the layout breaks. The odd tail of the second packet is padded with a zero
nibble.

The app's default table (`getDefaultList`): 12.90, 12.80, 12.70, 12.60, 12.50, 12.40,
12.30, 12.20, 12.10, 12.00, 11.90.

Model vs live sensor (table 12.75…11.80):

| voltage | table | sensor showed |
|---|---|---|
| 12.00 V | 20 % | **20 %** |
| 12.08 V | 28 % | 27 % |
| 12.49 V | 69 % | **69 %** |
| 13.98 V | 100 % | **100 %** |

### Writing it — verified

Reversible experiment, sensor B at 13.68 V: writing a table shifted up by 1.25 V dropped
the reading from 100 % to **63 %** — exactly what the model predicted. Writing the
original frames back restored 100 %.

Second run, sensor B at 12.78 V, code 4, written by our own firmware without `01`/`02`:
the default table gave 88 %, the same table shifted up by 0.5 V gave 38 % — exactly the
linear interpolation; acknowledged both times.

- **Takes effect immediately**, from the next frame, no restart.
- **No `02` unlock needed** — we never sent it.
- **One acknowledgement `d1550800`, after the SECOND packet**; the first gets no reply.
- A history record logged while a foreign table is in place keeps the foreign percentage
  next to the correct voltage.

Safety net: the app sends `08` at the start of EVERY session, so its next launch overwrites
the table with its own.

Per the owner: in the app the thresholds constrain each other, so one edit cannot move
them far — hit the limit, save, repeat. Whether the sensor enforces the same: not checked.

---

## 7. Commands `01`, `03`, `04` — what is known

**`01` — version.** Reply `d1 55 01 01 08 00 04 00…`. The app turns it into a version
number; the meaning of each byte was not checked.

**`03` — cranking data.** The sensor records and stores the curve; the app only fetches
it. The reply is a series of notifications accumulated until the marker `ff ff fe`
(history uses `ff fe fe`), parsed into `CrankVoltBean`: a header where the age of the event
is counted in the same 2-minute units, then voltage samples — three hex digits each, /100 V,
as in the SoC table.

Not established: the **sample step** (milliseconds are plausible, but there is no constant
in the parser) and how many curves the sensor keeps.

**We could not provoke a cranking event on the bench.** Three sharp drops to 11 V with a
bench supply: `03` stayed empty, history showed 13.68–13.69 V with zero flags, the live
minimum was 12.49 V — the drop never reached 11 V (why was not measured). Worth knowing:
**history flags and the cranking buffer are separate things** — ten minutes at 12.0 V
produced stop/start flags but no curve.

**`04` — charging-system test**, three parts. Replies intermittently: two out of six
requests answered. `d1550402` returned zeros, `d1550403` — `d1 55 04 03 | 03 | 05 58`,
where `0x558` = 13.68 V, the current voltage. Format not decoded: nobody started an engine
on the bench.

---

## 8. How this was captured

**HCI log from an Android phone.** Developer options → "Enable Bluetooth HCI snoop log",
screen unlocked; restart the stack with `adb shell svc bluetooth disable && adb shell svc
bluetooth enable`; open the app; `adb bugreport out.zip`, the log is
`FS/data/log/bt/btsnoop_hci.log` inside.

`settings put global bluetooth_hci_log 1` is **not the switch**: the key becomes 1, the
stack does not read it. With the toggle off, the bugreport contains `btsnooz_hci.log`
(with a "z") — it has the `btsnoop\0` magic but no usable records; by name and magic it is
easy to mistake for the real one.

**APK.** `adb shell pm path com.dc.bm6` → `adb pull` → `java -jar apktool.jar d -o out
bm6.apk`. Reply parsing and request building are in
`smali/com/dc/bm6/ble/operation/BleDataOperation.smali`, the command table in
`smali/com/dc/bm6/mvp/model/Constants.smali`. Chinese debug strings label every command.

---

## Prior work

- [tarball.ca — Reverse engineering the BM6 BLE battery monitor][1]: the key, the
  characteristics, the live-measurement request, voltage and temperature.
- [JeffWDH/bm6-battery-monitor][2]: a reader for live voltage, temperature and SoC.

As far as we could see, neither covers the history (`05`/`06`), the SoC table (`08`),
link ownership or the status byte — that is what this document adds.

[1]: https://www.tarball.ca/posts/reverse-engineering-the-bm6-ble-battery-monitor/
[2]: https://github.com/JeffWDH/bm6-battery-monitor
