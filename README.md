# Cisco ACI FTAG Viewer for new versions

**ACI FTAG tree viewer for inspecting Cisco ACI multicast forwarding trees from the APIC CLI or offline captures.**

[![Python](https://img.shields.io/badge/Python-3.8%2B-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

`aci-ftag-viewer` is a Python tool for Cisco Application Centric Infrastructure (ACI) FTAG tree visualization and troubleshooting. It reads Cisco ACI managed objects, correlates FTAG, IS-IS, and LLDP information, and displays a human-readable **switch hierarchy**, the **interfaces at both ends of each fabric link**, and **per-pod reachability findings**.

> **Read-only:** All live fabric data is retrieved using APIC-local `icurl` GET requests. The tool does not change fabric configuration.

## Quick start

Copy `aci_ftag_viewer.py` to your Cisco APIC and run it with Python 3.

**Show FTAG 4 across all pods:**

```bash
python3 aci_ftag_viewer.py --ftag 4
```

**Show FTAG 4 in Pod 1 only:**

```bash
python3 aci_ftag_viewer.py --pod 1 --ftag 4
```

**Show all FTAGs in all pods:**

```bash
python3 aci_ftag_viewer.py
```

## More examples

| Goal | Command |
| --- | --- |
| Hide external OIF references | `python3 aci_ftag_viewer.py --pod 1 --ftag 4 --no-external` |
| Use ASCII-only characters | `python3 aci_ftag_viewer.py --ftag 4 --ascii` |
| Show detailed debug messages | `python3 aci_ftag_viewer.py --pod 1 --ftag 4 --debug debug` |
| Save the output to a file | `python3 aci_ftag_viewer.py --ftag 4 > /tmp/ftag4-report.txt 2>&1` |
| See all CLI arguments | `python3 aci_ftag_viewer.py --help` |
| Print offline data collection instructions | `python3 aci_ftag_viewer.py --offlineHelp` |

**Transfer and execute** (replace `<APIC_USER>` and `<APIC_HOST>` and adapt the destination to your environment):

```bash
# On your workstation:
scp aci_ftag_viewer.py "<APIC_USER>@<APIC_HOST>:/data/techsupport"

# On the APIC:
python3 --version
python3 aci_ftag_viewer.py --ftag 4
```

You can also run the script directly after making it executable:

```bash
chmod +x aci_ftag_viewer.py
./aci_ftag_viewer.py --ftag 4
```

## Example output

The following sanitized example illustrates a three-pod FTAG 4 analysis. Switch labels are role-and-ID labels, not hostnames. External peers and addresses are placeholders; no actual device hostnames are included.

### Pod 1: a tree with one switch not reached by traversal

```text
========================================================================
POD 1  |  FTAG 4  |  CHECK
Root: spine-201     Switches in tree: 7/8     Not reached: 1
========================================================================

FABRIC FTAG TREE  (switch hierarchy; link ownership shown on each line)
  spine-201  [ROOT]
  ├── leaf-102
  │   └── spine-201 Eth1/2 <--> leaf-102 Eth1/49
  ├── leaf-103
  │   └── spine-201 Eth1/3 <--> leaf-103 Eth1/49
  └── leaf-104
      │   spine-201 Eth1/4 <--> leaf-104 Eth1/49
      ├── spine-202
      │   └── leaf-104 Eth1/50 <--> spine-202 Eth1/4
      └── spine-203
          │   leaf-104 Eth1/51 <--> spine-203 Eth2/4
          └── leaf-101
              └── spine-203 Eth2/1 <--> leaf-101 Eth1/51

FABRIC INTERFACE TABLE  (validated FTAG tree links)
  FROM SWITCH  LOCAL PORT  TO SWITCH  REMOTE PORT
  -----------  ----------  ---------  -----------
  spine-201    Eth1/2      leaf-102   Eth1/49
  spine-201    Eth1/3      leaf-103   Eth1/49
  spine-201    Eth1/4      leaf-104   Eth1/49
  leaf-104     Eth1/50     spine-202  Eth1/4
  leaf-104     Eth1/51     spine-203  Eth2/4
  spine-203    Eth2/1      leaf-101   Eth1/51

EXTERNAL OIF REFERENCES  (6; not verified as ACI adjacencies)
  Local ACI port -> LLDP peer port and name
  spine-201
    Eth1/11  -> Eth2/13   <EXTERNAL_PEER_1>
    Eth1/12  -> Eth2/9    <EXTERNAL_PEER_2>
  spine-202
    Eth1/11  -> Eth2/14   <EXTERNAL_PEER_1>
    Eth1/12  -> Eth2/10   <EXTERNAL_PEER_2>
  spine-203
    Eth2/11  -> Eth2/15   <EXTERNAL_PEER_1>
    Eth2/12  -> Eth2/11   <EXTERNAL_PEER_2>

NOT REACHED BY FTAG TRAVERSAL  (1; investigate)
  spine-2001  |  pod 1  |  address <SWITCH_ADDRESS>  |  IS-IS <system-id>
```

### Across pods: consolidated summary

When several pods are included, the viewer finishes with a summary such as:

```text
========================================================================
FTAG 4 - POD SUMMARY
========================================================================
POD   ROOT         IN TREE    UNVALID OIFS  NOT REACHED
------------------------------------------------------------------------
1     spine-201    7/8        0             2001
2     spine-1201   2/2        0             -
3     spine-2201   6/6        0             -

Note: 'Not reached' refers to this FTAG software traversal, not proof
      that a switch or physical interface is down.
```

In this example, Pod 1 needs additional investigation of node 2001; Pods 2 and 3 have no nodes missing from the script's traversal.

## Offline analysis

Offline mode is useful when you want to collect the relevant ACI MIT objects once and analyze them away from a live APIC—for example, during an incident review.

### 1. Generate the collection commands on the APIC

```bash
python3 aci_ftag_viewer.py --offlineHelp
```

This prints a set of local `icurl` queries and shell commands to create `/tmp/offline_data.tgz`. **Run those collection commands on the APIC**, where the local API is available. The generated collection includes these managed-object classes:

```text
fabricNode
fmcastTreeEp
isisFmcastTree
isisOifListLeaf
isisOifListSpine
isisAdjEp
isisDom
l3extRsPathL3OutAtt
lldpAdjEp
topSystem
```

Keep the generated file naming convention (`off_<ClassName>.json`) intact. The offline loader relies on those class-name patterns.

> On large fabrics, check that the captured JSON files contain the full expected objects and are not truncated or limited by API pagination. Offline accuracy is limited by the completeness and consistency of the capture.

### 2. Transfer the `.tgz` archive to your analysis host

Use your normal secure file-transfer mechanism. The archive can contain internal topology details, switch identifiers, and addresses; **do not commit actual capture files or unredacted reports to a public repository**.

### 3. Run offline analysis with Python 3

```bash
python3 aci_ftag_viewer.py --offline ./offline_data.tgz --ftag 4
```

You can combine `--offline` with `--pod`, `--ascii`, `--no-external`, and other display options.

## Command-line reference

| Option | Description |
| --- | --- |
| `--ftag <ID>` | Analyze a specific FTAG. If omitted, analyze all available FTAGs. |
| `--pod <ID>` | Limit analysis to one pod. If omitted, analyze all pods. |
| `--ascii` | Draw the tree using ASCII characters for terminals that do not display Unicode box characters. |
| `--no-external` | Hide the external OIF references section. |
| `--debug {debug,info,warn,error}` | Choose the logging verbosity; default is `info`. |
| `--offline <FILE.tgz>` | Analyze captured API data from a `.tgz` bundle rather than querying an APIC. |
| `--offlineHelp` | Print the commands needed to collect an offline data bundle on the APIC. |
| `-h`, `--help` | Display command-line help. |

## How to read the topology

Each **switch name** is a node in the logical FTAG tree. Each connection line explicitly names **both switches and their interfaces**.

For example:

```text
spine-201 [ROOT]
└── leaf-104
    │   spine-201 Eth1/4 <--> leaf-104 Eth1/49
    └── spine-202
        └── leaf-104 Eth1/50 <--> spine-202 Eth1/4
```

This means:

1. `spine-201 Eth1/4` is connected to `leaf-104 Eth1/49`.
2. `leaf-104 Eth1/50` is connected to `spine-202 Eth1/4`.
3. **Within the reconstructed FTAG 4 tree**, the path from root `spine-201` to `spine-202` goes **through `leaf-104`**.

The `<-->` marker identifies the two ends of a connection; **it does not mean that the FTAG forwards traffic in both directions along that edge**. The indentation expresses the root-to-child hierarchy. It is a **logical FTAG tree**, **not** a map of every physical cable in the fabric.

### Output sections

| Section | What it tells you |
| --- | --- |
| **FABRIC FTAG TREE** | The root and the switch hierarchy reconstructed from the FTAG OIFs and adjacency information. |
| **FABRIC INTERFACE TABLE** | A flat table of the same validated tree edges and their two endpoint interfaces. |
| **EXTERNAL OIF REFERENCES** | IPN-/external-facing ports and LLDP peer information, shown separately from internal fabric links. |
| **UNVALIDATED FABRIC OIFS** | Internal OIFs that the script could not validate, including the available reason. This section appears when relevant. |
| **NOT REACHED BY FTAG TRAVERSAL** | In-service switches in the pod that the walk did not discover. This section appears when relevant. |
| **POD SUMMARY** | Root, number of switches reached, unvalidated OIF count, and nodes not reached for each analyzed pod. |

## How it works

At a high level, the script:

1. **Builds the in-service switch inventory** from `topSystem` (leaf/spine nodes, pod IDs, names, and addresses).
2. **Correlates identities and adjacencies** using `isisDom`, `lldpAdjEp`, and `isisAdjEp`.
3. **Finds FTAG roots and per-node tree state** from `fmcastTreeEp` and `isisFmcastTree`.
4. **Loads per-tree OIFs** from `isisOifListLeaf` and `isisOifListSpine`.
5. **Walks the tree from each root**, checking the corresponding adjacency, peer OIF, and pod membership before following a fabric link.
6. **Separates external OIF references**, using L3Out path information and LLDP metadata where available, and reports links that could not be validated.
7. **Formats the results** as a hierarchy, interface table, investigation sections, and pod summary.

Live API queries are issued through APIC-local `icurl` against `http://127.0.0.1:7777/api/...`. The script does not require direct HTTP access to the APIC API from your workstation when run on an APIC.

## Understanding results and limitations

### `COMPLETE (script traversal)`

All in-service switches in the analyzed pod were reached by the script's FTAG traversal, and the output contains no recorded unvalidated fabric OIFs for that walk. **This is not an end-to-end multicast health certification** and does not independently test data-plane traffic.

### `CHECK`

The report includes at least one switch not reached by the walk or an unvalidated internal OIF. Investigate the relevant FTAG state, OIF information, and LLDP/IS-IS adjacency on the affected nodes.

### `NOT REACHED BY FTAG TRAVERSAL`

The switch appears in the script's **in-service** inventory but was not discovered by its walk from the selected FTAG root. This **does not automatically mean** the switch is powered off, its interfaces are down, or multicast traffic is being dropped.

### `UNVALIDATED FABRIC OIFS`

The script could not prove that a candidate internal FTAG OIF maps to a valid link in this tree. Reasons may include absent/incomplete adjacency information, a missing reciprocal OIF, or a pod mismatch. Review the raw ACI objects and live operational state to determine the cause.

### `EXTERNAL OIF REFERENCES`

These entries are reported from external-facing FTAG OIFs and LLDP peer metadata. They **are not validated as internal ACI leaf/spine adjacencies**, and their presence does not prove that the IPN or another external network is healthy.

### Other boundaries

- A tree is analyzed **per pod and FTAG**. The display does not reconstruct a full end-to-end inter-pod IPN path.
- The tree intentionally includes **only the accepted FTAG edges**; it is not a complete physical fabric topology or cable inventory.
- Results are a **snapshot** of APIC operational data. Changing state during collection can affect consistency.
- Nodes excluded from the `topSystem` in-service inventory are not included in the expected switch count. Old configuration objects may still refer to those nodes.
- The program is a troubleshooting aid, not a substitute for switch-level verification, packet captures, or ELAM when diagnosing data-plane behavior.

## Troubleshooting

**`skipping unknown node` in debug output**

The queried MIT data references a node that is absent from the script's in-service leaf/spine inventory. This can happen for APIC controllers (not part of the switch traversal), out-of-service switches, or retained configuration references. Review the specific node if it matters to the current FTAG investigation; the message by itself is not proof of failure.

**A pod has a `Not reached` count greater than zero**

Run a focused report with debug logging:

```bash
python3 aci_ftag_viewer.py --pod 1 --ftag 4 --debug debug
```

Check the affected switch's FTAG `operSt`, `rootPort`, OIF list, IS-IS overlay adjacency, and the reciprocal OIF/adjacency toward the root.

**Tree characters are garbled**

```bash
python3 aci_ftag_viewer.py --ftag 4 --ascii
```

**Live API query fails / script says the device is unsupported**

Verify that the command is running on an APIC with access to its local `icurl` API. Offline analysis requires `--offline` and a correctly collected JSON archive instead.

**Need more logging without changing the display?**

Use `--debug debug`. Normal runs intentionally suppress routine unknown-node inventory messages to keep reports readable.

## License

Distributed under the [MIT License](LICENSE). See the repository's license file for the applicable copyright notice and terms.

---

**Project:** [ehaminian/aci-ftag-viewer](https://github.com/ehaminian/aci-ftag-viewer) · **Platform:** Cisco ACI / APIC · **Purpose:** FTAG visibility and troubleshooting
