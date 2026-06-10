---
name: customize-dds
description: Customize the DDS (CycloneDDS) middleware configuration for a MoveIt Pro installation — single-computer or dual-computer setups. Use when the user wants MoveIt Pro to talk to ROS 2 on another machine, asks about DDS / CycloneDDS / RMW configuration, sets up a separate driver PC or ML-inference PC, or wants to decide whether MoveIt Pro manages its own DDS config. Follows https://docs.picknik.ai/how_to/computer_configuration/customize_dds_configuration/.
---

# Customize MoveIt Pro DDS Configuration

Configures the ROS 2 DDS middleware for an **installed** MoveIt Pro (the customer
workflow). Walks the user through a **single-computer** (default) or **dual-computer**
setup, decides whether MoveIt Pro manages the DDS config, and writes the right
environment variables.

Reference docs: https://docs.picknik.ai/how_to/computer_configuration/customize_dds_configuration/

## What lives where (read this first)

There are **two** distinct configuration surfaces. Do not conflate them.

1. **The MoveIt Pro config file** — `~/.config/moveit_pro/moveit_pro_config.<MAJOR>.yaml`
   (e.g. `moveit_pro_config.9.yaml` for MoveIt Pro v9.x). A plain YAML file. This is
   where the **`USE_HOST_DDS`** key lives. The `moveit_pro` CLI reads it at launch.

2. **Shell environment variables** — `CYCLONEDDS_NETWORK_INTERFACE`,
   `CYCLONEDDS_PEER_ADDRESSES`, `RMW_IMPLEMENTATION`, `ROS_DOMAIN_ID`. These are read
   from the **environment of the shell that runs `moveit_pro run`** (docker compose
   substitutes `${VAR:-default}` from that environment). They are set by exporting
   them, conventionally in `~/.bashrc`. Shell exports override the CLI defaults.

### `USE_HOST_DDS` semantics — do not invert these

| Value | Meaning |
|-------|---------|
| `'false'` (default) | **MoveIt Pro manages** the DDS config. It generates a CycloneDDS XML inside the container at launch (loopback for single-PC; built from the `CYCLONEDDS_*` env vars for dual-PC). The user does not hand-write any XML. |
| `'true'` | MoveIt Pro **uses the user's own host config** — it copies `~/.ros/cyclonedds.xml` (pointed to by `$CYCLONEDDS_URI`) into the container instead of generating one. The user is fully responsible for that file. |

So *"Should MoveIt Pro manage your DDS configuration? → yes"* means `USE_HOST_DDS: 'false'`.

Both the single- and dual-computer managed workflows below keep `USE_HOST_DDS: 'false'`.

## Step 0 — Locate the config and read current state

```bash
CONFIG_DIR="$HOME/.config/moveit_pro"
# Highest-versioned config wins (e.g. .9.yaml for v9.x); fall back to the unversioned legacy file.
CONFIG_FILE=$(ls -1 "$CONFIG_DIR"/moveit_pro_config.*.yaml 2>/dev/null | sort -t. -k2 -n | tail -1)
[ -z "$CONFIG_FILE" ] && CONFIG_FILE="$CONFIG_DIR/moveit_pro_config.yaml"
echo "Using config: $CONFIG_FILE"

# Current DDS-relevant state.
grep -E '^USE_HOST_DDS:' "$CONFIG_FILE" || echo "USE_HOST_DDS not set (defaults to false)"
echo "RMW_IMPLEMENTATION=${RMW_IMPLEMENTATION:-<unset>}"
echo "ROS_DOMAIN_ID=${ROS_DOMAIN_ID:-<unset>}"
echo "CYCLONEDDS_NETWORK_INTERFACE=${CYCLONEDDS_NETWORK_INTERFACE:-<unset>}"
echo "CYCLONEDDS_PEER_ADDRESSES=${CYCLONEDDS_PEER_ADDRESSES:-<unset>}"
grep -nE 'CYCLONEDDS_(NETWORK_INTERFACE|PEER_ADDRESSES)|RMW_IMPLEMENTATION|ROS_DOMAIN_ID' "$HOME/.bashrc" 2>/dev/null
```

If the config file does not exist, the user has not run `moveit_pro configure` yet — tell
them to install/configure MoveIt Pro first, then return.

## Step 1 — Ask: single or dual computer?

- **Single computer** — MoveIt Pro and all ROS 2 processes run on one machine (the
  default). Talks to the host and the container, not across the network.
- **Dual computer** — drivers / real-time control on one PC, and a second PC for ML
  hardware acceleration or PLC data exchange. The two must discover each other over DDS.

