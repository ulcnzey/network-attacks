# TCP Connection Flood

## 1. Overview

TCP Connection Flood is a denial-of-service (DoS) attack that attempts to consume the resources of a target system by creating a large number of TCP connections in a short period of time.

Unlike a TCP SYN Flood, the TCP three-way handshake is completed during a TCP Connection Flood. The attacker repeatedly establishes TCP connections with the target and closes them or keeps them open depending on the attack technique.

In this laboratory study, the attack was performed in an isolated VirtualBox network using a Windows host as the attacking machine and a Kali Linux virtual machine as the target.

The purpose of the experiment is not to disrupt a real system, but to generate controlled attack traffic for the **CyberTrace network intrusion dataset**.

---

## 2. Attack Principle

TCP uses a three-way handshake to establish a connection:

```text
Attacker                         Target
   |                                |
   | -------- SYN ----------------> |
   | <------- SYN/ACK ------------- |
   | -------- ACK ----------------> |
   |                                |
   |       TCP Connection           |
```

After the handshake is completed, a TCP connection is established.

During a TCP Connection Flood, this process is repeated many times in a short period:

```text
SYN
SYN/ACK
ACK
Connection
Connection Close

SYN
SYN/ACK
ACK
Connection
Connection Close

SYN
SYN/ACK
ACK
...
```

The large number of connections increases the workload of the target system's TCP/IP stack and the application listening on the target port.

---

## 3. Purpose of the Attack

The primary purpose of a TCP Connection Flood is to consume resources associated with handling TCP connections.

Potentially affected resources include:

* TCP sockets
* Kernel memory
* File descriptors
* Connection tracking tables
* Server processes or threads
* CPU resources
* Application connection pools

If the attack becomes sufficiently large, the target service may experience increased latency, reduced availability, or complete service disruption.

---

## 4. TCP Connection Flood vs. TCP SYN Flood

TCP Connection Flood and TCP SYN Flood are related but different attacks.

| Feature             | TCP SYN Flood              | TCP Connection Flood                     |
| ------------------- | -------------------------- | ---------------------------------------- |
| SYN packets         | High                       | High                                     |
| SYN/ACK response    | Yes                        | Yes                                      |
| Final ACK           | Usually absent             | Present                                  |
| Three-way handshake | Usually incomplete         | Completed                                |
| Connection state    | Half-open                  | Established                              |
| Main objective      | Exhaust connection backlog | Consume connection and service resources |
| Wireshark pattern   | SYN-heavy traffic          | Repeated complete TCP handshakes         |

This distinction is particularly important for CyberTrace because the two attacks should be represented as separate attack classes.

---

## 5. Laboratory Environment

The experiment was performed in an isolated VirtualBox Host-Only network.

### Attacking Machine

```text
Operating System: Windows
IP Address: 192.168.56.1
Role: Attacker
```

### Target Machine

```text
Operating System: Kali Linux
Interface: eth1
IP Address: 192.168.56.101
Role: Target
```

### Target Service

A simple Python TCP server was used to provide a controlled TCP service.

```text
Protocol: TCP
Target IP: 192.168.56.101
Target Port: 9000
```

The server was intentionally created for the laboratory environment so that the attack traffic could be captured without affecting an external system.

---

## 6. Attack Configuration

The target system was configured to listen for TCP connections on port `9000`.

The Python TCP server accepted incoming connections and immediately closed them.

The attacking Windows system repeatedly established TCP connections to:

```text
192.168.56.101:9000
```

A controlled number of connections was generated during the experiment.

Example configuration:

```text
Target: 192.168.56.101
Port: 9000
Protocol: TCP
Connection Count: 500
```

A larger controlled traffic sample can also be generated when additional PCAP data is required.

---

## 7. Attack Execution

The attack was generated from the Windows machine against the isolated Kali target.

Each connection follows the TCP handshake:

```text
SYN
SYN/ACK
ACK
```

After the connection was established, it was closed and a new connection was created.

This process was repeated many times.

The resulting traffic therefore contains a large number of TCP connection establishment and termination sequences within a relatively short period.

---

## 8. Wireshark Capture

Wireshark was used on the Kali machine to capture the traffic on the `eth1` interface.

The following display filter can be used to isolate the attack traffic:

```text
ip.src == 192.168.56.1 && ip.dst == 192.168.56.101 && tcp.port == 9000
```

### SYN Packets

To examine TCP connection initiation:

```text
ip.src == 192.168.56.1 && ip.dst == 192.168.56.101 && tcp.flags.syn == 1 && tcp.flags.ack == 0
```

### SYN/ACK Packets

To examine the target's responses:

```text
ip.src == 192.168.56.101 && ip.dst == 192.168.56.1 && tcp.flags.syn == 1 && tcp.flags.ack == 1
```

### ACK Packets

To identify completed handshakes:

```text
ip.src == 192.168.56.1 && ip.dst == 192.168.56.101 && tcp.flags.ack == 1
```

---

## 9. Observed Traffic Characteristics

The most important characteristic of the generated traffic is the repeated establishment of TCP connections.

A typical sequence appears as:

```text
Client → Server     SYN
Server → Client     SYN/ACK
Client → Server     ACK
Client → Server     Connection/Data
Connection Close
```

This sequence is repeated many times.

The following characteristics can therefore be observed in the PCAP:

