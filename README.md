# Smart Oil Gauge Duo: local Python monitoring notes

## Goal and result

Build a Python monitor that reads this owner's Connected Consumer Fuel Smart Oil Gauge Duo (CCF-903) uploads locally, stores measurements, and calculates gallons without querying the company's account API.

We identified the gauge, captured its uploads through an Ubuntu Wi-Fi hotspot, and found a binary field that closely tracks the company's displayed gallons. With a tank-specific calibration, our estimates agree with the supplied app readings to about 0.01 gallon. This is empirical agreement near the current tank level, not a claim of that much physical measurement accuracy.

## How we found it

The Ubuntu computer uses Ethernet (`eno1`) for internet and its Wi-Fi adapter (`wlp0s20f3`) as a temporary 2.4 GHz WPA2 hotspot. NetworkManager's `ipv4.method shared` supplies DHCP and routes hotspot traffic through Ethernet.

The gauge was configured to join that hotspot. This puts Ubuntu directly in its traffic path; capturing from an ordinary Ethernet client on the original LAN could see broadcasts but not reliably see the gauge's cloud uploads.

Observed device identity:

- Model: CCF-903, with an Espressif ESP-WROOM-02 module.
- MAC: `bc:ff:4d:d3:e7:db`.
- DHCP hostname: `ESP-D3E7DB`.
- Original LAN address: `192.168.86.233`.
- Hotspot address during testing: `10.42.0.216` (do not hard-code this; DHCP can change it).

Its upload is an ordinary, unencrypted HTTP POST to `data.connectedconsumerfuel.com`, path `/upload_903.php`, TCP port 80. The observed destination IP was `23.235.203.195`; use the hostname or traffic identity rather than assuming that IP is permanent.

We did not find or establish a local polling API. The proposed monitor observes outgoing uploads instead.

## Capture workflow

Keep the hotspot active and Ubuntu awake. Capture only the gauge's HTTP traffic:

```bash
sudo timeout --signal=INT 24h tcpdump -ni wlp0s20f3 -s 0 -U \
  -w "/tmp/oil-gauge-$(date +%Y%m%d-%H%M%S).pcap" \
  'ether host bc:ff:4d:d3:e7:db and tcp port 80'
```

Use a new filename for each run. The command stops after 24 hours; Ctrl+C stops it sooner. For continuous monitoring, use a managed capture process and rotating files rather than this diagnostic command.

Inspect a saved file:

```bash
# Requests and upload times
sudo tcpdump -nn -tttt -r CAPTURE.pcap 'tcp dst port 80'

# Request bytes, including the binary measurements
sudo tcpdump -nn -tttt -X -r CAPTURE.pcap 'tcp dst port 80'

# Server responses and any configuration text
sudo tcpdump -nn -tttt -A -r CAPTURE.pcap 'tcp src port 80'
```

Treat captures as private device data. The multipart boundary `123456789` is a separator, not a password.

## Upload format

The HTTP body is `multipart/form-data`. Extract the part named `binary_data` using the boundary declared in the HTTP Content-Type header. Do not use absolute packet offsets: headers vary, and a future request could span multiple TCP packets.

In every supplied sample, the binary part consists of a 14-byte header followed by one or more 8-byte records. Single-reading uploads contain 22 binary bytes; four-record uploads contain 46. HTTP Content-Length also includes multipart framing, so it is not the binary payload length.

Header offsets are zero-based within `binary_data`:

| Offset | Length | Observed meaning |
|---|---:|---|
| 0 | 4 | Stable bytes `db e7 d3 42`; first three match the MAC suffix in reverse order. Preserve as an opaque identifier. |
| 4 | 4 | Little-endian value 0 or 1. Value 1 correlates with configuration-bearing server responses; exact semantics unconfirmed. |
| 8 | 4 | Stable bytes `be 60 01 00`; unknown. |
| 12 | 2 | Little-endian 903, consistent with the device model. |
| 14 onward | 8 each | Measurement records. |

Each record:

| Relative offset | Length | Interpretation |
|---|---:|---|
| 0 | 2 | Unsigned little-endian ultrasonic timing; adopt microseconds of round-trip travel as the working interpretation. |
| 2 | 2 | Unsigned little-endian unknown field, observed around 91–100. Do not label it inches. |
| 4 | 4 | Unknown bytes. Preserve exactly; could contain packed measurements or status. |

The Duo has two sensors. We have not identified whether or how the second sensor is represented.

Minimal decoder after HTTP/multipart extraction:

```python
import struct

def decode_binary_data(payload: bytes):
    if len(payload) < 22 or (len(payload) - 14) % 8:
        raise ValueError("Unexpected CCF-903 payload layout")
    records = []
    for offset in range(14, len(payload), 8):
        raw = payload[offset:offset + 8]
        timing_us, unknown_u16 = struct.unpack_from("<HH", raw)
        records.append({
            "record_index": len(records),
            "timing_us": timing_us,
            "unknown_u16": unknown_u16,
            "unknown_tail_hex": raw[4:].hex(),
            "raw_record_hex": raw.hex(),
        })
    return {"header_hex": payload[:14].hex(), "records": records}
```