## Step 2 — Decide who manages the DDS config (`USE_HOST_DDS`)

Read the current value from Step 0, then:

- **If `USE_HOST_DDS` is currently `'true'`** (user supplies their own host config): ask
  whether they want to **keep it `'true'`** (they continue managing `~/.ros/cyclonedds.xml`
  themselves — no further action from this skill) or **switch to MoveIt-Pro-managed**
  (`'false'`). Do **not** diff or rewrite their XML; just honor their choice.
- **If `USE_HOST_DDS` is `'false'` or unset**: confirm they want MoveIt Pro to manage it
  (the normal case). Keep `'false'`.

For both the single- and dual-computer managed flows the target value is `'false'`.

Write the chosen value into the YAML config (preserve the single-quoted string style the
file already uses):

```bash
if grep -qE '^USE_HOST_DDS:' "$CONFIG_FILE"; then
  sed -i "s/^USE_HOST_DDS:.*/USE_HOST_DDS: 'false'/" "$CONFIG_FILE"   # or 'true' if the user chose to keep their own
else
  printf "USE_HOST_DDS: 'false'\n" >> "$CONFIG_FILE"
fi
grep -E '^USE_HOST_DDS:' "$CONFIG_FILE"
```

`moveit_pro configure` (interactive) is the official alternative for this key, but it
prompts; editing the YAML directly is reliable and non-interactive.

## Step 3a — Single computer (managed)

Nothing else is required: with `USE_HOST_DDS: 'false'`, MoveIt Pro generates a loopback
CycloneDDS config inside the container at launch. Just verify the prerequisites:

- **`ROS_DOMAIN_ID` should be unset.** If it is set, ROS 2 will not discover the
  container's processes. Offer to remove any `export ROS_DOMAIN_ID=...` from `~/.bashrc`.
- **`RMW_IMPLEMENTATION` should be unset or `rmw_cyclonedds_cpp`.**
- **Stale dual-PC exports break single-PC.** If `CYCLONEDDS_NETWORK_INTERFACE` or
  `CYCLONEDDS_PEER_ADDRESSES` are exported (from a prior dual setup), they force
  non-loopback discovery. Offer to remove those lines from `~/.bashrc` so the defaults
  (`lo` / `127.0.0.1`) apply.

Then tell the user to launch normally: `moveit_pro run`.

## Step 3b — Dual computer (managed via env vars)

Keep `USE_HOST_DDS: 'false'` on **both** machines. MoveIt Pro builds the CycloneDDS XML
from two environment variables:

| Variable | Value | Same on both PCs? |
|----------|-------|-------------------|
| `CYCLONEDDS_NETWORK_INTERFACE` | the **local** machine's NIC name on the shared subnet (e.g. `enp0s31f6`) | No — each PC its own |
| `CYCLONEDDS_PEER_ADDRESSES` | comma-separated IPs of **all** participating PCs, **including this machine's own IP** | Yes — identical list |

### 1. Ask about SSH

Ask: *"Can you SSH into the PC where the drivers will be launched?"* Get the SSH target
(`user@host`) if yes. SSH lets the skill read `ip a` and write exports on the remote PC
directly. If no, the skill produces the exact commands for the user to run there manually.

Decide roles with the user: typically the PC running this skill is the **main** PC
(UI / ML) and the SSH target is the **driver** PC. Confirm which is which.

### 2. Discover interfaces and IPs (needs user input)

The correct interface/IP is the one on the **subnet that physically links the two PCs** —
the skill cannot infer this, so show candidates and ask.

```bash
# This (local) machine — compact list, drop loopback and down interfaces:
ip -brief address show | awk '$1 != "lo" && $2 == "UP" && $3 != "" {print $1, $3}'

# The driver PC (if SSH is available):
ssh user@driver-host "ip -brief address show | awk '\$1 != \"lo\" && \$2 == \"UP\" && \$3 != \"\" {print \$1, \$3}'"
```

If SSH is unavailable, ask the user to run the `ip -br a` command on the driver PC and
paste the output.

Present the candidates and ask the user to confirm, for **each** PC:
- the **interface name** on the shared subnet → that PC's `CYCLONEDDS_NETWORK_INTERFACE`,
- the **IPv4 address** (without the `/NN` prefix) on that subnet.

Then build the shared peer list: `CYCLONEDDS_PEER_ADDRESSES` = both IPs, comma-separated,
identical on both machines. Example with main `192.168.10.12` (iface `enp112s0`) and
driver `192.168.10.10` (iface `enp0s31f6`):

