# Meraki Firmware Upgrade Rescheduler

A single-file command-line tool for Cisco Meraki admins to reschedule firmware upgrades across multiple networks in one guided run.

## Features

- Filter networks by **name keyword** or **Meraki tag**
- Select individual networks, ranges, or all at once
- Enforces a **one-month scheduling window** to prevent far-future mistakes
- Skips networks already running the target firmware
- Auto-verifies schedules after applying them
- Optional CSV export saved alongside the script

## Requirements

- Python 3.8+
- `requests` library

```bash
pip install requests
```

## Setup

Set your Meraki API key and Org ID under the `CONFIG` block at the top of the script directly.

```bash
MERAKI_API_KEY="your_api_key_here"
MERAKI_ORG_ID="your_org_id_here"
```

## Usage

```bash
python3 meraki_firmware_rescheduler.py
```

The tool will guide you through the following steps:

1. Choose a filter method — by network name keyword or Meraki tag
2. View matching networks
3. Select which networks to target
4. Choose the product type (appliance, switch, wireless, etc.)
5. Enter an upgrade date and time (UTC) — must be within the next 30 days
6. Review and confirm
7. Execute rescheduling via the Meraki API
8. Auto-verify results
9. Optionally export a CSV report

## CSV Export

If you choose to export, the CSV is saved in the same directory as the script with a timestamped filename:

```
meraki_reschedule_20260402_143000.csv
```

## Supported Product Types

- `appliance`
- `switch`
- `wireless`
- `camera`
- `cellularGateway`
- `sensor`

## Notes

- All scheduled times are in **UTC**
- Networks already on the target firmware version are automatically skipped
- The script will not make any changes until you explicitly confirm
