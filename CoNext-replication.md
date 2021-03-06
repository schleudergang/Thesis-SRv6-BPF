# Instructions for replicating our results

We hereby describe how to replicate the results described in the paper _"Leveraging eBPF for programmable network functions with IPv6 Segment Routing"_, presented at CoNext 2018.
Whenever the installation of a BPF program is required, we suppose that our custom kernel [available here](https://github.com/Zashas/linux-seg6-bpf) (Linux 4.18 with minor modifications), iproute2 4.18, LLVM >= 6.0, [bcc](https://github.com/iovisor/bcc) and our fork of [pyroute2](https://github.com/Zashas/pyroute2) are installed on the machine hosting the program.

The following [configuration script](https://raw.githubusercontent.com/target0/thesis-data/master/comp4-data/setup.sh) (adapted for each machine) was  always executed prior to any measure, tweaking the parameters of our NICs and assigning a single core for packet forwarding.

## Performance evaluation of section 3.3

1. Setup a functional IPv6 network such as described in Figure 1, Setup 1.
1. Install on R using iproute2 on the SRv6 SID _R_BPF_ the following actions:
    1. Static endpoints: End, End.T (`ip -6 route add SID6 encap seg6local action End/End.T`)
    1. The BPF programs _pass_ (End BPF), _end\_t_ (End.T BPF), _inc\_tag_ (Tag++ BPF), _add\_8_ (Add TLV BPF) from [here](seg6-bpf-tests/tests_bpf.c) and [here](seg6-bpf-tests/tlv_bpf.c) (`ip -6 route add SID6 encap seg6local action End.BPF endpoint obj OBJECT_FILE.o sec FCT_NAME`). Compile the object files using `make`.
    1. Enable/disable the BPF JIT using `# sysctl net.core.bpf_jit_enable=1/0`
1. Generate packets using trafgen from S1 with segments _[R_BPF, S2]_  and the parameters described in the article, measure the number of packets per second received on S2.

## Performance evaluation of section 4.1

1. Setup a functional IPv6 network such as described in Figure 1, Setup 1.
1. Download the files from [POWD-monitoring](use-cases/POWD-monitoring).
1. Measuring the performance of the encapsulation program:
    1. Compile on R the _dm\_injector_ files using `make`.
    1. Install on Rthe BPF program DM TLV injector using `ip -6 route add PREFIX encap bpf out obj dm_injector_bpf.o sec main headroom 112 dev IFOUT` with  `PREFIX` the IPv6 prefix and `IFOUT` the interface towards S2.
    1. Configure the encapsulation parameters: `./dm_injector_usr SID-OTP FREQUENCY CONTROLLER-IP6 CONTROLLER-DPORT`, with: `SID-OTP` the address of S2, `FREQUENCY` the probing ratio, and the IP:port tuple for the controller (irrelevant here).
    1. Use pktgten to generate IPv6 packets from S1 towards S2, without SRH. The script used is available [here](https://raw.githubusercontent.com/target0/thesis-data/master/comp4-data/pktgen.sh). Measure the number of packets per second received on S2.
1. Measuring the performance of _End.DM_:
    1. Craft a packet configuration for trafgen, IPv6 UDP 64 bits payload packets from S1 with segments _[R_DM, S2]_ and a valid DM TLV.
    1. Install on R the _End.DM_ program on SID _R\_DM_ using `./end_otp.py R_DM IFOUT`, with `IFOUT` any physical interface.
    1. Use trafgen to generate DM probes from S1. Measure the number of packets per second received on S2.

## Performance evaluation of section 4.2

1. Flash a Turris Omnia with our [OpenWRT `End.BPF` fork](https://github.com/Zashas/openwrt-seg6).
1. Setup a functional IPv6 network such as described in Figure 1, Setup 2. Enable native SRv6 support `sysctl net.ipv6.conf.all.seg6_enabled=1`, and on the used inbound interfaces: `sysctl net.ipv6.conf.IFACE.seg6_enabled=1`.
1. Download the files from [use-cases/Links-aggreg](Links-aggreg). Compile the source code in _cpe\_bpf_ using `make`. Configure the initial `tc netem` policies (such as in the [simulation script](use-cases/Links-aggreg/simulation.sh)).
1. On the Turris Omnia:
        1. Install the two-way delay measurement function: `ip -6 route add SID-OTP-CPE encap seg6local action End.BPF endpoint obj cpe_bpf/end_otp_bpf.o section end_otp dev IFACE`, with `IFACE` any physical interface.
        1. Install the WRR encapsulation `ip -6 route add default encap bpf in obj cpe_bpf/uplink_wrr_bpf.o section main dev IFACE`, with `IFACE` any physical interface.
        1. Configure the BPF programs by running `cpe_bpf/end_otp_usr` and `cpe_bpf/uplink_wrr_usr IP6-AGG SEG-LINK1 WEIGHT-LINK1 SEG-LINK2 WEIGHT-LINK2`. `SEG-LINK1/2` are the intermediate segments forcing the usage of a specific link.
1. On the aggregator: start the aggregation daemon using `./link_aggreg.py PREFIX-CPE SID-CPE SID1-UP SID1-DOWN WEIGHT1 SID2-UP SID2-DOWN WEIGHT2 SID-OTP-CPE SID-OTP-AGG IFACE-DOWN1,IFACE-DOWN2,... IFACE-UP1,IFACE-UP2,...`, with `PREFIX-CPE` the prefix given to the CPE, `SID-CPE` an IPv6 loopback address on the CPE, `SID1-UP` a SID on the uplink path of WAN 1, `SID1-DOWN` a SID on the downlink path of WAN 1, `SID2-UP` a SID on the uplink path of WAN 2, `SID2-DOWN` a SID on the downlink path of WAN 2, `IFACE-DOWN*` the intefaces of the aggregator towards the WANs, and `IFACE-UP*` the bridges interfaces installed by the aggregator to deploy the uplink `tc netem` policies (tc cannot delay incoming traffic, only outgoing, hence the need for extra bridges).
1. Use nttcp to generate TCP connections between S1 and S2 and measure the aggregated bandwidth.
