---
name: serial-console
description: Test firmware over serial consoles with automated expect scripts
---

# Serial Console Testing Skill

## Purpose & When to Use

Use this skill when you need to:
- Test embedded firmware via serial connection
- Verify boot sequences on embedded devices
- Perform command-response testing on serial-connected hardware
- Test AT command devices (modems, cellular modules, GPS modules)
- Automate repetitive serial console testing tasks

## Phase 1: Device Discovery

First, determine the user's platform and discover available serial ports.

### Platform Detection

Use AskUserQuestion to determine the platform:
- **macOS**: Most common for development
- **Linux**: Common for embedded development and servers
- **Windows**: Requires different tooling

### Listing Available Ports

**macOS:**
```bash
ls /dev/tty.* 2>/dev/null | grep -E "(usb|serial|UART)" || ls /dev/tty.*
```

Common patterns:
- `/dev/tty.usbserial-*` - FTDI and similar USB-serial adapters
- `/dev/tty.usbmodem*` - USB CDC ACM devices
- `/dev/tty.SLAB_USBtoUART*` - Silicon Labs adapters
- `/dev/tty.wchusbserial*` - CH340/CH341 adapters

**Linux:**
```bash
ls /dev/ttyUSB* /dev/ttyACM* /dev/ttyS* 2>/dev/null
```

Common patterns:
- `/dev/ttyUSB0` - USB-serial adapters
- `/dev/ttyACM0` - USB CDC ACM devices (Arduino, etc.)
- `/dev/ttyS0` - Built-in serial ports

**Windows:**
Guide the user to:
1. Open Device Manager → Ports (COM & LPT)
2. Run `mode` command in Command Prompt to list COM ports
3. Use PowerShell: `Get-WMIObject Win32_SerialPort`

Use AskUserQuestion to have the user select the correct port from discovered options.

## Phase 2: Connection Configuration

### Baud Rate Selection

Common baud rates (offer as options):
- **9600** - Legacy devices, slow but reliable
- **19200** - Older equipment
- **38400** - Moderate speed
- **57600** - Common for GPS modules
- **115200** - Most common modern default
- **230400** - High-speed devices
- **460800** - Very high speed
- **921600** - Maximum for most USB-serial adapters

Use AskUserQuestion to select baud rate, defaulting to 115200.

### Serial Settings

Standard configuration (8N1):
- **Data bits**: 8
- **Parity**: None
- **Stop bits**: 1

Flow control options:
- **None** (default) - Most devices
- **Hardware (RTS/CTS)** - High-speed reliable connections
- **Software (XON/XOFF)** - Legacy systems

## Phase 3: Connection Establishment

### Using Screen

**Basic connection:**
```bash
screen /dev/tty.usbserial-XXXX 115200
```

**With logging enabled:**
```bash
screen -L -Logfile serial_log.txt /dev/tty.usbserial-XXXX 115200
```

### Screen Session Management

| Action | Keystroke | Description |
|--------|-----------|-------------|
| Detach | `Ctrl-a d` | Leave session running in background |
| Reattach | `screen -r` | Reconnect to detached session |
| Kill session | `Ctrl-a k` | Terminate the session |
| Toggle logging | `Ctrl-a H` | Start/stop logging to screenlog.0 |
| Scroll mode | `Ctrl-a [` | Enter copy/scrollback mode |
| Exit scroll | `Esc` | Exit scrollback mode |
| List sessions | `screen -ls` | Show all screen sessions |

### Pre-connection Checks

Before connecting, verify:
```bash
# Check if device exists
ls -la /dev/tty.usbserial-XXXX

# Check for existing screen sessions
screen -ls

# Check permissions (Linux)
groups | grep -E "(dialout|uucp)"
```

## Phase 4: Test Scenario Selection

Use AskUserQuestion to determine the testing scenario:

### 1. Boot Sequence Testing
Verify device boots correctly through expected stages:
- Bootloader initialization
- Kernel loading
- System startup
- Login prompt or application start

### 2. Interactive Shell Testing
Test command-line interface:
- Command execution
- Response validation
- Error handling
- State management

### 3. AT Command Testing
Test modems and cellular/GPS modules:
- Basic AT commands (AT, ATI, AT+GMI)
- Signal quality (AT+CSQ)
- Network registration (AT+CREG?)
- Custom AT commands

### 4. Custom Command Sequence
User-defined command/response patterns for specialized hardware.

## Phase 5: Automated Testing with Expect

Generate expect scripts based on the selected scenario.

### Boot Sequence Testing Script

