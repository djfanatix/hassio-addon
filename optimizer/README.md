# optimizer (djfanatix fork)

Runs [djfanatix/optimizer](https://github.com/djfanatix/optimizer) (`feat/per-battery-efficiency`) as a Home Assistant add-on, so evcc's optimizer feature can talk to it locally instead of the hosted `https://optimizer.evcc.io`.

Moves `eta_c`/`eta_d` from a single site-wide value to a per-battery field on `BatteryConfig`, so each configured battery's own round-trip efficiency (from its `efficiency` template param in evcc) is honored instead of one shared value for every battery.

## Usage

1. Install and start this add-on. It listens on port `7050`.
2. In the **evcc** add-on, set **Optimizer URI** to `http://<this-host>:7050`.

## Updating to a newer commit

`OPTIMIZER_REF` in `Dockerfile` pins the branch to build. Bump `version` in `config.yaml` on every change so the Home Assistant Supervisor and Docker both rebuild instead of reusing a cached layer - the build itself also self-busts its cache whenever `OPTIMIZER_REF`'s HEAD commit changes.
