# The Things Network (TTN) - Tektelic CLOVER Configuration Notes

This document details the configuration for the **TEKTELIC CLOVER Agriculture Sensors** on The Things Network (TTN v3) for the UCL200 Bio-Connected Canopy project.

---

## 1. Background & Why a Custom Formatter is Needed

By default, selecting **"Use Device Repository"** in TTN loads Tektelic's generic agricultural codec (`decoder_agriculture_sensor.js`). 

While this default codec parses ambient temperature, relative humidity, and light intensity, **it does not calculate soil metrics for the CLOVER surface version**:
* **`input1_frequency`** is left as a raw resonant frequency in kHz rather than converting to **Soil Moisture (% Gravimetric Water Content)**.
* **`input2_voltage`** is left as a raw voltage in mV rather than converting to **Soil Temperature (°C)** via the probe's thermistor curve.

By switching the Uplink Payload Formatter in TTN to a **Custom Javascript Formatter**, TTN will perform these conversions on the fly. Telegraf and InfluxDB will then automatically receive calibrated `soil_temperature` and `soil_moisture_gwc` fields with no configuration changes required.

---

## 2. TTN Setup Instructions

To manage this cleanly across all 4 sensors (`tektelic-a` through `tektelic-d`), apply the formatter once at the **Application level**:

### Step 1: Set End Devices to Inherit Application Default
1. In the TTN Console, open **Applications > UCL200 Tektelic > End devices**.
2. For each of the 4 devices (`tektelic-a-2145a0081`, `tektelic-b-2145a0102`, `tektelic-c-2145a0117`, `tektelic-d-2145a0080`):
   * Open the device.
   * Go to **Payload formatters > Uplink**.
   * Select **"Use application default"**.
   * Click **Save changes**.

### Step 2: Configure the Application-Level Payload Formatter
1. Navigate to **Applications > UCL200 Tektelic**.
2. In the left navigation menu, click **Payload formatters > Uplink**.
3. Set **Formatter type** to: **Custom Javascript formatter**.
4. Paste the decoder code below into the code editor.
5. Click **Save changes**.

---

## 3. Custom Uplink Payload Formatter Code

Copy and paste this code into the TTN **Custom Javascript formatter** window:

