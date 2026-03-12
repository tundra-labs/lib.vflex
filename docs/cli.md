---
title: CLI Tool
parent: lib.vflex.app
nav_order: 9
---

# CLI Tool

`vflex-cli` is a command-line tool for configuring and reading VFLEX USB-PD devices from Node.js. It uses the `lib-vflex` library and communicates over MIDI.

## Install

```bash
npm install -g vflex-cli
```

After installing globally, the `vflex-cli` command is available from any terminal.

## Usage

```
vflex-cli [options]
```

You can also run without installing globally using npx:

```bash
npx vflex-cli --status
```

## Commands

### Device Info

```bash
vflex-cli --status              # Print full device status and configuration
vflex-cli --measure             # Read current voltage measurement (raw ADC + mV)
vflex-cli --pdo-log             # Read and display full PDO log
```

### Get Parameters

```bash
vflex-cli --get-voltage         # Get configured output voltage (mV)
vflex-cli --get-current-limit   # Get current limit (mA)
vflex-cli --get-adc-offset      # Get ADC offset calibration value
vflex-cli --get-adc-scale       # Get ADC scale calibration value
vflex-cli --get-tol-nominal     # Get nominal voltage tolerance (mV)
vflex-cli --get-tol-sag         # Get sag tolerance value
vflex-cli --get-vlimit          # Get voltage operating window (low/high mV)
```

### Set Parameters

```bash
vflex-cli --set-voltage <mV>              # Set target output voltage in millivolts
vflex-cli --set-current-limit <mA>        # Set current limit in milliamps
vflex-cli --set-adc-offset <val>          # Set ADC offset calibration value
vflex-cli --set-adc-scale <val>           # Set ADC scale calibration value
vflex-cli --set-tol-nominal <mV>          # Set nominal voltage tolerance in mV
vflex-cli --set-tol-sag <val>             # Set sag tolerance value
vflex-cli --set-vlimit <low_mV> <high_mV> # Set voltage operating window
```

### LED Control

```bash
vflex-cli --led <color>
```

Available colors: `off`, `red`, `green`, `blue`, `white`, `yellow`, `magenta`, `cyan`

### Utilities

```bash
vflex-cli --list-midi           # List available MIDI devices
vflex-cli --help                # Show help message
```

## Chaining Commands

Multiple commands can be combined in a single invocation. They execute sequentially from left to right.

```bash
vflex-cli --set-voltage 12000 --get-voltage --measure
vflex-cli --led green --status
vflex-cli --set-vlimit 3000 20000 --get-vlimit
```

## Examples

Set voltage to 9V and verify:

```bash
vflex-cli --set-voltage 9000 --measure
```

Read all device info:

```bash
vflex-cli --status
```

Example output:

```
=== VFLEX Device Status ===
  Serial Number:    VF-00123
  Hardware ID:      HW-2.1
  Firmware Version: FW-1.4.2
  Mfg Date:         2025-01-15
  Voltage:          5000 mV
  Current Limit:    3000 mA
  V-Limit Low:      3300 mV
  V-Limit High:     20000 mV
  Tol Nominal:      200 mV
  Tol Sag/mA:       5
  ADC Offset:       0
  ADC Scale:        1000
```

## Requirements

- Node.js 16 or later
- VFLEX device connected via USB
