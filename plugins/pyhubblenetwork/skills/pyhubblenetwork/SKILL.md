---
name: pyhubblenetwork
description: Test and provision Hubble Network devices using the pyhubblenetwork SDK
---

# Hubble Network Device Testing Skill

## Purpose & When to Use

Use this skill when you need to:
- Test BLE packet transmission from Hubble devices
- Provision Hubble Ready devices via BLE
- Verify device time synchronization
- Check device connectivity and signal strength
- Validate Hubble API credentials

## Phase 1: Environment Setup

Before running any tests, verify the environment is properly configured.

### Check pyhubblenetwork Installation

```bash
python -c "import hubblenetwork; print(f'pyhubblenetwork installed')" 2>/dev/null || echo "NOT INSTALLED"
```

If not installed, inform the user:
```
pyhubblenetwork is not installed. Install it with:
  pip install pyhubblenetwork
```

### Check Environment Variables

```bash
echo "HUBBLE_ORG_ID: ${HUBBLE_ORG_ID:-(not set)}"
echo "HUBBLE_API_TOKEN: ${HUBBLE_API_TOKEN:+(set)}"
```

If variables are missing, inform the user:
```
Required environment variables:
  export HUBBLE_ORG_ID="your-org-id"
  export HUBBLE_API_TOKEN="your-api-token"

Get these from your Hubble Network dashboard.
```

### Validate Credentials

```bash
hubblenetwork validate-credentials
```

Expected output: `Valid credentials (env="production")` or similar.

If invalid, stop and inform the user to check their credentials.

## Phase 2: Test Mode Selection

Use AskUserQuestion to determine what the user wants to do:

| Mode | Description | Requirements |
|------|-------------|--------------|
| BLE Packet Detection | Detect and decrypt a single BLE packet | Device key (base64) |
| BLE Packet Scanning | Continuously scan for BLE packets | Optional: Device key |
| Device Time Check | Verify device UTC time is within spec | Device key (base64) |
| Ready Device Scan | Find devices ready for provisioning | None |
| Ready Device Info | Read characteristics from a Ready device | None |
| Device Provisioning | Provision a Ready device via BLE | Valid API credentials |
| End-to-End Test | Provision then verify BLE transmission | Valid API credentials |

## Phase 3: Platform Discovery

### Detect Platform and BLE Adapter

**macOS:**
```bash
system_profiler SPBluetoothDataType 2>/dev/null | head -20
```

**Linux:**
```bash
hciconfig 2>/dev/null || echo "hciconfig not available"
bluetoothctl show 2>/dev/null | head -10
```

### Common Issues

| Issue | Solution |
|-------|----------|
| No BLE adapter | Ensure Bluetooth is enabled in system settings |
| Permission denied | On Linux: add user to `bluetooth` group |
| Adapter busy | Close other BLE applications |

## Phase 4: Test Execution

### BLE Packet Detection

Detects a single BLE packet and attempts decryption with the provided key.

**Gather device key:**
Use AskUserQuestion to get the device key (base64 encoded).

**Run detection:**
```bash
hubblenetwork ble detect --key "${DEVICE_KEY}" --timeout 30 --format json
```

**JSON Output (success):**
```json
{
  "success": true,
  "packet": {
    "datetime": "Sun Feb  1 12:34:56 2026",
    "rssi": -65,
    "payload_bytes": 8
  }
}
```

**JSON Output (failure):**
```json
{
  "success": false,
  "error": "No valid packets found within timeout period"
}
```

### BLE Packet Scanning

Continuously scans for BLE packets. Can decrypt if key is provided.

**Without key (raw packets):**
```bash
hubblenetwork ble scan --timeout 60 --format json
```

**With key (decrypted packets):**
```bash
hubblenetwork ble scan --key "${DEVICE_KEY}" --timeout 60 --count 5 --format json
```

**Options:**
- `--timeout`: Scan duration in seconds
- `--count`: Stop after N packets (optional)
- `--key`: Device key for decryption (optional)
- `--days`: Days to check back for time counter (default: 2)
- `--format`: Output format (tabular or json)

### Device Time Check

Verifies the device's UTC time counter is within specification (+/- 2 days).

```bash
hubblenetwork ble check-time --key "${DEVICE_KEY}" --timeout 30 --json-output
```

**JSON Output:**
```json
{
  "resolved": true,
  "delta_days": 0,
  "in_spec": true,
  "rssi": -62,
  "timestamp": "Sun Feb  1 12:34:56 2026"
}
```

**Interpretation:**
- `delta_days`: 0 = correct, negative = behind, positive = ahead
- `in_spec`: true if within +/- 2 days

### Ready Device Scan

Scans for devices advertising the Hubble Provisioning Service (0xFCA7).

```bash
hubblenetwork ready scan --timeout 15 --format json
```

**Output:**
```json
[
  {"name": "HubbleReady-A1B2", "address": "AA:BB:CC:DD:EE:FF", "rssi": -55},
  {"name": "HubbleReady-C3D4", "address": "11:22:33:44:55:66", "rssi": -72}
]
```

### Ready Device Info

Connects to a Ready device and reads its provisioning characteristics.

