# AC Infinity BLE Decoder

Reverse-engineered Bluetooth Low Energy (BLE) decoder for AC Infinity smart thermo-hygrometers. Reads temperature, humidity, and battery level directly from BLE advertisement packets — **no official AC Infinity app required**.

> **Unofficial project.** Not affiliated with, endorsed by, or supported by AC Infinity Inc. AC Infinity is a trademark of AC Infinity Inc. Provided as-is, for personal/hobby use.

## Status

| Model | Status |
|---|---|
| **B2** (Smart Thermo-Hygrometer, integrated sensor) | ✅ Decoded and verified |
| **B1** (Smart Thermo-Hygrometer, 12' probe) | ❓ Untested — likely same protocol, needs confirmation |
| **A1** (Mini Thermo-Hygrometer, 12' probe) | ❓ Untested |
| **A2** (Mini Thermo-Hygrometer, integrated sensor) | ❓ Untested |

**If you own an A1, A2, or B1 device, testing help is very welcome** — see [Contributing](#contributing) below.

<img width="957" height="781" alt="image" src="https://github.com/user-attachments/assets/15b98139-5cba-4a8d-bb75-3e2d0f8fed34" />


## How it works

The device broadcasts a BLE advertisement packet with manufacturer-specific data under company ID `0x0902` (bytes `02 09`, little-endian). No connection/pairing is needed — the data can be read passively by any nearby BLE scanner, which means multiple listeners (e.g. this decoder *and* the official AC Infinity app) can read it simultaneously without conflict.

Packet layout (31 bytes total, standard AD structure):

```
1E FF | 02 09 | [6 bytes: MAC, reversed] | [21 bytes: payload]
```

Payload fields used by this decoder:

| Field | Bytes | Encoding |
|---|---|---|
| Battery (%) | `payload[3]` | uint8 |
| Temperature (°C) | `payload[9]`, `payload[10]` (high nibble) | 12-bit: `(b9 << 4) \| (b10 >> 4)`, then `/ 10.0` |
| Humidity (%) | `payload[10]` (low nibble), `payload[11]` | 12-bit: `((b10 & 0x0F) << 8) \| b11`, then `/ 10.0` |
| VPD (kPa) | `payload[2]` | Suspected 0.01 kPa units — **unverified, not used** |
| Rolling counter | `payload[20]` | Ignored |

## Verification

Decoded values were cross-checked against a 21-minute CSV export from the official AC Infinity app (temperature drop from 21°C to 14.5°C):

- Regression slope: **0.994** (temperature), **1.000** (humidity)
- Mean deviation: **-0.03°C** / **-0.09%**

Humidity readings show occasional transient spikes of ~±1% within a 5-second cycle — this is a device characteristic, not a decoder bug.

## Usage (Kotlin / Android)

```kotlin
// valueBytes = whatever ScanRecord.getManufacturerSpecificData(0x0902) returns
val result = AcInfinityDecoder.decode(valueBytes)

if (result != null) {
    val (temperature, humidity, battery) = result
    println("Temp: $temperature°C, Humidity: $humidity%, Battery: ${battery ?: "unknown"}%")
}
```

```kotlin
val scanCallback = object : ScanCallback() {
    override fun onScanResult(callbackType: Int, result: ScanResult) {
        val data = result.scanRecord?.getManufacturerSpecificData(AcInfinityDecoder.COMPANY_ID)
            ?: return
        AcInfinityDecoder.decode(data)?.let { (temp, hum, batt) ->
            // handle reading
        }
    }
}
```

## Contributing

If you have an **A1, A2, or B1** device:

1. Run a BLE scanner app (or this decoder, if you have a test build) near your device.
2. Check whether readings match what the official AC Infinity app shows.
3. If the format differs, capturing raw manufacturer-specific bytes (company ID `0x0902`) alongside the known-correct temp/humidity from the app would let the decoder be extended — a PR or an issue with that raw data is very welcome.

No BLE sniffing tools are required if the decoder produces plausible values — a simple side-by-side comparison with the official app is enough.

## License

MIT — see [LICENSE](LICENSE).
