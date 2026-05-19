# POX

POX is a networking software platform written in Python.

POX started life as an OpenFlow controller, but can now also function as an
OpenFlow switch, and can be useful for writing networking software in
general.  It currently supports OpenFlow 1.0 and includes special support
for the Open vSwitch/Nicira extensions.

POX versions are named.  Starting with POX "gar", POX officially requires
Python 3.  The last version with support for Python 2 was POX "fangtooth".
POX should run under Linux, Mac OS, and Windows.  (And just about anywhere
else -- we've run it on Android phones, under FreeBSD, Haiku, and elsewhere.
All you need is Python!)  Some features are not available on all platforms.
Linux is the most featureful.

This README contains some information to get you started, but is purposely
brief.  For more information, please see the full documentation.


## Running POX

`pox.py` boots up POX. It takes a list of component names on the command line,
locates the components, calls their `launch()` function (if it exists), and
then transitions to the "up" state.

If you run `./pox.py`, it will attempt to find an appropriate Python 3
interpreter itself.  In particular, if there is a copy of PyPy in the main
POX directory, it will use that (for a potentially large performance boost!).
Otherwise it will look for things called `python3` and fall back to `python`.
You can also, of course, invoke the desired Python interpreter manually
(e.g., `python3 pox.py`).

The POX commandline optionally starts with POX's own options (see below).
This is followed by the name of a POX component, which may be followed by
options for that component.  This may be followed by further components
and their options.

  ./pox.py [pox-options...] [component] [component-options...] ...

### POX Options

While components' options are up to the component (see the component's
documentation), as mentioned above, POX has some options of its own.
Some useful ones are:

 | Option        | Meaning                                                   |
 | ------------- | --------------------------------------------------------- |
 |`--verbose`    | print stack traces for initialization exceptions          |
 |`--no-openflow`| don't start the openflow module automatically             |


## Components

POX components are basically Python modules with a few POX-specific
conventions.  They are looked for everywhere that Python normally looks, plus
the `pox` and `ext` directories.  Thus, you can do the following:

  ./pox.py forwarding.l2_learning

As mentioned above, you can pass options to the components by specifying
options after the component name.  These are passed to the corresponding
module's `launch()` funcion.  For example, if you want to run POX as an
OpenFlow controller and control address or port it uses, you can pass those
as options to the openflow._01 component:

  ./pox.py openflow.of_01 --address=10.1.1.1 --port=6634


## Further Documentation

The full POX documentation is available on GitHub at
https://noxrepo.github.io/pox-doc/html/

## SDN-Based Firewall Project Explanation

### What is SDN?
Software Defined Networking (SDN) is a networking approach where the **control plane** (decision making — who can talk to whom) is **separated** from the **data plane** (actual packet forwarding). Instead of each switch deciding on its own, a **central controller** makes all the decisions and programs the switches accordingly.

---

### What This Project Does
We built a **controller-based firewall** using SDN principles. Traditionally, firewalls are hardware devices. In our project, the **POX controller acts as the firewall brain** — it decides which traffic to allow or block, and programs the OpenFlow switch (OVS) to enforce those decisions.

---

### Components Used
| Component | Role |
|---|---|
| **Mininet** | Creates a virtual network with hosts and switches |
| **POX Controller** | The SDN controller that acts as the firewall |
| **Open vSwitch (OVS)** | The virtual switch that enforces the firewall rules |
| **OpenFlow Protocol** | Communication protocol between POX and OVS |

---

### How It Works — Flow of Events

1. Mininet starts a virtual network with **3 hosts (h1, h2, h3)** connected to **1 switch (s1)**
2. The switch connects to the **POX controller** via OpenFlow
3. POX immediately **pushes drop rules** into the switch for blocked traffic
4. When a packet arrives at the switch, it checks the rules and either **forwards or drops** it
5. All blocked packets are **logged** to `blocked_packets.log` with timestamps

---

### Firewall Rules We Implemented
| Rule | Type | Action |
|---|---|---|
| h1 (10.0.0.1) → h3 (10.0.0.3) | ICMP (ping) | ❌ BLOCKED |
| h2 (10.0.0.2) → h3 (10.0.0.3) | TCP (SSH) | ❌ BLOCKED |
| All other traffic | Any | ✅ ALLOWED |

---

### Key Concepts Demonstrated
- **Rule-based filtering** — traffic filtered by IP, MAC, and protocol/port
- **Drop rules via OpenFlow** — `ofp_flow_mod` with empty action list = drop
- **ARP handling** — ARP packets are flooded so hosts can discover each other
- **MAC learning** — switch learns which port each host is on dynamically
- **Packet logging** — all blocked packets are recorded with timestamps

---

### Complete List of All Commands

#### Setup
```bash
# Install Mininet
sudo apt-get install mininet

# Clone POX
git clone https://github.com/noxrepo/pox
cd pox

# Test Mininet works
sudo mn --test pingall
```

#### Running the Project
```bash
# Terminal 1 — Start POX Firewall Controller
cd ~/Desktop/CN_SDN/pox
python3 pox.py log.level --DEBUG firewall_controller

# Terminal 2 — Clean old Mininet state
sudo mn -c

# Terminal 2 — Start Mininet Topology
sudo python3 firewall_topo.py
```

#### Testing Inside Mininet CLI
```bash
# Test allowed traffic — should SUCCEED
h1 ping h2 -c 3

# Test blocked traffic — should FAIL
h1 ping h3 -c 3

# Full connectivity test
pingall
```

#### Verifying Flow Rules on Switch
```bash
# Check drop rules are installed on switch s1
sudo ovs-ofctl dump-flows s1
```

#### Checking Logs
```bash
# View full log
cat ~/Desktop/CN_SDN/pox/blocked_packets.log

# Watch log update live
tail -f ~/Desktop/CN_SDN/pox/blocked_packets.log
```

### Expected Final Output
```
# pingall result:
h1 -> h2 X        (h1→h3 blocked, h1→h2 works)
h2 -> h1 X        (h2→h3 blocked, h2→h1 works)
h3 -> h1 h2       (h3 can reach everyone)

# blocked_packets.log:
2026-04-21 10:00:01 - ========== Controller Started ==========
2026-04-21 10:00:02 - [DROP RULE INSTALLED] 10.0.0.1 -> 10.0.0.3 (icmp)
2026-04-21 10:00:02 - [DROP RULE INSTALLED] 10.0.0.2 -> 10.0.0.3 (tcp)
2026-04-21 10:00:15 - BLOCKED: 10.0.0.1 -> 10.0.0.3 | Protocol: icmp
2026-04-21 10:00:22 - BLOCKED: 10.0.0.2 -> 10.0.0.3 | Protocol: tcp
```