- Main PC: `CYCLONEDDS_NETWORK_INTERFACE=enp112s0`, `CYCLONEDDS_PEER_ADDRESSES=192.168.10.10,192.168.10.12`
- Driver PC: `CYCLONEDDS_NETWORK_INTERFACE=enp0s31f6`, `CYCLONEDDS_PEER_ADDRESSES=192.168.10.10,192.168.10.12`

### 3. Ask where to write the exports (default `~/.bashrc`)

Ask which file the exports should be appended to. **Default to `~/.bashrc`.** (Other
reasonable choices: `~/.bash_profile`, `~/.profile`, or a sourced `~/.moveit_pro_dds.env`.)

Write on the **main** PC (substitute the confirmed values):

```bash
RC_FILE="$HOME/.bashrc"   # or the file the user chose
cat >> "$RC_FILE" <<'EOF'

# MoveIt Pro dual-computer DDS configuration (CycloneDDS).
export CYCLONEDDS_NETWORK_INTERFACE=enp112s0
export CYCLONEDDS_PEER_ADDRESSES=192.168.10.10,192.168.10.12
EOF
```

Write on the **driver** PC over SSH (note the *different* interface, *same* peer list):

```bash
ssh user@driver-host 'cat >> ~/.bashrc <<EOF

# MoveIt Pro dual-computer DDS configuration (CycloneDDS).
export CYCLONEDDS_NETWORK_INTERFACE=enp0s31f6
export CYCLONEDDS_PEER_ADDRESSES=192.168.10.10,192.168.10.12
EOF'
```

Before appending, `grep` the target file for existing `CYCLONEDDS_NETWORK_INTERFACE` /
`CYCLONEDDS_PEER_ADDRESSES` lines and update those in place instead of adding duplicates.

### 4. Verify prerequisites on both PCs

Same as single-computer: `USE_HOST_DDS: 'false'` (set the YAML on the driver PC too —
over SSH, using the Step 2 snippet against its own `~/.config/moveit_pro/...`),
`ROS_DOMAIN_ID` unset, `RMW_IMPLEMENTATION` unset or `rmw_cyclonedds_cpp`.

### 5. Launch

Exports only take effect in a **new** shell (or after `source ~/.bashrc`). Then split the
stack across the two PCs:

```bash
# On the driver PC:
moveit_pro run --only-drivers
# On the main PC:
moveit_pro run --no-drivers
```

Confirm cross-PC discovery, e.g. `ros2 topic list` on one PC should show topics published
by the other.

## Appendix — reference

### Key environment variables

| Variable | Purpose | Default |
|----------|---------|---------|
| `USE_HOST_DDS` (YAML config) | `false` = MoveIt Pro generates the config; `true` = use the user's `~/.ros/cyclonedds.xml` | `false` |
| `RMW_IMPLEMENTATION` | DDS vendor; unset or `rmw_cyclonedds_cpp` for the managed flows | `rmw_cyclonedds_cpp` |
| `ROS_DOMAIN_ID` | Must be **unset** for the managed flows | unset |
| `CYCLONEDDS_NETWORK_INTERFACE` | This PC's NIC on the shared subnet (dual-PC) | `lo` |
| `CYCLONEDDS_PEER_ADDRESSES` | Comma-separated IPs of all PCs incl. self (dual-PC) | `127.0.0.1` |
| `CYCLONEDDS_MAX_AUTO_PARTICIPANT_INDEX` | Highest participant index probed; raise if discovery silently fails with many processes | `120` |

### Canonical single-PC CycloneDDS XML (for reference only)

MoveIt Pro generates this automatically when `USE_HOST_DDS: 'false'`. Only relevant if a
user chooses `USE_HOST_DDS: 'true'` and wants to author their own `~/.ros/cyclonedds.xml`:

```xml
<?xml version='1.0' encoding='us-ascii'?>
<CycloneDDS xmlns="https://cdds.io/config" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="https://cdds.io/config https://raw.githubusercontent.com/eclipse-cyclonedds/cyclonedds/master/etc/cyclonedds.xsd">
    <Domain id="any">
        <General>
            <Interfaces>
                <NetworkInterface autodetermine="false" name="lo" />
            </Interfaces>
            <AllowMulticast>false</AllowMulticast>
        </General>
        <Discovery>
            <ParticipantIndex>auto</ParticipantIndex>
            <Peers>
                <Peer Address="127.0.0.1" />
            </Peers>
            <MaxAutoParticipantIndex>120</MaxAutoParticipantIndex>
        </Discovery>
        <Internal>
            <SocketReceiveBufferSize min="10MB" />
        </Internal>
    </Domain>
</CycloneDDS>
```
