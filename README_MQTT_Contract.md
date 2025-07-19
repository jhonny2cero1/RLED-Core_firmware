# MQTT Contract for LED Controller Firmware

This document defines the standardized MQTT topics, QoS levels, retain policies, and JSON payload schemas for both the **production** channel (real-time LED control) and the **management** channel (configuration, telemetry, OTA) of the LED controller firmware.

---

## 1. Production Channel (Real-time LED Control)

| Function                                                    | Publish Topic                          | Subscribe Topic (Firmware) | QoS | Retain | Payload Schema |
| ----------------------------------------------------------- | -------------------------------------- | -------------------------- | --- | ------ | -------------- |
| **Segment Command**                                         | `ptl/{controller}/seg`                 | Yes                        | 1   | ❌      | \`\`\`json     |
| {                                                           |                                        |                            |     |        |                |
| "gpio":<int>,       // GPIO number of the strip             |                                        |                            |     |        |                |
| "start":<int>,      // Start index (0-based)                |                                        |                            |     |        |                |
| "end":<int>,        // End index (inclusive)                |                                        |                            |     |        |                |
| "on":<bool>,        // true=turn on segment, false=turn off |                                        |                            |     |        |                |
| "r":<0-255>,        // Red channel intensity                |                                        |                            |     |        |                |
| "g":<0-255>,        // Green channel intensity              |                                        |                            |     |        |                |
| "b":<0-255>         // Blue channel intensity               |                                        |                            |     |        |                |
| }  \`\`\`                                                   |                                        |                            |     |        |                |
| **Command ACK**                                             | `ptl/{controller}/ack`                 | No                         | 1   | ❌      | \`\`\`json     |
| {                                                           |                                        |                            |     |        |                |
| "controller":"<string>",  // Controller name                |                                        |                            |     |        |                |
| "status":"ok"                                               | "error",  // Command processing status |                            |     |        |                |
| "gpio":<int>,  // Echoed GPIO                               |                                        |                            |     |        |                |
| "start":<int>,  // Echoed start index                       |                                        |                            |     |        |                |
| "end":<int>,    // Echoed end index                         |                                        |                            |     |        |                |
| "on":<bool>,    // Echoed on/off command                    |                                        |                            |     |        |                |
| "received\_topic":"<string>"  // Original topic received    |                                        |                            |     |        |                |
| }  \`\`\`                                                   |                                        |                            |     |        |                |

---

## 2. Management Channel (SCADA / Configuration / Telemetry)

### 2.1 Telemetry Topics

All telemetry topics are under the wildcard `controlador/+/status/#`. SCADA systems subscribe to `controlador/+/status/#` to receive:

