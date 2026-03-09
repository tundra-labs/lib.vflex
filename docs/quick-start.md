---
title: Quick Start
parent: VFLEX Public API
nav_order: 3
---

# Quick Start

```html
<script src="vflex-lib.js"></script>
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
