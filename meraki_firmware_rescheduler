#!/usr/bin/env python3
"""
Meraki Firmware Upgrade Rescheduler
────────────────────────────────────
A single-file tool for a Meraki admin to reschedule firmware upgrades
across multiple networks with minimal effort.

Flow (all in one guided Quick Run):
  1. Choose filter method  — by network name keyword  OR  by Meraki network tag
  2. Filter & display matching networks
  3. Select which networks to target (individual, range, or all)
  4. Choose product type  (appliance / switch / wireless / etc.)
  5. Enter one upgrade date & time  — applied to all selected networks
  6. Review summary, confirm
  7. Execute rescheduling via Meraki API
  8. Auto-verify results
  9. Optional CSV export


Requirements:
    pip install requests

Setup:
    export MERAKI_API_KEY="<your key>"
    export MERAKI_ORG_ID="<your org id>"
    python meraki_firmware_rescheduler.py
"""

import os
import sys
import time
import csv
from datetime import datetime, timezone
from pathlib import Path

# ══════════════════════════════════════════════════════
#  CONFIG  — edit here OR set as environment variables
# ══════════════════════════════════════════════════════
API_KEY = os.environ.get("MERAKI_API_KEY", "YOUR_API_KEY_HERE")
ORG_ID  = os.environ.get("MERAKI_ORG_ID",  "YOUR_ORG_ID_HERE")

BASE_URL      = "https://api.meraki.com/api/v1"
HEADERS       = {
    "X-Cisco-Meraki-API-Key": API_KEY,
    "Content-Type": "application/json",
    "Accept":       "application/json",
}
PRODUCT_TYPES = ["appliance", "switch", "wireless", "camera", "cellularGateway", "sensor"]


# ══════════════════════════════════════════════════════
#  TERMINAL COLOURS
# ══════════════════════════════════════════════════════
GREEN = "\033[92m"
RED   = "\033[91m"
CYAN  = "\033[96m"
BOLD  = "\033[1m"
DIM   = "\033[2m"
RESET = "\033[0m"
W     = 74



# ══════════════════════════════════════════════════════
#  HTTP HELPERS
# ══════════════════════════════════════════════════════
try:
    import requests
except ImportError:
    print("[ERROR] The 'requests' library is required.  Run: pip install requests")
    sys.exit(1)


def _api_call(method: str, path: str, payload: dict = None):
    url = f"{BASE_URL}{path}"
    for _ in range(4):
        r = (requests.get(url, headers=HEADERS, timeout=30)
             if method == "get"
             else requests.put(url, headers=HEADERS, json=payload, timeout=30))
        if r.status_code == 401:
            raise RuntimeError(
                "API request unauthorized (401) — check that your MERAKI_API_KEY is valid and active."
            )
        if r.status_code == 403:
            raise RuntimeError(
                "API request forbidden (403) — your API key may lack permission for this organization."
            )
        if r.status_code == 429:
            wait = int(r.headers.get("Retry-After", 2))
            print(f"    {DIM}[rate-limited] waiting {wait}s ...{RESET}")
            time.sleep(wait)
            continue
        r.raise_for_status()
        return r.json()
    raise RuntimeError(f"{method.upper()} {path} failed after retries")


def api_get(path: str):
    return _api_call("get", path)


def api_put(path: str, payload: dict) -> dict:
    return _api_call("put", path, payload)


# ══════════════════════════════════════════════════════
#  UI HELPERS
# ══════════════════════════════════════════════════════

def div(ch="─", w=W):
    print(ch * w)


def fmt_dt(iso: str) -> str:
    if not iso:
        return "—"
    try:
        dt = datetime.fromisoformat(iso.replace("Z", "+00:00"))
        return dt.strftime("%Y-%m-%d %H:%M UTC")
    except Exception:
        return iso


