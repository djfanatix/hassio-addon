# optimizer (djfanatix fork)

Runs [djfanatix/optimizer](https://github.com/djfanatix/optimizer) (`feat/self-consumption-goal`) as a Home Assistant add-on, so evcc's optimizer feature can talk to it locally instead of the hosted `https://optimizer.evcc.io`.

Adds a `primary_goal` option to the optimizer request on top of upstream `evcc-io/optimizer`: `minimize_cost` (default, unchanged) or `maximize_self_consumption`, which weighs grid export as a cost like import instead of revenue, so a battery with headroom is preferred over selling even when selling would earn more.

## Usage

1. Install and start this add-on. It listens on port `7050`.
2. In the **evcc** add-on, set **Optimizer URI** to `http://<this-host>:7050`.
3. In evcc's Optimize page, "Primary goal" becomes a live choice instead of locked to "Lowest cost".

## Updating to a newer commit

`OPTIMIZER_REF` in `Dockerfile` pins the branch to build. Bump `version` in `config.yaml` on every change so the Home Assistant Supervisor and Docker both rebuild instead of reusing a cached layer - the build itself also self-busts its cache whenever `OPTIMIZER_REF`'s HEAD commit changes.