```expect
#!/usr/bin/expect -f
# Boot sequence verification script

set timeout 60
set device [lindex $argv 0]
set baud [lindex $argv 1]

if {$device eq ""} {
    puts "Usage: boot_test.exp <device> <baud>"
    exit 1
}

if {$baud eq ""} {
    set baud 115200
}

log_file -a boot_test.log

spawn -open [open $device r+]
stty $baud raw -echo < $device

puts "\n=== Boot Sequence Test Started ==="
puts "Device: $device"
puts "Baud: $baud"
puts "================================\n"

expect {
    -re "U-Boot|u-boot|UBOOT" {
        puts "\[PASS\] Bootloader detected"
        exp_continue
    }
    -re "Linux version|Starting kernel" {
        puts "\[PASS\] Kernel starting"
        exp_continue
    }
    -re "login:|Login:|#|\\$" {
        puts "\[PASS\] Boot complete - System ready"
        puts "\n=== Boot Sequence Test PASSED ==="
        exit 0
    }
    timeout {
        puts "\[FAIL\] Timeout waiting for boot sequence"
        puts "\n=== Boot Sequence Test FAILED ==="
        exit 1
    }
    eof {
        puts "\[FAIL\] Connection lost"
        exit 1
    }
}
```

### Command-Response Testing Script

```expect
#!/usr/bin/expect -f
# Command-response testing script

set timeout 10
set device [lindex $argv 0]
set baud [lindex $argv 1]

if {$device eq ""} {
    puts "Usage: cmd_test.exp <device> <baud>"
    exit 1
}

if {$baud eq ""} {
    set baud 115200
}

log_file -a cmd_test.log

spawn -open [open $device r+]
stty $baud raw -echo < $device

# Wait for prompt
expect {
    -re "#|\\$|>" {
        puts "\[PASS\] Shell prompt detected"
    }
    timeout {
        puts "\[FAIL\] No shell prompt"
        exit 1
    }
}

# Test command: echo test
send "echo TESTMARKER\r"
expect {
    "TESTMARKER" {
        puts "\[PASS\] Echo command works"
    }
    timeout {
        puts "\[FAIL\] Echo command failed"
        exit 1
    }
}

# Test command: uname
send "uname -a\r"
expect {
    -re "Linux|Darwin|FreeBSD" {
        puts "\[PASS\] System information retrieved"
    }
    timeout {
        puts "\[WARN\] Could not get system info"
    }
}

puts "\n=== Command Test PASSED ==="
exit 0
```

### AT Command Testing Script

```expect
#!/usr/bin/expect -f
# AT command testing script

set timeout 5
set device [lindex $argv 0]
set baud [lindex $argv 1]

if {$device eq ""} {
    puts "Usage: at_test.exp <device> <baud>"
    exit 1
}

if {$baud eq ""} {
    set baud 115200
}

log_file -a at_test.log

spawn -open [open $device r+]
stty $baud raw -echo < $device

puts "\n=== AT Command Test Started ==="

# Basic AT test
send "AT\r"
expect {
    "OK" {
        puts "\[PASS\] AT - Device responding"
    }
    timeout {
        puts "\[FAIL\] AT - No response"
        exit 1
    }
}

# Device identification
send "ATI\r"
expect {
    -re ".*OK" {
        puts "\[PASS\] ATI - Device info retrieved"
    }
    timeout {
        puts "\[WARN\] ATI - No response"
    }
}

# Manufacturer
send "AT+GMI\r"
expect {
    -re ".*OK" {
        puts "\[PASS\] AT+GMI - Manufacturer info"
    }
    timeout {
        puts "\[WARN\] AT+GMI - No response"
    }
}

# Signal quality (for cellular)
send "AT+CSQ\r"
expect {
    -re "\\+CSQ: (\\d+),(\\d+)" {
        set rssi $expect_out(1,string)
        set ber $expect_out(2,string)
        if {$rssi > 0 && $rssi < 32} {
            puts "\[PASS\] AT+CSQ - Signal: $rssi, BER: $ber"
        } else {
            puts "\[WARN\] AT+CSQ - No signal or invalid"
        }
    }
    "ERROR" {
        puts "\[INFO\] AT+CSQ - Not a cellular device"
    }
    timeout {
        puts "\[INFO\] AT+CSQ - Command not supported"
    }
}

# Network registration (for cellular)
send "AT+CREG?\r"
expect {
    -re "\\+CREG: \\d,(\\d)" {
        set status $expect_out(1,string)
        switch $status {
            0 { puts "\[WARN\] AT+CREG - Not registered" }
            1 { puts "\[PASS\] AT+CREG - Registered (home)" }
            2 { puts "\[INFO\] AT+CREG - Searching..." }
            3 { puts "\[FAIL\] AT+CREG - Registration denied" }
            5 { puts "\[PASS\] AT+CREG - Registered (roaming)" }
            default { puts "\[INFO\] AT+CREG - Status: $status" }
        }
    }
    "ERROR" {
        puts "\[INFO\] AT+CREG - Not a cellular device"
    }
    timeout {
        puts "\[INFO\] AT+CREG - Command not supported"
    }
}

puts "\n=== AT Command Test PASSED ==="
exit 0
```