def to_iso(date_str: str, time_str: str) -> str:
    dt = datetime.strptime(f"{date_str}T{time_str}:00", "%Y-%m-%dT%H:%M:%S")
    return dt.replace(tzinfo=timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")


def prompt_int(msg: str, lo: int, hi: int) -> int:
    while True:
        v = input(msg).strip()
        if v.isdigit() and lo <= int(v) <= hi:
            return int(v)
        print(f"    {RED}Enter a number between {lo} and {hi}.{RESET}")


def confirm(msg: str) -> bool:
    return input(f"\n  {msg} [y/N]: ").strip().lower() == "y"


def _prompt_date(msg: str):
    """Return 'YYYY-MM-DD' or None if the user presses Enter to cancel."""
    while True:
        raw = input(msg).strip()
        if raw == "":
            return None
        try:
            entered = datetime.strptime(raw, "%Y-%m-%d").replace(tzinfo=timezone.utc)
            now     = datetime.now(timezone.utc)
            # Must be in the future
            if entered.date() < now.date():
                print(f"    {RED}Date is in the past — enter a future date.{RESET}")
                continue
            # Must be within one month
            one_month = now.replace(month=now.month % 12 + 1) if now.month < 12 \
                        else now.replace(year=now.year + 1, month=1)
            if entered > one_month:
                print(f"    {RED}Date exceeds the one-month maximum window "
                      f"(must be on or before {one_month.strftime('%Y-%m-%d')}).{RESET}")
                continue
            return raw
        except ValueError:
            print(f"    {RED}Invalid — use YYYY-MM-DD (e.g. 2025-09-15).{RESET}")


def _prompt_time(msg: str) -> str:
    while True:
        raw = input(msg).strip()
        try:
            datetime.strptime(raw, "%H:%M")
            return raw
        except ValueError:
            print(f"    {RED}Invalid — use HH:MM in 24-hour format (e.g. 02:00).{RESET}")


def _parse_selection(raw: str, max_n: int) -> list:
    """
    Parse '1,3,5-8,10' into a sorted list of 0-based indices.
    Returns [] if nothing valid is found.
    """
    indices = set()
    for part in raw.split(","):
        part = part.strip()
        if "-" in part:
            lo, _, hi = part.partition("-")
            if lo.isdigit() and hi.isdigit():
                indices.update(range(int(lo), int(hi) + 1))
        elif part.isdigit():
            indices.add(int(part))
    return [i - 1 for i in sorted(indices) if 1 <= i <= max_n]


def _print_network_table(networks: list):
    div()
    print(f"  {'#':<5} {'Network Name':<38} {'Tags':<20} TimeZone")
    div()
    for i, n in enumerate(networks, 1):
        tags = ", ".join(n.get("tags", [])) or "—"
        print(f"  {i:<5} {n['name'][:36]:<38} {tags[:18]:<20} {n['timeZone']}")
    div()




# ══════════════════════════════════════════════════════
#  NETWORK FETCHING  — by name keyword OR by tag
# ══════════════════════════════════════════════════════

def _fetch_all_networks() -> list:
    """Return all networks in the org, with tags included."""
    print(f"\n  {DIM}Fetching networks from Meraki ...{RESET}")
    raw = api_get(f"/organizations/{ORG_ID}/networks")
    return [
        {
            "name":     n["name"],
            "id":       n["id"],
            "timeZone": n.get("timeZone", ""),
            "tags":     n.get("tags") or [],   # Meraki returns None or a list
        }
        for n in raw
    ]


def _filter_by_name(networks: list, keyword: str) -> list:
    kw = keyword.lower()
    return [n for n in networks if kw in n["name"].lower()]


def _filter_by_tag(networks: list, tag: str) -> list:
    tag = tag.lower()
    return [n for n in networks if any(tag == t.lower() for t in n["tags"])]


# ══════════════════════════════════════════════════════
#  EXECUTE & VERIFY
# ══════════════════════════════════════════════════════

def _build_and_execute(targets: list, product: str, date_str: str, time_str: str) -> list:
    """
    For every target network:
      1. GET current firmware info
      2. PUT the new scheduled upgrade time
      3. Record Success / Failed
    Returns the completed execution queue.
    """
    iso_time = to_iso(date_str, time_str)
    queue    = []

    print(f"\n  Scheduling {BOLD}[{product}]{RESET} upgrades → {CYAN}{iso_time}{RESET}\n")
    div()
    print(f"  {'Network Name':<38} {'Current → Target':<28} Result")
    div()

    for net in targets:
        name = net["name"]
        nid  = net["id"]

        # Fetch current firmware details
        try:
            data     = api_get(f"/networks/{nid}/firmwareUpgrades")
            products = data.get("products", {})
            if product in products:
                pinfo    = products[product]
                curr_ver = pinfo.get("currentVersion", {}).get("firmware", "N/A")
                to_ver   = pinfo.get("nextUpgrade", {}).get("toVersion", {}).get("firmware", "N/A")
                orig     = pinfo.get("nextUpgrade", {}).get("time", "")
            else:
                curr_ver = to_ver = "N/A"
                orig     = ""
        except Exception as e:
            print(f"  {name[:36]:<38} {'fetch error':<28} {RED}FAILED{RESET}: {e}")
            queue.append({
                "network_name": name, "network_id": nid,
                "original_upgrade": "", "formatted_datetime": iso_time,
                "product_type": product, "current_version": "?",
                "upgrade_to_version": "?", "status": "Failed (fetch)",
            })
            time.sleep(0.12)
            continue

        # Detect already-on-target case before attempting the PUT
        if curr_ver != "N/A" and to_ver != "N/A" and curr_ver == to_ver:
            status  = "Skipped"
            ver_col = f"{curr_ver} == {to_ver}"
            result  = (f"{CYAN}SKIPPED{RESET}  "
                       f"{DIM}(already on target firmware: {curr_ver}){RESET}")
            print(f"  {name[:36]:<38} {ver_col[:26]:<28} {result}")
            queue.append({
                "network_name":       name,
                "network_id":         nid,
                "original_upgrade":   orig,
                "formatted_datetime": iso_time,
                "product_type":       product,
                "current_version":    curr_ver,
                "upgrade_to_version": to_ver,
                "status":             status,
            })
            time.sleep(0.12)
            continue

        # PUT the new schedule
        payload = {"products": {product: {"nextUpgrade": {"time": iso_time}}}}
        try:
            api_put(f"/networks/{nid}/firmwareUpgrades", payload)
            status = "Success"
            result = f"{GREEN}SUCCESS{RESET}"
        except Exception as e:
            status = "Failed"
            result = f"{RED}FAILED{RESET}: {e}"

        ver_col = f"{curr_ver} -> {to_ver}"
        print(f"  {name[:36]:<38} {ver_col[:26]:<28} {result}")

        queue.append({
            "network_name":       name,
            "network_id":         nid,
            "original_upgrade":   orig,
            "formatted_datetime": iso_time,
            "product_type":       product,
            "current_version":    curr_ver,
            "upgrade_to_version": to_ver,
            "status":             status,
        })
        time.sleep(0.12)

    div()
    ok      = sum(1 for r in queue if r["status"] == "Success")
    skipped = sum(1 for r in queue if r["status"] == "Skipped")
    failed  = len(queue) - ok - skipped
    parts   = [f"{GREEN}{ok} succeeded{RESET}"]
    if skipped:
        parts.append(f"{CYAN}{skipped} skipped (already on target){RESET}")
    if failed:
        parts.append(f"{RED}{failed} failed{RESET}")
    print(f"\n  Results: {', '.join(parts)}.")
    return queue


def _verify(queue: list, product: str):
    """Re-read live schedules from Meraki and compare to what was submitted."""
    print(f"\n  {BOLD}Verifying [{product}] schedules against Meraki ...{RESET}\n")
    div()
    print(f"  {'Network Name':<36} {'Live Schedule':<22} {'Submitted':<22} Result")
    div()

    for row in queue:
        submitted = row["formatted_datetime"]
        try:
            data      = api_get(f"/networks/{row['network_id']}/firmwareUpgrades")
            pdata     = data.get("products", {}).get(product, {})
            live_time = pdata.get("nextUpgrade", {}).get("time", "")
            match     = live_time == submitted
            result    = f"{GREEN}Successful{RESET}" if match else f"{RED}Failed{RESET}"
            print(f"  {row['network_name'][:34]:<36} {fmt_dt(live_time):<22} {fmt_dt(submitted):<22} {result}")
        except Exception as e:
            print(f"  {row['network_name'][:34]:<36} {'ERROR':<22} {fmt_dt(submitted):<22} {RED}ERROR: {e}{RESET}")
        time.sleep(0.12)

    div()


def _export_csv(queue: list):
    fname = Path(__file__).parent / f"meraki_reschedule_{datetime.now(timezone.utc).strftime('%Y%m%d_%H%M%S')}.csv"
    with open(fname, "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=queue[0].keys())
        writer.writeheader()
        writer.writerows(queue)
    print(f"\n  {GREEN}-> Saved to {fname}{RESET}")


# ══════════════════════════════════════════════════════
#  QUICK RUN  — the one and only flow
# ══════════════════════════════════════════════════════

def quick_run():
    div("═")
    print(f"  {BOLD}MERAKI FIRMWARE UPGRADE RESCHEDULER{RESET}")
    div("═")
    print(f"  Org ID : {ORG_ID}\n")

    # ── STEP 1: Choose filter method ─────────────────
    print(f"  {BOLD}How would you like to filter networks?{RESET}\n")
    print(f"  [1] By network name  {DIM}(keyword match){RESET}")
    print(f"  [2] By Meraki tag    {DIM}(exact tag match){RESET}")

    raw_method = input(f"\n  Select filter method [1/2]: ").strip()

    if raw_method == "2":
        filter_method = "tag"
    else:
        filter_method = "name"

    # ── STEP 2: Enter the filter value ───────────────
    if filter_method == "name":
        filter_value = input("  Network name keyword (Enter for ALL): ").strip().lower()
    else:
        filter_value = input("  Meraki tag (exact, case-insensitive): ").strip().lower()

    # ── STEP 3: Fetch and filter networks ────────────
    all_networks = _fetch_all_networks()

    if filter_method == "name":
        networks = _filter_by_name(all_networks, filter_value) if filter_value else all_networks
        filter_desc = f"name containing '{filter_value}'" if filter_value else "all networks"
    else:
        if not filter_value:
            print(f"  {RED}A tag value is required for tag filtering.{RESET}")
            return
        networks = _filter_by_tag(all_networks, filter_value)
        filter_desc = f"tag '{filter_value}'"

    if not networks:
        print(f"\n  {RED}No networks matched {filter_desc}.{RESET}")
        print("  Check the filter value and try again.")
        return

    print(f"\n  {GREEN}{len(networks)} network(s){RESET} matched {filter_desc}:\n")
    _print_network_table(networks)

    # ── STEP 4: Select which networks to reschedule ──
    print(f"  Enter numbers or ranges to select specific networks,")
    print(f"  or press {BOLD}Enter{RESET} to select ALL {len(networks)} network(s).\n")

    raw_sel = input("  Selection: ").strip()
    if not raw_sel:
        targets = networks
    else:
        indices = _parse_selection(raw_sel, len(networks))
        targets = [networks[i] for i in indices]

    if not targets:
        print(f"  {RED}No valid networks selected.{RESET}")
        return

    print(f"\n  {BOLD}{len(targets)} network(s) selected:{RESET}")
    for n in targets:
        tags = f"  {DIM}[{', '.join(n['tags'])}]{RESET}" if n.get("tags") else ""
        print(f"    • {n['name']}{tags}")

    # ── STEP 5: Product type ──────────────────────────
    print(f"\n  {BOLD}Select product type to reschedule:{RESET}\n")
    for i, pt in enumerate(PRODUCT_TYPES, 1):
        print(f"    [{i}] {pt}")

    raw_pt = input(f"\n  Select product type (1-{len(PRODUCT_TYPES)}): ").strip()

    if raw_pt.isdigit() and 1 <= int(raw_pt) <= len(PRODUCT_TYPES):
        product = PRODUCT_TYPES[int(raw_pt) - 1]
    else:
        print(f"  {RED}Invalid selection.{RESET}")
        return

    # ── STEP 6: Date & time ───────────────────────────
    print(f"\n  {BOLD}Enter the upgrade date and time (UTC):{RESET}\n")
    date_str = _prompt_date("  Date (YYYY-MM-DD): ")
    if date_str is None:
        print("  Cancelled.")
        return
    time_str = _prompt_time("  Time (HH:MM, 24h): ")

    iso_time = to_iso(date_str, time_str)

    # ── STEP 7: Confirm ───────────────────────────────
    print()
    div("─", 50)
    print(f"  {BOLD}Summary{RESET}")
    div("─", 50)
    print(f"  Filter   : {filter_desc}")
    print(f"  Networks : {len(targets)}")
    print(f"  Product  : {product}")
    print(f"  Schedule : {CYAN}{fmt_dt(iso_time)}{RESET}")
    div("─", 50)

    if not confirm(f"Reschedule {len(targets)} network(s)? This will make changes in Meraki."):
        print("  Cancelled — no changes made.")
        return

    # ── STEP 8: Execute ───────────────────────────────
    queue = _build_and_execute(targets, product, date_str, time_str)

    # ── STEP 9: Auto-verify ───────────────────────────
    _verify(queue, product)

    # ── STEP 10: CSV export ───────────────────────────
    if confirm("Export results to CSV?"):
        _export_csv(queue)

    print(f"\n  {BOLD}Done.{RESET} Run the script again to reschedule another batch.\n")


# ══════════════════════════════════════════════════════
#  ENTRY POINT
# ══════════════════════════════════════════════════════

def validate_config():
    if not API_KEY or "YOUR_API_KEY_HERE" in API_KEY:
        print("[ERROR] MERAKI_API_KEY is not set.")
        print("        Edit the CONFIG block at the top of this script.")
        sys.exit(1)
    if not ORG_ID or "YOUR_ORG_ID_HERE" in ORG_ID:
        print("[ERROR] MERAKI_ORG_ID is not set.")
        print("        Edit the CONFIG block at the top of this script.")
        sys.exit(1)


def main():
    validate_config()
    try:
        quick_run()
    except KeyboardInterrupt:
        print(f"\n\n  {DIM}Interrupted — no changes made.{RESET}\n")
    except Exception as e:
        print(f"\n  {RED}[ERROR]{RESET} {e}\n")
        sys.exit(1)


if __name__ == "__main__":
    main()