| Function                              | Topic                                        | QoS | Retain | Payload Schema |
| ------------------------------------- | -------------------------------------------- | --- | ------ | -------------- |
| **Connection**                        | `controlador/{controller}/status/connection` | 1   | ✔️     | \`\`\`json     |
| { "status":"offline" }  // Last Will  |                                              |     |        |                |
| { "status":"online" }   // On connect |                                              |     |        |                |

````|
| **Hardware Info** | `controlador/{controller}/status/info`         | 0   | ✔️     | ```json
{  
  "ip":"<string>",      // Local IP address  
  "mac":"<string>",     // MAC address  
  "firmware":"<string>",// Firmware version  
  "heap":<int>,           // Free heap (bytes)  
  "chip_model":"<string>",  // Chip model  
  "chip_revision":<int>,     // Revision  
  "cores":<int>,           // CPU cores  
  "uptime":<int>           // Seconds since boot  
}  ``` |
| **Reset Reason**  | `controlador/{controller}/status/reset_reason` | 0   | ✔️     | ```json
{  
  "reason":"<string>",      // e.g. "TASK_WDT"  
  "reason_code":<int>  // Reset reason code  
}  ```                                                                     |
| **Heartbeat**     | `controlador/{controller}/status/heartbeat`    | 0   | ❌     | ```json
{  
  "timestamp":<int>,  // UNIX epoch seconds  
  "heap":<int>,       // Free heap  
  "uptime":<int>      // Seconds since boot  
}  ```                                                                     |

### 2.2 Command Topics

Firmware subscribes to `controlador/{controller}/cmd/+` for management commands:

| Command              | Topic                                            | QoS | Retain | Payload Schema                                                                                       |
|----------------------|--------------------------------------------------|-----|--------|------------------------------------------------------------------------------------------------------|
| **Network Config**   | `controlador/{controller}/cmd/config_network`    | 1   | ❌     | ```json
{  
  "use_dhcp":<bool>,  
  "static_ip":"<string>",  
  "subnet":"<string>",  
  "gateway":"<string>",  
  "dns":"<string>"  
}```
|
| **Wi-Fi Config**     | `controlador/{controller}/cmd/config_wifi`       | 1   | ❌     | ```json
{  
  "ssid":"<string>",  
  "password":"<string>"  
}```  |
| **LED Config**       | `controlador/{controller}/cmd/config_leds`       | 1   | ❌     | ```json
{  
  "gpio16":<int>,  
  "gpio4":<int>,   
  "gpio1":<int>,   
  "gpio3":<int>    
}```  |
| **Test Segment**     | `controlador/{controller}/cmd/test_segment`      | 1   | ❌     | ```json
{  
  "gpio":<int>,
  "start":<int>,
  "end":<int>,
  "on":<bool>,
  "r":<0-255>,
  "g":<0-255>,
  "b":<0-255>
}```  |
| **Restart**          | `controlador/{controller}/cmd/restart`           | 1   | ❌     | `{}`  |
| **Firmware Update**  | `controlador/{controller}/cmd/update_firmware`   | 1   | ❌     | `{ "url":"<string>" }`  |

### 2.3 ACK Topics

After processing a management command, firmware publishes ACK on `controlador/{controller}/ack/{command}`:

| ACK for Command      | Topic                                             | QoS | Retain | Payload Schema                                                         |
|----------------------|---------------------------------------------------|-----|--------|------------------------------------------------------------------------|
| **config_network**   | `controlador/{controller}/ack/config_network`     | 1   | ❌     | `{ "status":"ok"|"error", "timestamp":<int>, "error":"<string>" }`  |
| **config_wifi**      | `controlador/{controller}/ack/config_wifi`        | 1   | ❌     | `{ "status":"ok"|"error", "timestamp":<int>, "error":"<string>" }`  |
| **config_leds**      | `controlador/{controller}/ack/config_leds`        | 1   | ❌     | `{ "status":"ok"|"error", "timestamp":<int>, "error":"<string>" }`  |
| **test_segment**     | `controlador/{controller}/ack/test_segment`       | 1   | ❌     | `{ "status":"ok"|"error", "timestamp":<int>, "error":"<string>" }`  |
| **restart**          | `controlador/{controller}/ack/restart`            | 1   | ❌     | `{ "status":"ok"|"error", "timestamp":<int>, "error":"<string>" }`  |
| **update_firmware**  | `controlador/{controller}/ack/update_firmware`    | 1   | ❌     | `{ "status":"ok"|"error", "timestamp":<int>, "error":"<string>" }`  |

---

## 3. Broker ACL Example (Mosquitto)
```conf
# Production client (real-time control)
user prod_client
topic readwrite ptl/+/seg
topic read ptl/+/ack

# SCADA/client de gestión
user scada_client
topic readwrite controlador/+/cmd/#
topic readwrite controlador/+/ack/#
topic read controlador/+/status/#
````

---

## 4. Usage Examples

**Publish segment command (PowerShell)**

```powershell
mosquitto_pub -h 192.168.1.16 -t "ptl/ESP32_PICK_P/seg" -m '{"gpio":16,"start":0,"end":10,"on":true,"r":0,"g":0,"b":255}'
```

**SCADA subscribe to all telemetry**

```bash
mosquitto_sub -h 192.168.1.16 -t "controlador/+/status/#" -v
```

**SCADA send LED config**

```bash
mosquitto_pub -h 192.168.1.16 -t "controlador/ESP32_PICK_P/cmd/config_leds" -m '{"gpio16":120,"gpio4":80,"gpio1":0,"gpio3":0}'
```