* High number of TCP packets
* High number of connection attempts
* Repeated SYN packets
* Repeated SYN/ACK responses
* Repeated ACK packets
* Completed TCP handshakes
* Repeated connection termination
* High connection rate
* Same source and destination pair
* Same destination port

These characteristics can later be used as features for machine learning.

---

## 10. TCP Connection States

TCP connection states can also be inspected on the Kali target using:

```bash
ss -ant
```

The following command can be used to focus on port `9000`:

```bash
ss -ant | grep ':9000'
```

Depending on the timing of the capture, states such as the following may be observed:

```text
ESTAB
TIME-WAIT
CLOSE-WAIT
```

A large number of short-lived connections can result in an increase in `TIME-WAIT` entries because TCP maintains recently closed connections for a period of time.

---

## 11. Impact on the Target

A TCP Connection Flood can cause the target system to spend a significant amount of resources processing connection requests.

Potential effects include:

1. Increased CPU usage
2. Increased memory consumption
3. Increased number of TCP sockets
4. Increased connection tracking activity
5. Increased application workload
6. Increased response latency
7. Reduced service availability

The severity depends on the attack rate, target configuration, operating system, application architecture, and available system resources.

The controlled attack performed in this laboratory was designed to generate representative traffic rather than cause actual service disruption.

---

## 12. Detection Indicators

A network intrusion detection system can look for several indicators of a TCP Connection Flood.

Possible indicators include:

* High TCP connection rate
* Large number of connections from one source
* Large number of connections to the same destination
* Repeated SYN → SYN/ACK → ACK sequences
* High packet rate
* High number of short-lived TCP connections
* Large number of `TIME-WAIT` states
* Repeated connections to the same destination port

A simplified detection concept can be represented as:

```text
High connection rate
        +
Repeated TCP handshakes
        +
Same source/destination
        +
Short connection duration
        ↓
Potential TCP Connection Flood
```

---

## 13. Features for CyberTrace Dataset

The captured traffic can later be transformed into machine-learning features.

Potential features include:

| Feature              | Description                         |
| -------------------- | ----------------------------------- |
| `src_ip`             | Source IP address                   |
| `dst_ip`             | Destination IP address              |
| `src_port`           | Source TCP port                     |
| `dst_port`           | Destination TCP port                |
| `protocol`           | Network protocol                    |
| `packet_count`       | Number of packets                   |
| `byte_count`         | Total transferred bytes             |
| `flow_duration`      | Duration of the network flow        |
| `packets_per_second` | Packet transmission rate            |
| `bytes_per_second`   | Byte transmission rate              |
| `syn_count`          | Number of SYN packets               |
| `syn_ack_count`      | Number of SYN/ACK packets           |
| `ack_count`          | Number of ACK packets               |
| `fin_count`          | Number of FIN packets               |
| `rst_count`          | Number of RST packets               |
| `connection_rate`    | Number of connections per unit time |
| `label`              | Attack classification               |

The most important features for this attack are expected to include `connection_rate`, packet rate, TCP flag counts, and the number of completed connections.

---

## 14. CyberTrace Dataset Label

The generated traffic should be labelled as:

```text
TCP_Connection_Flood
```

Example dataset record:

```text
Source IP:        192.168.56.1
Destination IP:   192.168.56.101
Protocol:         TCP
Destination Port: 9000
Attack Type:      TCP_Connection_Flood
```

The label allows the generated traffic to be combined with other attack classes and normal network traffic during the later machine-learning stage of CyberTrace.

---

## 15. PCAP File

The captured network traffic is stored as:

```text
tcp_connection_flood.pcapng
```

Suggested repository location:

```text
tcp-connection-flood/tcp_connection_flood.pcapng
```

The PCAP file contains the TCP traffic generated during the controlled laboratory experiment.

---

## 16. Screenshots

### Wireshark – TCP Connection Flood Overview

```text
[SCREENSHOT PLACEHOLDER]
```

The screenshot should show the large number of TCP connections generated during the experiment.

### TCP Three-Way Handshake

```text
[SCREENSHOT PLACEHOLDER]
```

The screenshot should show:

```text
SYN
SYN/ACK
ACK
```

and demonstrate that the TCP handshake is completed.

### TCP Connection Rate

```text
[SCREENSHOT PLACEHOLDER]
```

The screenshot should demonstrate the high frequency of TCP connection establishment during the attack.

---

## 17. Conclusion

The TCP Connection Flood experiment demonstrated how a large number of TCP connections can be generated against a target service in a controlled network environment.

Unlike a TCP SYN Flood, the TCP handshake was completed, resulting in real TCP connections between the attacking and target systems.

The resulting PCAP provides a useful example of TCP connection-based DoS traffic for the CyberTrace project.

The traffic can later be processed to extract network-flow features and combined with other attack and normal traffic samples to create a labelled dataset for machine-learning-based intrusion detection.

---

## 18. Experiment Summary

| Parameter         | Value                         |
| ----------------- | ----------------------------- |
| Attack            | TCP Connection Flood          |
| Category          | Denial of Service             |
| Protocol          | TCP                           |
| Attacker          | Windows                       |
| Attacker IP       | 192.168.56.1                  |
| Target            | Kali Linux                    |
| Target IP         | 192.168.56.101                |
| Target Port       | 9000                          |
| Capture Interface | eth1                          |
| PCAP Format       | PCAPNG                        |
| Dataset Label     | `TCP_Connection_Flood`        |
| Environment       | Isolated VirtualBox Network   |
| Purpose           | CyberTrace Dataset Generation |
