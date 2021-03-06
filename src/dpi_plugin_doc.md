# DPI plugin for VPP    {#dpi_plugin_doc}

## Overview

DPI plugin can identify and analyze the traffic running on networks in real time.
It can be used on many use cases, such as Web Application Firewall,
Policy based routing, Intrusion Detection System, Intrusion Prevention System, etc.

The main use case for current approach would be identification of cooperating traffic 
for an established TCP connection (i.e. traffic that is not intentionally disguised) 
to support application-based QoS.


## Design

The DPI plugin leverage Hyperscan to perform regex matching.

Hyperscan is a high-performance multiple regex matching library.
Please refer to below for details:
http://intel.github.io/dpi/dev-reference/

Below is the brief design:

1. Provides a default APPID database for detection.

2. Support TCP connection state tracking.

3. Support TCP segments reassembly on the fly, which handles out-of-order tcp segments and overlapping segments.
   It means that we do not need to reassembly segments first, then dedect applicaion, 
   and then fragment segments again, which helps to improve performance.

4. Support Hyperscan Stream mode, which can detect one rule straddling into some tcp segments.
   It means that if there is a rule "abcde", then "abc" can be in packet 1, 
   and "de" can be in packet 2.

5. Configure static dpi flows with 5-tuple and VRF-aware, and supports both ipv4 and ipv6 flows.
   These flows will first try to HW offload to NIC based on DPDK rte_flow mechanism
   and vpp/vnet/flow infrastructure.
   If failed, then will create static SW flow mappings.
   Each flow configuration will create two HW or SW flow mappings, i.e. for forward and reverse traffic.
   And both flow mappings will be mapped to the same dpi flow.
   Dynamically create new SW mapping and aging out mechanism will be added later.

        "dpi flow [add | del]  "
        "[src-ip <ip-addr>] [dst-ip <ip-addr>] "
        "[src-port <port>] [dst-port <port>] "
        "[protocol <protocol>] [vrf-id <nn>]",
        
        "dpi tcp reass flow_id <nn> <enable|disable> "
        "[ <client | server | both> ]",

        "dpi set flow-offload hw <interface-name> rx <flow-id> [del]",

        "dpi set ip4 flow-bypass <interface> [del]",

6. When HW flow offload matched, packets will be redirected to DPI plugin with dpi flow_id in packet descriptor.
   If not, packets will be bypassed to DPI plugin from ip-input, and then lookup SW flow mapping table.

7. Then will detect layer 7 applications.
   This first patch only detect sub protocls within SSL/TLS.
   1). Identify SSL/TLS certificate message and subsequent segments.
   2). Scan SSL/TLS certificate message through hyperscan, and get application id if matched.
   3). If maximum packets for this flow are checked and not found matched application, the detection will end up.