## Distance and gallons calculation

Working physical model: the first record field is ultrasonic round-trip time in microseconds. At an assumed sound speed of 343 m/s:

```python
gap_inches = timing_us * 343 / 50800
```

This includes division by two for the outbound and return journey. Example: 1455 gives 9.824 inches; 1461 gives 9.865 inches. We have not decoded temperature compensation.

The owner's measurements were 44 inches of internal tank height and 34.5 inches of oil, implying 9.5 inches of air to the tank top. The sensor face can have a different reference height. The manually measured depth and the app/chart estimate did not exactly agree, so the final conversion is explicitly calibrated to the app, not an independent absolute-volume calibration.

The supplied tank chart gives 228 gallons at 35 inches and 235 gallons at 36 inches: locally, 7 gallons per inch. The calibration anchor is raw timing 1455 with an app reading of 229.69 gallons.

**Use this formula near the current oil level:**

```python
def estimated_gallons(timing_us: int) -> float:
    gallons_per_microsecond = 7 * 343 / 50800
    return 229.69 - (timing_us - 1455) * gallons_per_microsecond
```

Equivalently:

`gallons = 229.69 - (timing_us - 1455) × 0.047263779527559`

The equivalent effective sensor height is approximately 45.0656 inches above the bottom:

```python
effective_sensor_height = (35 + (229.69 - 228) / 7) + (1455 * 343 / 50800)
oil_depth_inches = effective_sensor_height - gap_inches
```

That height is a fitted calibration parameter, not a measured mounting dimension. Keep full precision internally; round gallons to two decimals only for display.

| Raw timing | Calculated gallons | Supplied app comparison |
|---:|---:|---:|
| 1454 | 229.74 | 229.74 |
| 1455 | 229.69 | 229.69, calibration anchor |
| 1460 | 229.45 | Not separately matched |
| 1461 | 229.41 | 229.42 at 21:31 and 22:32 |
| 1462 | 229.36 | 229.37 reported at 13:30 |

The two single-record uploads at 21:31 and 22:32 provide the cleanest paired comparison. Individual timestamp assignments in older batches remain unverified.

Do not extrapolate the constant 7-gallons-per-inch slope across the whole tank. For a full-range app, obtain the complete chart matching the actual tank orientation and interpolate volume from oil depth. The owner called it a horizontal 275-gallon tank, but measured a 44-inch internal height; resolve geometry from the actual tank/chart rather than that description alone. Until then, label volumes outside the calibrated chart range unsupported.

## Upload timing and batching

The manufacturer describes hourly measurement with uploads grouped according to an interval in hours. Captures support this:

- 12:30 → 16:31 → 20:31: approximately four hours apart, four records each.
- At 20:31 the server returned `upload_interval: 1 latest version: 903-02`.
- Subsequent uploads at 21:31 and 22:32 contained one record each.

Configuration is therefore not limited to initial setup. Some uploads received an empty HTTP 200 response; others received interval/version text. We observed the header field at offset 4 correlate with this difference, but have not established the configuration-request schedule.

Preserve upload time and record index separately. Do not assume records are oldest-first, assign exact hourly timestamps to batch records, or assume the final record is always the latest. For single-record uploads, upload time is a useful observed timestamp, though not a proven exact sensor measurement time.

## Python implementation direction

1. Start with an offline PCAP importer, then add live capture through the Ubuntu hotspot.
2. Reassemble TCP streams and parse complete HTTP requests. Deduplicate TCP retransmissions within a connection; do not discard identical readings from separate legitimate uploads.
3. Extract `binary_data`, validate its length, and retain unknown layouts for inspection rather than silently misdecoding them.
4. Store uploads and child records in SQLite: upload timestamp with timezone, MAC, connection identity, raw payload, header, record index, raw timing, unknown bytes, calculated distance, estimated gallons, and calibration version.
5. Store server interval/version responses separately. Record freshness based on uploads, and distinguish missing data from zero consumption.
6. Preserve raw values so calibration and field interpretations can change without losing history. Small increases in estimated gallons should remain visible; do not automatically call them deliveries or force the series to decrease.
7. Test against the existing single-record and four-record captures, including duplicate response packets and varying HTTP headers.

The initial monitor should be passive: let the gauge continue talking to its server while collecting its uploads locally. No portal credentials or paid API subscription are required for this capture path. However, this arrangement still allows cloud communication; fully removing the cloud would require a separately tested local receiver and a correct understanding of the response/configuration behavior.

The GitHub integration previously reviewed (`bergb91-coder/ha-smartoilgauge`) logs into the company's portal and retrieves processed tank data. It does not decode this binary protocol and is not a local-device solution.