```bash
hubblenetwork ready info --timeout 15 --format json
```

This command is interactive - it will prompt for device selection.

**Output includes:**
- Status: Version, mode (Open/Locked), flags (Key/Config/Time written)
- Device Key: Encryption mode (AES-256-CTR or AES-128-CTR)
- Device Configuration: EID type, rotation period
- Epoch Time: Current device time

### Device Provisioning

Provisions a Ready device by registering with the backend and writing configuration.

```bash
hubblenetwork ready provision --verbose
```

**Provisioning Steps:**
1. Scan for Ready devices
2. Select device interactively
3. Enter device name
4. Connect and verify Open Mode
5. Register device with Hubble backend
6. Write encryption key
7. Write device configuration
8. Write epoch time
9. Verify provisioning success

**Success Output:**
```
[SUCCESS] Device provisioned!
  Device ID: abc123-def456-...
  Name: MyDevice
  Encryption: AES-256-CTR
  Key: base64encodedkey==
```

### End-to-End Test

For complete validation, perform provisioning followed by BLE verification:

1. Run `hubblenetwork ready provision --verbose`
2. Save the returned device key
3. Wait 15 seconds for device to start transmitting
4. Run `hubblenetwork ble detect --key "${KEY}" --timeout 30 --format json`
5. Verify success

## Phase 5: Verification Criteria

### BLE Packet Detection
| Check | Pass Criteria |
|-------|---------------|
| Packet received | `success: true` in JSON output |
| Decryption works | Packet payload decrypted successfully |
| Signal strength | RSSI > -90 dBm (reasonable range) |

### Device Time Check
| Check | Pass Criteria |
|-------|---------------|
| Time resolved | `resolved: true` |
| Within spec | `in_spec: true` (delta within +/- 2 days) |

### Device Provisioning
| Check | Pass Criteria |
|-------|---------------|
| Device found | At least one Ready device discovered |
| Mode is Open | Device not in Locked mode |
| Registration | Device ID returned from backend |
| Key written | Status shows `Key=Yes` |
| Config written | Status shows `Config=Yes` |
| Time written | Status shows `Time=Yes` |

## Phase 6: Reporting

Generate a test report summarizing results.

### Report Template

```markdown
# Hubble Device Test Report

**Date:** YYYY-MM-DD HH:MM:SS
**Test Mode:** [Selected Mode]

## Environment
- Platform: macOS/Linux
- pyhubblenetwork: Installed
- Credentials: Valid

## Device Under Test
- Device ID: [if available]
- Name: [if available]
- BLE Address: [if available]

## Test Results

### [Test Name]
- **Status:** PASS/FAIL
- **Details:** [specifics]

| Metric | Value |
|--------|-------|
| RSSI | -XX dBm |
| Packets | N |
| Time Delta | X days |

## Overall Result: PASS/FAIL
```

## Configuration Options

| Option | Default | Description |
|--------|---------|-------------|
| `scan_timeout` | 30 | BLE scan timeout in seconds |
| `packet_count` | 1 | Packets to collect (scanning mode) |
| `eid_type` | "utc" | EID type for provisioning |
| `output_format` | "json" | Output format: json or tabular |
| `verification_delay` | 15 | Seconds to wait before verification after provisioning |

## Error Handling

| Error | Cause | Recovery |
|-------|-------|----------|
| `pyhubblenetwork not installed` | SDK not in Python path | `pip install pyhubblenetwork` |
| `HUBBLE_ORG_ID not set` | Missing env var | Set environment variable |
| `HUBBLE_API_TOKEN not set` | Missing env var | Set environment variable |
| `Invalid credentials` | Wrong org ID or token | Verify credentials in dashboard |
| `No BLE packets found` | Device not transmitting or out of range | Check device power, move closer |
| `Base64 decoding failed` | Invalid key format | Verify key is valid base64 |
| `No valid packets found` | Key doesn't match device | Verify correct key for device |
| `No Hubble Ready devices found` | No devices in provisioning mode | Check device is advertising 0xFCA7 |
| `Device is in Locked Mode` | Already provisioned | Cannot re-provision locked devices |
| `Connection failed` | BLE connection issue | Retry, check distance, restart Bluetooth |
| `Device time out of spec` | Time counter drifted | Re-provision device to reset time |

## CLI Reference

```
hubblenetwork --help
hubblenetwork ble --help
hubblenetwork ble detect --help
hubblenetwork ble scan --help
hubblenetwork ble check-time --help
hubblenetwork ready --help
hubblenetwork ready scan --help
hubblenetwork ready info --help
hubblenetwork ready provision --help
hubblenetwork validate-credentials --help
hubblenetwork org --help
hubblenetwork org info
hubblenetwork org list-devices
hubblenetwork org get-packets DEVICE_ID
```

## Success Criteria

A successful test session includes:

1. **Environment Ready**: pyhubblenetwork installed, credentials valid
2. **BLE Functional**: Can detect/scan BLE packets
3. **Decryption Works**: Packets decrypt with correct key
4. **Time In Spec**: Device time within +/- 2 days (if checked)
5. **Provisioning Complete**: All characteristics written (if provisioning)
6. **Report Generated**: Summary of all test results
