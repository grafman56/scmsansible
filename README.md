# CMTS Modem Count Poller

Reads an external CSV device list, runs `show cable modem summary total` against
every CMTS in parallel, and writes a single timestamped CSV report.

## Layout

    ansible.cfg
    cmts_modem_count.yml          main playbook (3 plays)
    inventory/devices.csv         external device list (edit this)
    group_vars/cmts.yml           connection + credential vars
    templates/report.csv.j2       report format
    reports/                      output lands here (auto-created)

## Requirements

    ansible-galaxy collection install cisco.ios ansible.netcommon community.general

## Run

    export CMTS_USER=youruser
    export CMTS_PASS=yourpass
    ansible-playbook cmts_modem_count.yml

Point at a different list without touching the playbook:

    ansible-playbook cmts_modem_count.yml -e device_list=inventory/lab.csv

## The device list

`inventory/devices.csv` is the only file your NOC needs to edit:

    hostname,address,platform,site
    cmts-rva-01,10.0.0.1,cisco.ios,Market

The `platform` column is per-device so you can branch to another OS later
without restructuring anything.

## One thing to verify: column order

The poll parses the single `Total` row that the `total` keyword produces, then
maps its integers onto `cmts_summary_columns` (default
`[total, reg, unreg, offline, wideband]`). Column order varies across IOS-XE
trains, so confirm it once against real output:

    cmts-rva-01# show cable modem summary total

If your build orders columns differently, edit `cmts_summary_columns` in
`cmts_modem_count.yml`. Nothing else changes. Unmatched columns are dropped
safely, so a shorter row will not error: it just fills the names it has.

## Why it is efficient

- The `total` keyword aggregates on the CMTS, so each device returns one line,
  not a full per-interface dump.
- One command, one task per device: no loops, no extra round trips.
- `strategy: free` plus `forks = 50` means all CMTS poll concurrently and fast
  boxes never block on slow ones.
- `network_cli` holds one persistent SSH session per device for the whole run.
- Unreachable or unparseable devices are flagged (`reachable=false`) instead of
  failing the run, so the report is always complete.

## Credentials

`group_vars/cmts.yml` pulls creds from env vars by default. For production,
swap those lookups for Ansible Vault:

    ansible-vault create group_vars/cmts/vault.yml
    ansible-playbook cmts_modem_count.yml --ask-vault-pass
