# Gotcha #13: image printing over Bluetooth is unreliable on the MTP-II/MP58C7 family — use Serial

Confirmed on a real MP58C7 (sold under several rebrands of the same
Xiamen Hanin Electronic Technology hardware — MTP-II, HPRT HM-A200U,
PixPos MP58C7; `bluetooth/profiles.ts`'s existing `MTP-II` profile
already targets this device family): printing an image over Bluetooth
comes out as banded/sheared garbage, while text, barcode and QR print
perfectly over the same connection. Exhaustively ruled out before
landing on this:

- `imageMode: 'column'` vs `'raster'` — identical corruption in both,
  so it isn't the ESC/POS command dialect.
- BLE write chunk size (`messageSize`: 100 default, 20, 8) and
  inter-write delay (`sleepAfterCommand`: 0, 10, 20ms), individually and
  combined — none fixed it; some combinations made it visibly worse.
- Payload size — even a ~256-byte synthetic stripe-pattern image
  corrupts the same way as a full-size photo.
- The printer's own graphics engine isn't at fault: the *exact same*
  encoded bytes (`column` or `raster`, doesn't matter) print perfectly
  when sent over Serial/USB instead of Bluetooth.

**Confirmed fix: use `transport: 'serial'` (or `'usb'`) instead of
Bluetooth for any print job containing an image on this printer family.**
This isn't a library-side bug to patch: Serial/USB send the encoded
bytes as one continuous write with no artificial chunking
(`SerialTransport.ts`/`UsbTransport.ts`), while Bluetooth GATT writes
are always chunked through `writeChunked.ts`. The corruption most
likely originates in this printer's separate BLE-to-serial bridge chip
— common on cheap clones — whose buffer/flow-control couldn't be tuned
into working from the browser side across every chunk-size/pacing
combination tried.

No code-level fix — same shape as gotcha #11 ("no code-level fix,
prefer Serial"). Note that `bluetooth/profiles.ts`'s existing `MTP-II`
profile's `messageSize: 20, sleepAfterCommand: 20` was added for a
*disconnect* mid-print, a different failure mode on the same hardware
family — it doesn't fix this image-corruption issue, so don't assume
that profile makes Bluetooth image printing safe on this device.

---
[AGENTS.md](../../AGENTS.md) gotcha #13.
