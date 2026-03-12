# lib.vflex.app

## What is VFLEX?

VFLEX is a universal USB-C power adapter that converts any USB-C Power Delivery charger into a configurable power solution. Set the exact voltage your device needs, plug into a USB-C PD charger or power bank, and power everything from legacy electronics to modern gear. It supports standard (SPR) and extended (EPR) power ranges, including PPS and AVS modes.

## Get the hardware

The **VFLEX Base** is available for $8.00 USD from Werewolf.

- [Buy VFLEX Base](https://werewolf.us/products/vflex-base)
- [Datasheet](https://werewolf.us/vflex/base/datasheet)
- [User Manual](https://werewolf.us/vflex/user-manual)

## Official VFLEX App

Looking for a full-featured app? The official VFLEX app includes additional features beyond this developer library and is available on web, iOS, and Android.

- [VFLEX Web App](https://vflex.app)
- [iOS App](https://werewolf.us/vflex-ios)
- [Android App](https://werewolf.us/vflex-android)

## About this library

To ensure the long term viability of VFLEX and encourage community development, we're sharing `lib.vflex.js`. This JavaScript library documents the VFLEX communication protocol and provides a MIDI interface for configuring voltage, current, and reading device diagnostics — from the browser or from Node.js. We love when people build cool stuff with our hardware!

- [Source Code](https://github.com/tundra-labs/lib.vflex)
- [Configuration Tool](https://lib.vflex.app/vflex-tool.html)
- [Debug Tool](https://lib.vflex.app/vflex-debug.html)

### Install via npm

```bash
npm install lib-vflex
```

```js
const { VFLEX, VFLEX_COMMANDS } = require('lib-vflex');
```

Or include directly in a browser:

```html
<script src="lib.vflex.js"></script>
```

## CLI Tool

A standalone command-line tool is available for quick configuration and testing from the terminal.

### Install

```bash
npm install -g vflex-cli
```

### Usage

```bash
vflex-cli --status                          # Print full device status
vflex-cli --measure                         # Read voltage measurement
vflex-cli --pdo-log                         # Read PDO log
```

#### Get Parameters

```bash
vflex-cli --get-voltage                     # Get configured voltage
vflex-cli --get-current-limit               # Get current limit
vflex-cli --get-adc-offset                  # Get ADC offset calibration value
vflex-cli --get-adc-scale                   # Get ADC scale calibration value
vflex-cli --get-tol-nominal                 # Get nominal voltage tolerance
vflex-cli --get-tol-sag                     # Get sag tolerance value
vflex-cli --get-vlimit                      # Get voltage operating window
```

#### Set Parameters

```bash
vflex-cli --set-voltage <mV>                # Set target output voltage in millivolts
vflex-cli --set-current-limit <mA>          # Set current limit in milliamps
vflex-cli --set-adc-offset <val>            # Set ADC offset calibration value
vflex-cli --set-adc-scale <val>             # Set ADC scale calibration value
vflex-cli --set-tol-nominal <mV>            # Set nominal voltage tolerance in mV
vflex-cli --set-tol-sag <val>               # Set sag tolerance value
vflex-cli --set-vlimit <low_mV> <high_mV>   # Set voltage operating window
```

#### LED Control

```bash
vflex-cli --led <color>
```

Available colors: `off`, `red`, `green`, `blue`, `white`, `yellow`, `magenta`, `cyan`

#### Utilities

```bash
vflex-cli --list-midi                       # List available MIDI devices
vflex-cli --help                            # Show help message
```

Commands can be chained in a single invocation:

```bash
vflex-cli --set-voltage 12000 --get-voltage --measure
```

## MIDI Communication Protocol

### Connection

The library uses MIDI to discover and connect to VFLEX devices. In the browser, it uses the [Web MIDI API](https://developer.mozilla.org/en-US/docs/Web/API/Web_MIDI_API) (`navigator.requestMIDIAccess()`) with `tryConnect()`. In Node.js, use `connectWithPorts(input, output)` to supply MIDI ports from any MIDI backend (e.g. [JZZ](https://www.npmjs.com/package/jzz)).

The library scans for a port whose name contains **"vflex"** (case-insensitive). Both an input (for receiving responses) and an output (for sending commands) must be found for a successful connection.

### Framing Protocol

MIDI messages are limited to 7-bit data values per byte. The VFLEX protocol works around this by splitting each 8-bit data byte into two 7-bit MIDI data fields (high nibble and low nibble), and using standard MIDI status bytes as framing delimiters:

| MIDI Status Byte | Meaning         | Description |
|-------------------|-----------------|-------------|
| `0x80` (Note Off) | **Start frame** | Signals the beginning of a new message. Data bytes are ignored. |
| `0x90` (Note On)  | **Data byte**   | Carries one byte of payload. `d1` = high nibble, `d2` = low nibble. The original byte is reconstructed as `(d1 << 4) \| d2`. |
| `0xA0` (Aftertouch)| **End frame**   | Signals the end of the message. Triggers response processing. |

A 20 ms delay is inserted between each MIDI packet to ensure reliable delivery.

### Command Packet Structure

Each command sent to the device is a byte array with the following structure:

| Byte Index | Field     | Description |
|------------|-----------|-------------|
| 0          | Length    | Total length of the packet (preamble + payload) |
| 1          | Command   | Command ID with optional flag bits (see below) |
| 2+         | Payload   | Command-specific data (variable length) |

**Command byte flags:**

| Bit   | Mask   | Meaning |
|-------|--------|---------|
| Bit 7 | `0x80` | **Write** — set when writing a value to the device |
| Bit 6 | `0x40` | **Scratchpad** — set for scratchpad (temporary) writes |
| Bits 0-5 | `0x3F` | **Command ID** — the actual command identifier |

### Response Handling

Responses from the device use the same framing protocol. The library reassembles incoming data bytes until an end frame (`0xA0`) is received, then parses the response. The command ID in the response (masked to 6 bits) is matched against the last sent command to generate an acknowledgment.

The library uses a polling-based ACK mechanism with a configurable timeout (default 1500 ms for commands).

## Commands

| Command | ID | Read Response Format | Description |
|---------|----|----------------------|-------------|
| `CMD_SERIAL_NUMBER` | 8 | UTF-8 string | Device serial number |
| `CMD_HARDWARE_ID` | 10 | UTF-8 string | Hardware revision identifier |
| `CMD_FIRMWARE_VERSION` | 11 | UTF-8 string | Firmware version string |
| `CMD_MFG_DATE` | 12 | UTF-8 string | Manufacturing date |
| `CMD_FLASH_LED_SEQUENCE_ADVANCED` | 13 | Write: `[10, 1, color, 2, 0]` | LED color control |
| `CMD_PDO_LOG` | 17 | Chunked binary (see below) | USB PD Power Data Object log |
| `CMD_VOLTAGE_MV` | 18 | uint16 big-endian | Configured output voltage in millivolts |
| `CMD_CURRENT_LIMIT_MA` | 19 | uint16 big-endian | Current limit in milliamps |
| `CMD_RESERVED_A` | 22 | `[subcmd, level]` | Authorization lock level |
| `CMD_USER_VLIMIT` | 23 | `[high_msb, high_lsb, low_msb, low_lsb]` | User voltage limits (high and low) in mV |
| `CMD_VTOLERANCE_NOMINAL_MV` | 24 | uint16 big-endian | Voltage tolerance nominal value in mV |
| `CMD_VTOLERANCE_SAG_PER_MA` | 25 | uint16 big-endian | Voltage sag tolerance per mA |
| `CMD_VMEASURE_ADC_COUNT_OFFSET` | 26 | int32 big-endian (signed) | ADC count offset calibration |
| `CMD_VMEASURE_ADC_COUNT_SCALE` | 27 | int32 big-endian (signed, milli-units) | ADC count scale calibration |
| `CMD_VMEASURE` | 28 | `[raw_msb, raw_lsb, v_mv_msb, v_mv_lsb]` | Live voltage measurement (raw ADC + mV) |

### LED Colors

| Color | Value |
|-------|-------|
| off | 0 |
| red | 1 |
| green | 2 |
| blue | 3 |
| white | 4 |
| yellow | 5 |
| magenta | 6 |
| cyan | 7 |

### PDO Log

The PDO log is retrieved in 12 chunks (requested sequentially with chunk IDs 0-11). Each chunk carries 8 bytes of payload, producing a 90-byte record once assembled. The record contains:

- Target and measured voltage
- Number of PDOs received and selected PDO index
- Status flags for USB PD negotiation (SPR, EPR, PPS states)
- Up to 20 raw PDO entries (parsed per USB PD specification into Fixed, Battery, Variable, and Augmented/PPS/AVS types)

## Quick Start

### Browser

```html
<script src="lib.vflex.js"></script>
<script>
  const vflex = new VFLEX();

  async function run() {
    await vflex.tryConnect();

    // Read device info
    await vflex.getString(VFLEX_COMMANDS.CMD_SERIAL_NUMBER);
    console.log("Serial:", vflex.device_data.serial_num);

    // Read voltage
    await vflex.getVoltageMv();
    console.log("Voltage:", vflex.device_data.voltage_mv, "mV");

    // Get full PDO log
    const result = await vflex.getFullPdoLog();
    console.log(result.output);
  }

  run();
</script>
```

### Node.js

```bash
npm install lib-vflex jzz
```

```js
const JZZ = require('jzz');
const { VFLEX, VFLEX_COMMANDS } = require('lib-vflex');

async function run() {
  const jzz = await JZZ();
  const info = jzz.info();

  const inputInfo = info.inputs.find(p => p.name.toLowerCase().includes('vflex'));
  const outputInfo = info.outputs.find(p => p.name.toLowerCase().includes('vflex'));

  const midiOut = await jzz.openMidiOut(outputInfo.name);
  const midiIn  = await jzz.openMidiIn(inputInfo.name);

  // Wrap JZZ ports to match the lib.vflex.js port interface
  const wrappedInput = {
    _cb: null,
    set onmidimessage(cb) {
      this._cb = cb;
      if (cb) {
        midiIn.connect(function (msg) {
          cb({ data: [msg[0], msg[1], msg[2]] });
        });
      }
    },
    get onmidimessage() { return this._cb; },
  };

  const wrappedOutput = {
    send(data) { midiOut.send(data); },
  };

  const vflex = new VFLEX();
  vflex.connectWithPorts(wrappedInput, wrappedOutput);

  await vflex.getString(VFLEX_COMMANDS.CMD_SERIAL_NUMBER);
  console.log("Serial:", vflex.device_data.serial_num);

  await vflex.getVoltageMv();
  console.log("Voltage:", vflex.device_data.voltage_mv, "mV");

  vflex.disconnect();
  jzz.close();
}

run();
```

## API Reference

### `new VFLEX()`

Creates a new instance. Set `vflex.logLevel` to `'silent'`, `'info'`, `'warn'`, or `'error'` to control console output.

### Connection

- **`tryConnect()`** — Browser only. Requests MIDI access via the Web MIDI API and connects to the first VFLEX device found. Throws if no device is available or if called in Node.js.
- **`connectWithPorts(input, output)`** — Manual connection with user-supplied MIDI ports. Works in any environment (Node.js, browser, etc.). The `input` must accept `onmidimessage = callback` where callback receives `{ data: [status, d1, d2] }`. The `output` must have a `send([status, d1, d2])` method.
- **`disconnect()`** — Disconnects from the device and clears MIDI port references.

### Read Methods

All read methods are async. After awaiting, the result is available on `vflex.device_data`.

| Method | Result Property |
|--------|-----------------|
| `getString(CMD_SERIAL_NUMBER)` | `device_data.serial_num` |
| `getString(CMD_HARDWARE_ID)` | `device_data.hw_id` |
| `getString(CMD_FIRMWARE_VERSION)` | `device_data.fw_id` |
| `getString(CMD_MFG_DATE)` | `device_data.mfg_date` |
| `getVoltageMv()` | `device_data.voltage_mv` |
| `getMaxCurrentMa()` | `device_data.max_current_ma` |
| `getAuthLockLevel()` | `device_data.authlock_level` |
| `getUserVLimit()` | `device_data.vlimit_high_mv`, `device_data.vlimit_low_mv` |
| `getVToleranceNominalMv()` | `device_data.vtolerance_nominal_mv` |
| `getVToleranceSagPerMa()` | `device_data.vtolerance_sag_per_ma` |
| `getVMeasureAdcCountOffset()` | `device_data.vmeasure_adc_count_offset` |
| `getVMeasureAdcCountScale()` | `device_data.vmeasure_adc_count_scale` |
| `getVMeasure()` | `device_data.vmeasure_raw_adc`, `device_data.vmeasure_voltage_mv` |
| `getFullPdoLog()` | Returns `{ logData, parsed_pdos, output }` with parsed PDO information |

### Write Methods

Write operations use `sendCommand()` with `write=true` and a payload. All values are big-endian.

| Operation | Example |
|-----------|---------|
| Set voltage (mV) | `sendCommand(CMD_VOLTAGE_MV, uint16BE(9000), true)` |
| Set current limit (mA) | `sendCommand(CMD_CURRENT_LIMIT_MA, uint16BE(3000), true)` |
| Set voltage limits | `sendCommand(CMD_USER_VLIMIT, [high_msb, high_lsb, low_msb, low_lsb], true)` |
| Set tolerance nominal (mV) | `sendCommand(CMD_VTOLERANCE_NOMINAL_MV, uint16BE(val), true)` |
| Set tolerance sag | `sendCommand(CMD_VTOLERANCE_SAG_PER_MA, uint16BE(val), true)` |
| Set ADC offset | `sendCommand(CMD_VMEASURE_ADC_COUNT_OFFSET, int32BE(val), true)` |
| Set ADC scale | `sendCommand(CMD_VMEASURE_ADC_COUNT_SCALE, int32BE(val), true)` |
| Set LED color | `sendCommand(CMD_FLASH_LED_SEQUENCE_ADVANCED, [10, 1, color, 2, 0], true)` |
| Clear PDO log | `clearPdoLog()` |

### Low-Level

- **`sendCommand(cmd, payload, write, scratchpad, expectAck)`** — Sends a command with optional payload and flag bits. Set `write=true` for write operations, `scratchpad=true` for temporary writes.
- **`sendRaw(data)`** — Sends raw bytes using the MIDI framing protocol.

## Platform Support

### Browser

Requires a browser with [Web MIDI API](https://developer.mozilla.org/en-US/docs/Web/API/Web_MIDI_API) support (Chrome, Edge, Opera). Firefox and Safari do not currently support Web MIDI.

### Node.js

Use `connectWithPorts(input, output)` with any MIDI backend. [JZZ](https://www.npmjs.com/package/jzz) is recommended. Requires Node.js 16 or later.