### Running Expect Scripts

```bash
# Make script executable
chmod +x boot_test.exp

# Run the script
./boot_test.exp /dev/tty.usbserial-XXXX 115200

# Or run with expect directly
expect boot_test.exp /dev/tty.usbserial-XXXX 115200
```

## Phase 6: Logging & Capture

### Screen's Built-in Logging

```bash
# Start screen with logging
screen -L -Logfile mylog.txt /dev/tty.usbserial-XXXX 115200

# Toggle logging during session
# Press Ctrl-a H
```

### Script-based Capture

```bash
# Capture with timestamp
script -q serial_session_$(date +%Y%m%d_%H%M%S).log screen /dev/tty.usbserial-XXXX 115200

# Using tee for live output and logging
screen /dev/tty.usbserial-XXXX 115200 | tee serial_output.log
```

### Expect Script Logging

All expect scripts above include `log_file` commands that capture the session to a log file automatically.

## Edge Cases & Troubleshooting

### Device Not Found
```
Error: cannot open /dev/tty.usbserial-XXXX
```
Solutions:
1. Verify the device is connected: `ls /dev/tty.*`
2. Check USB connection
3. Try unplugging and replugging the device
4. Verify drivers are installed (especially on macOS/Windows)

### Permission Denied
```
Error: Permission denied: /dev/ttyUSB0
```
Solutions (Linux):
```bash
# Add user to dialout group
sudo usermod -a -G dialout $USER
# Log out and back in, or:
newgrp dialout

# Or temporarily change permissions
sudo chmod 666 /dev/ttyUSB0
```

### Baud Rate Mismatch (Garbage Output)
Symptoms: Random characters, unprintable symbols
Solutions:
1. Try common baud rates: 9600, 115200
2. Check device documentation
3. Use oscilloscope to measure actual baud rate

### Device Not Responding
Symptoms: No output at all
Solutions:
1. Verify TX/RX connections (may need to swap)
2. Check if device needs DTR/RTS signals
3. Verify device is powered
4. Try hardware flow control

### Connection Drops
Symptoms: Session terminates unexpectedly
Solutions:
1. Check cable quality
2. Verify power supply to device
3. Check for EMI interference
4. Try shorter cable

### Existing Screen Sessions
```bash
# List existing sessions
screen -ls

# Kill zombie sessions
screen -X -S <session_id> quit

# Force kill all sessions
killall screen
```

## Best Practices

1. **Always verify device path** before connecting - paths can change between reboots
2. **Start with standard baud rates** (115200, then 9600) if unknown
3. **Use appropriate timeouts** in expect scripts - too short causes false failures
4. **Clean up screen sessions** after testing to avoid resource leaks
5. **Check for existing sessions** before starting new ones
6. **Document device-specific settings** for your hardware
7. **Use logging** for debugging and record-keeping
8. **Test expect scripts manually first** before automating

## Configuration Options

### Default Values
- **Baud rate**: 115200
- **Data bits**: 8
- **Parity**: None
- **Stop bits**: 1
- **Flow control**: None
- **Expect timeout**: 10 seconds (30-60 for boot tests)

### Common Device Patterns by Platform

**macOS:**
- FTDI: `/dev/tty.usbserial-*`
- CP210x: `/dev/tty.SLAB_USBtoUART*`
- CH340: `/dev/tty.wchusbserial*`
- Arduino: `/dev/tty.usbmodem*`

**Linux:**
- USB-Serial: `/dev/ttyUSB0`
- Arduino/ACM: `/dev/ttyACM0`
- Built-in: `/dev/ttyS0`

## Output Format

### Test Results Summary

```
=== Serial Console Test Results ===
Device: /dev/tty.usbserial-A12345
Baud Rate: 115200
Test Type: Boot Sequence

Results:
  [PASS] Bootloader detected (2.3s)
  [PASS] Kernel starting (5.1s)
  [PASS] System ready (12.4s)

Overall: PASSED
Total Time: 12.4 seconds
Log File: boot_test.log
===================================
```

### Failure Output

```
=== Serial Console Test Results ===
Device: /dev/tty.usbserial-A12345
Baud Rate: 115200
Test Type: Boot Sequence

Results:
  [PASS] Bootloader detected (2.3s)
  [FAIL] Kernel starting - Timeout after 60s

Overall: FAILED
Total Time: 62.3 seconds
Log File: boot_test.log

Recommendations:
  - Check kernel image integrity
  - Verify boot arguments
  - Review log file for errors
===================================
```

## Success Criteria

A successful serial console test session includes:

1. **Connection Established**: Screen session connects without errors
2. **Communication Verified**: Device responds to input
3. **Expected Output Matched**: All expect patterns found
4. **Commands Executed**: All test commands complete
5. **Logs Captured**: Session log saved (if requested)
6. **Clean Exit**: Session terminated properly, no zombie processes
