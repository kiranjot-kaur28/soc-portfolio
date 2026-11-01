# Week 01 – DNS and TCP Handshake with Wireshark

## Objective
Capture my own web traffic and trace one website visit from the DNS lookup to an open, encrypted connection.

## Environment
- Wireshark on Windows 11 (personal laptop), home Wi-Fi with IPv6
- Browser: <chrome>

## Steps
1. Cleared the DNS cache with `ipconfig /flushdns`
2. Captured Wi-Fi traffic for about 60 seconds while visiting wikipedia.org
3. Filtered `dns.qry.name contains "wikipedia"` to find the DNS lookups
4. Filtered `dns.id == 0xd657` to isolate one query and its response
5. Filtered `tcp.port == 64323` to isolate one TCP connection

## Findings

### 1. DNS lookup
(dns-01.png)
- Packet 3927: my PC asked the DNS server for the IPv6 address (AAAA record) of wikipedia.org over UDP port 53.
- Packet 3930: the server answered `2620:0:860:ed1a::1` after 28 ms. Both packets share Transaction ID `0xd657`.
- The browser sent A, AAAA and HTTPS queries at the same time. 

### 2. TCP 3-way handshake
(tcp-handshake-01.png)

| Packet | Direction | Meaning |
|--------|-----------|---------|
| 3932 SYN | My PC (port 64323) → Wikipedia (port 443) | Request to open a connection |
| 3948 SYN-ACK | Wikipedia → My PC | Server agrees and acknowledges |
| 3949 ACK | My PC → Wikipedia | Connection is open |

- The SYN was sent 2 ms after the DNS answer arrived (14.4095 s → 14.4113 s), so the browser used that answer.
- The handshake round trip took about 51 ms (SYN at 14.4113 s, SYN-ACK at 14.4628 s).

### 3. Encryption starts
- Packet 3950 is a TLS 1.3 Client Hello showing `SNI=wikipedia.org`, and packet 3958 is the Server Hello.
- Everything after this is encrypted Application Data.

## What I learned
- The browser opened three connections at once to load the page faster. I filtered out for TCP port 64323
- My network prefers IPv6, so the browser used the AAAA answer instead of the A answer.
- HTTPS hides the page content, but the site name is still visible in the SNI.

## How this applies in a SOC
- DNS logs show which domains a computer looked up. A lookup to a known malicious domain is an early sign of infection.
- Many SYN packets with no completed handshake can indicate a port scan.
- SNI lets network security tools see which site HTTPS traffic goes to without decrypting it.

## Privacy note
My own IP and MAC addresses are blacked out in the screenshots. The raw capture file is not published because it contains all of my network traffic.
