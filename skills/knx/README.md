# 🏠 KNX — Home automation over KNXnet/IP

Connects Arkimede to a KNX installation through a KNX/IP gateway (router or
interface). The agent can switch lights, drive dimmers, shutters, scenes and
thermostat setpoints, read sensor/actuator state, monitor live bus traffic and
discover gateways on the LAN.

Built on [xknx](https://github.com/XKNX/xknx), the KNX library used by Home Assistant.

---

## ⚙️ Configuration

### 1. Find your gateway

Ask the agent to run `knx_discover`, or check your router/ETS project. You need
the IP address of the KNX/IP gateway (e.g. `192.168.1.160`).

### 2. Configure the skill in Arkimede

Settings → Skills → knx → Configure:

| Variable | Required | Description |
|---|:---:|---|
| `KNX_GATEWAY_HOST` | ✅ | IP of the KNX/IP gateway |
| `KNX_GATEWAY_PORT` | — | KNXnet/IP port (default `3671`) |
| `KNX_CONNECTION` | — | `tunneling` (UDP, default), `tunneling_tcp`, `routing` |
| `KNX_GA_MAP` | — | JSON map of friendly names → group addresses (see below) |

### 3. Grant LAN network access (admin)

KNXnet/IP tunneling is **UDP on the local network**: it cannot go through the
HTTP egress proxy (`internet` tier). The skill needs a granted network that can
reach the LAN:

1. Make sure a LAN-capable Docker network is listed in `SKILL_NETWORK_CATALOG`
   (the guided `install.sh` can set this up; the default `bridge` network
   reaches the LAN via NAT).
2. Settings → Skills → knx → **Networks** (admin only) → grant that network.

Without the grant, jobs run on the internal-only baseline network and the
gateway is unreachable.

---

## 🗺️ Group address map (`KNX_GA_MAP`)

Optional but recommended: it lets the agent control devices by name
("turn on the kitchen light") instead of raw group addresses.

```json
{
  "luce cucina":  { "ga": "1/1/5", "dpt": "switch",      "status_ga": "1/4/5", "description": "Ceiling light" },
  "dimmer sala":  { "ga": "1/1/8", "dpt": "percent",     "status_ga": "1/4/8" },
  "tapparella camera": { "ga": "2/1/3", "dpt": "up_down" },
  "temperatura soggiorno": { "ga": "1/5/0", "dpt": "temperature" }
}
```

- `ga` — command group address (used by writes)
- `status_ga` — status group address (used by reads, optional)
- `dpt` — datapoint type: number (`"1.001"`, `"5.001"`, `"9.001"`) or alias
  (`switch`, `percent`, `temperature`, `up_down`, `scene_number`, …)

Export the group addresses from ETS (Group Addresses → Export) to build the map.

---

## 🛠️ Tools

| Tool | What it does |
|---|---|
| `knx_write` | Write a value to a group address (lights, dimmers, shutters, scenes, setpoints) |
| `knx_read` | Read current state/values (single or batch, uses `status_ga` when mapped) |
| `knx_monitor` | Listen to bus traffic for 1–60s, decoded via the map — great for finding unknown GAs |
| `knx_devices` | List the devices configured in `KNX_GA_MAP` |
| `knx_discover` | Multicast discovery of KNX/IP gateways on the LAN |

---

## 📝 Notes

- Tested against a KNX IP interface with plain (non-secure) tunneling, UDP and
  concurrent tunnel slots. **KNX IP Secure is not supported** in this version.
- Multicast discovery (`knx_discover`) and `routing` mode may not work from
  inside containers; unicast `tunneling` is the reliable default.
- A write actuates real devices: the skill instructs the agent to only write
  what the user explicitly asked for.