```javascript
/**
 * TEKTELIC CLOVER Agriculture Sensor Uplink Payload Decoder
 * UCL200 Bio-Connected Canopy Project
 * 
 * Decodes Port 10 telemetry:
 *  - Ambient Temperature (°C) & Relative Humidity (%)
 *  - Light Intensity (lux)
 *  - Battery Voltage (V) & End-of-Life Alert
 *  - Accelerometer X/Y/Z (G)
 *  - Soil Temperature (°C) [converted from input2_voltage mV]
 *  - Soil Moisture (% GWC)  [converted from input1_frequency kHz]
 */
function decodeUplink(input) {
  var bytes = input.bytes;
  var port = input.fPort;
  var decoded = {};
  var warnings = [];
  var errors = [];

  if (port === 10) {
    for (var i = 0; i < bytes.length; ) {
      var channel = bytes[i++];
      var type = bytes[i++];

      // 0x00 0xBA: Battery Status (1 byte)
      if (channel === 0x00 && type === 0xba) {
        var raw_batt = bytes[i++];
        decoded.battery_voltage = Number(((raw_batt & 0x7f) * 0.01 + 2.5).toFixed(2));
        decoded.battery_eos_alert = (raw_batt >> 7) & 0x01;
      }
      // 0x01 0x04: Input 1 Frequency (2 bytes, unsigned kHz) - Soil Moisture Transducer
      else if (channel === 0x01 && type === 0x04) {
        var freq = (bytes[i++] << 8) | bytes[i++];
        decoded.input1_frequency = freq;

        // Tektelic CLOVER TRM Table 4: Frequency (kHz) -> Soil Moisture (% GWC)
        // Normal soil range: 1402 kHz (0% dry) down to 1322 kHz (120% flooded)
        if (freq >= 1402) {
          decoded.soil_moisture_gwc = 0.0;
        } else if (freq >= 1320) {
          var gwc = (1400.0 - freq) * 2.0;
          decoded.soil_moisture_gwc = Math.max(0.0, Math.min(100.0, Number(gwc.toFixed(1))));
        }
      }
      // 0x02 0x02: Input 2 Voltage (2 bytes, unsigned mV) - Soil Temperature Transducer
      else if (channel === 0x02 && type === 0x02) {
        var mv = (bytes[i++] << 8) | bytes[i++];
        decoded.input2_voltage = mv;

        // Tektelic TRM Thermistor formula: Temp(°C) = -32.46 * ln(mV) + 236.36
        if (mv > 0) {
          var soil_t = -32.46 * Math.log(mv) + 236.36;
          decoded.soil_temperature = Number(soil_t.toFixed(1));
        }
      }
      // 0x09 0x65: Illuminance / Light Intensity (2 bytes, unsigned lux)
      else if (channel === 0x09 && type === 0x65) {
        decoded.light_intensity = (bytes[i++] << 8) | bytes[i++];
      }
      // 0x09 0x00: Light Alarm (1 byte)
      else if (channel === 0x09 && type === 0x00) {
        decoded.light_alarm = bytes[i++];
      }
      // 0x0A 0x71: Accelerometer (6 bytes: x, y, z signed 16-bit, 0.001 G)
      else if (channel === 0x0a && type === 0x71) {
        var parseSigned16 = function(b1, b2) {
          var val = (b1 << 8) | b2;
          return val > 32767 ? val - 65536 : val;
        };
        decoded.acceleration_x = Number((parseSigned16(bytes[i++], bytes[i++]) * 0.001).toFixed(3));
        decoded.acceleration_y = Number((parseSigned16(bytes[i++], bytes[i++]) * 0.001).toFixed(3));
        decoded.acceleration_z = Number((parseSigned16(bytes[i++], bytes[i++]) * 0.001).toFixed(3));
      }
      // 0x0A 0x00: Orientation Alarm (1 byte)
      else if (channel === 0x0a && type === 0x00) {
        decoded.orientation_alarm = bytes[i++];
      }
      // 0x0B 0x67: Ambient Temperature (2 bytes, signed 16-bit, 0.1 °C)
      else if (channel === 0x0b && type === 0x67) {
        var raw_t = (bytes[i++] << 8) | bytes[i++];
        if (raw_t > 32767) raw_t -= 65536;
        decoded.ambient_temperature = Number((raw_t * 0.1).toFixed(1));
      }
      // 0x0B 0x68: Relative Humidity (1 byte, 0.5 %)
      else if (channel === 0x0b && type === 0x68) {
        decoded.relative_humidity = Number((bytes[i++] * 0.5).toFixed(1));
      }
      // 0x0C 0x67: MCU Internal Temperature (2 bytes, signed 16-bit, 0.1 °C)
      else if (channel === 0x0c && type === 0x67) {
        var raw_mcu = (bytes[i++] << 8) | bytes[i++];
        if (raw_mcu > 32767) raw_mcu -= 65536;
        decoded.mcu_temperature = Number((raw_mcu * 0.1).toFixed(1));
      } else {
        warnings.push("Skipping unhandled channel 0x" + channel.toString(16) + " type 0x" + type.toString(16));
        break;
      }
    }
  } else {
    warnings.push("Unsupported Port: " + port);
  }

  return {
    data: decoded,
    warnings: warnings,
    errors: errors
  };
}
```

---

## 4. Verification in TTN Console

You can test the decoder right inside the TTN Console before saving:

1. Under the code editor in TTN, locate the **Test** section.
2. Enter:
   * **FPort:** `10`
   * **Payload (hex):** `010405500202038D096528AE0B6700AD0B68AF`
3. Click **Test**.
4. The output should produce:
   ```json
   {
     "ambient_temperature": 17.3,
     "input1_frequency": 1360,
     "input2_voltage": 909,
     "light_intensity": 10414,
     "relative_humidity": 87.5,
     "soil_moisture_gwc": 80,
     "soil_temperature": 15.2
   }
   ```

---

## 5. InfluxDB & Telegraf Ingestion

Because `ucl200.conf` is configured to ingest all keys from `uplink_message.decoded_payload`, Telegraf will write the new fields automatically:

```flux
from(bucket: "UCL200")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "tektelic_soil")
  |> filter(fn: (r) => r["device_id"] == "tektelic-a-2145a0081")
  |> filter(fn: (r) => r["_field"] == "ambient_temperature" or 
                      r["_field"] == "relative_humidity" or 
                      r["_field"] == "soil_temperature" or 
                      r["_field"] == "soil_moisture_gwc" or 
                      r["_field"] == "light_intensity")
  |> aggregateWindow(every: v.windowPeriod, fn: last, createEmpty: false)
  |> yield(name: "last")
```
