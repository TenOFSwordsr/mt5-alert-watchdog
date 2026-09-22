# MT5 Alert Watchdog

A small Python keep-alive and backup-detector that sits beside the EMA alert EA. When the
chart EA goes silent (crashes, detaches, or the MT5 terminal dies), this watches the market
directly through the MetaTrader5 Python API and posts the same cross alerts the EA would have.
It never competes with the EA: primary detection always stays with the chart EA, and the
watchdog only covers the gap while its ticker file is stale.

**Suggested repo name:** `mt5-alert-watchdog`
**Stack:** Python 3, `MetaTrader5` package, `requests`
**Status:** finished (single-file utility)
**Last modified:** 2026-08-24

## What it does

- Reads the EA's own ticker files (`ema_ticker_<SYMBOL>_<TF>.json`) from the terminal common-files
  directory. Each file doubles as the EA's last-known config and a heartbeat.
- Treats a stream as EA-owned while its file is fresh (`EA_SILENCE` = 180s). Only when a stream is
  silent does the watchdog attach to the terminal, pull bars via `mt5.copy_rates_from_pos`, and
  recompute the fast/slow MA cross itself - replicating RMA/EMA/SMA/LWMA so its signal matches what
  the EA would have produced.
- Fires a `POST /alert` to the local alert server (`http://127.0.0.1:5000`) with the identical JSON
  payload, tagged `strategy_name: "AutoWatchdog"` and `tag: "watchdog"` so backup alerts are
  distinguishable from EA alerts downstream.
- Launches `terminal64.exe` and retries `mt5.initialize()` when the terminal is not attached.
- `mt5-keepalive.cmd` is a coarser supervisor meant for a Windows scheduled task: it starts
  `terminal64.exe` if it is not running, and when the EA's ticker has been stale for over ten
  minutes it stops the terminal, restores the previously backed-up chart profiles with `robocopy`,
  and relaunches it - recovering the trader's saved layouts after a hang.

## Layout

```
watchdog.py            detector loop: read tickers, back up for silent EA streams, POST /alert
mt5-keepalive.cmd      scheduled-task supervisor: (re)start terminal64, back up/restore chart profiles
test-outbox.json       sample XAUUSD M5 alert payload for manual testing
assets/               Terminal.ico / ema-monitor.ico tray icons
```

## Running it

```bash
python watchdog.py            # continuous loop, polls every POLL (30s)
python watchdog.py --test     # one cycle that logs each stream's state without posting alerts
```

Requires the `MetaTrader5` Python package and the MT5 terminal installed on Windows; the Bearer key
is read from the alert server's `.env`. Set up `mt5-keepalive.cmd` as a recurring scheduled task.

## Notes

- Hard-coded to this machine's paths: `TICKER_DIR`, `TERMINAL`, `ENV` (the alert server's `.env`),
  `LOG` (`C:\NotifierWatchdog\watchdog.log`) and `BASE` (`127.0.0.1:5000`). Edit them to relocate.
- `api_key()` reads the Bearer token from `.env` at runtime - the token itself is not committed,
  but the `.env` it points at is secret and stays outside this folder.
- `ensure_terminal()` calls `mt5.initialize()` on the main thread; it expects a logged-in terminal
  profile and will spawn one if absent, which pops a GUI window.
- `mt5-keepalive.cmd` uses `Stop-Process -Force` on `terminal64` and overwrites the live
  `MQL5/Profiles/Charts` directory from its backup; a running EA loses its unsaved chart state.
- The terminal GUID `D0E8209F...` is baked into the `.cmd`; that is this install's data folder.
- Companion to `ema-alert-server` (receives the alerts) and `ema-alert-ea` (whose heartbeat it
  monitors); it is the "MT5 keep-alive" piece, not the EA or the server.
