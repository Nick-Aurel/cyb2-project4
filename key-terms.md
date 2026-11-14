# CYB102 Project 4 — Key Technical Terms

Definitions for the terms used in Unit 4 (DoS mitigation with nginx, Slowloris, Netdata, and packet captures).

## Attacks

**Denial of Service (DoS)**  
An attack that tries to make a service unavailable to legitimate users. In this project the attacker works from a single machine and exhausts a resource the server needs, such as the number of open connections it will accept.

**Distributed Denial of Service (DDoS)**  
The same goal as a DoS attack, but the traffic comes from many machines at once. That makes the attack larger and harder to block by source address. This lab uses a single-source DoS attack, not a DDoS attack.

**Slowloris**  
An application-layer DoS attack that opens many connections to a web server and keeps each one alive by sending an HTTP request very slowly, a few bytes at a time, and never finishing it. The server holds those connections open until it runs out of connection slots. The lab tool is `slowloris-run`; `-s 500` means open 500 sockets against the target.

**Volumetric attack (flood attack)**  
A DoS style that overwhelms a target with a huge amount of traffic — bandwidth, packets, or requests per second. Defenses often look for that volume. Slowloris is designed to avoid looking like a flood: each connection sends very little data.

**Application-layer DoS**  
A DoS attack that abuses how an application protocol works, here HTTP, instead of simply flooding the network. Each Slowloris connection can look like an ordinary, slow client, so blocking it at the network edge by volume alone does not work. The server has to enforce limits on how requests are allowed to behave.

**Partial HTTP request**  
An HTTP request that has been started but not completed. Slowloris sends headers slowly and never sends the blank line that marks the end of the headers, so nginx keeps the connection open waiting for the rest.

## Defense

**Mitigation**  
A control that reduces the impact of an attack. It does not have to make the attack impossible. In this lab, nginx rules limit how long and how many half-finished connections the server will tolerate.

**Rate limiting**  
A server rule that caps how often a client may send requests or how fast it may send data. Slowloris depends on being allowed to drip bytes forever; a rate or timeout limit cuts those connections off.

**Connection timeout**  
A limit on how long the server will wait for a client to finish sending (or for a connection to stay idle). If the client is too slow, nginx closes the connection and frees the slot for someone else.

**Resource boundary**  
A limit the server sets on something it controls: open connections, request time, request rate, or header size. Slowloris works by consuming one of those resources. The defense is to put a ceiling on it.

**Network perimeter**  
The outer edge of a network, where firewalls and filters usually sit. Application-layer attacks are hard to stop there because each packet can look like normal HTTP. The useful checks often live in the web server configuration.

**Bandwidth**  
The amount of data a link can carry over time. Adding bandwidth helps against floods. It does little against Slowloris, which uses connections, not a large amount of data.

## Web server

**nginx**  
The web server running on the lab VM. It accepts HTTP connections on port 80 and is the place where the DoS mitigation rules are configured. Changes are checked with `sudo nginx -t` and applied with `sudo systemctl reload nginx`.

**HTTP (Hypertext Transfer Protocol)**  
The request/response protocol browsers and clients use to talk to web servers. A normal request is sent completely and promptly. Slowloris breaks that expectation by never finishing the request.

**Port 80**  
The default TCP port for unencrypted HTTP. If nginx is already running, another process cannot bind to port 80 at the same time.

**`nginx.conf`**  
The main nginx configuration file, usually at `/etc/nginx/nginx.conf`. Global settings, including some connection and timeout limits, belong here.

**`sites-available`**  
The directory `/etc/nginx/sites-available/` where per-site config files live. The lab’s default site file is `/etc/nginx/sites-available/default`. Editing this file is one of the two allowed ways to add mitigation rules.

**`conf.d`**  
The directory `/etc/nginx/conf.d/`. nginx loads every `.conf` file in it. Creating `/etc/nginx/conf.d/default.conf` is the other allowed way to add the mitigation rules.

**`nginx -t`**  
An nginx command that tests the configuration files for syntax errors without applying them. The lab chains it with the reload: `sudo nginx -t && sudo systemctl reload nginx`, so a broken config is not loaded.

**`systemctl reload`**  
A systemd command that tells a running service to reread its configuration without a full stop and start. `sudo systemctl reload nginx` applies new rules while nginx keeps running.

## Monitoring

**Netdata**  
A real-time monitoring tool installed on the VM. Its local dashboard is at `http://127.0.0.1:19999`. The lab uses it to watch what the attack does to the server while Slowloris is running.

**Dashboard**  
The Netdata page that shows live graphs. For this project it needs two charts: Network Traffic (`networks`) and TCP Connections (`tcpsock`).

**Network traffic (`networks`)**  
The Netdata chart of data moving in and out of the machine’s network interfaces. During Slowloris this stays relatively quiet, because the attack sends little data.

**TCP connections (`tcpsock`)**  
The Netdata chart of TCP sockets and their states. This is the graph that shows the attack: an unprotected server accumulates a large number of open connections; a protected server should hold that number down because nginx closes slow connections.

**Kickstart**  
Netdata’s install script. The lab uses the course’s kickstart instructions to install Netdata on the VM.

## Networking

**TCP (Transmission Control Protocol)**  
The transport protocol HTTP runs on. TCP sets up a connection, delivers bytes in order, and keeps that connection open until one side closes it. Slowloris depends on TCP connections staying open.

**Socket**  
One endpoint of a network connection. `slowloris-run 127.0.0.1 -s 500` asks the tool to open 500 sockets to the local server, each one a separate connection nginx must track.

**TCP connection**  
A session between two sockets after the handshake completes. Each unfinished Slowloris request occupies one connection on the server until it times out or is closed.

**Localhost (`127.0.0.1`)**  
The address that means “this machine.” The attack targets `127.0.0.1` so Slowloris hits nginx on the same VM. Netdata is opened the same way, at `127.0.0.1:19999`.

**SYN (synchronize)**  
A TCP flag set on the first packet of a connection attempt. It means “I want to open a connection.” A large number of SYNs with few completed handshakes can mean connection attempts that are not finishing.

**ACK (acknowledgment)**  
A TCP flag that confirms receipt of data or, during setup, that the handshake is proceeding. A completed handshake is SYN, then SYN-ACK, then ACK. Established connections in a capture are ones that reached this point and stayed open.

**RST (reset)**  
A TCP flag that aborts a connection immediately. A server with DoS mitigation often sends RST (or otherwise closes the socket) when a client is too slow or over a limit. A vulnerable server tends to leave those connections established instead of resetting them.

**Three-way handshake**  
The exchange that opens a TCP connection: the client sends SYN, the server replies SYN-ACK, the client replies ACK. After that, HTTP data can flow. Slowloris completes the handshake, then stalls on the HTTP request.

## Packet analysis

**Packet**  
A single unit of data on the network. A `.pcap` file is a recording of packets, including their timestamps, addresses, and TCP flags.

**pcap (packet capture)**  
A file that stores recorded network traffic. The stretch task compares two captures: one from a server with no DoS limits and one from a server whose nginx rules close slow connections. The difference shows up in how connections end (they stay open, or the server resets them), not in a huge difference in traffic volume.

**Wireshark**  
The program used to open and filter `.pcap` files. It shows each packet’s TCP flags (SYN, ACK, RST) and lets you follow whether connections stay open or get reset.

## Lab environment

**Virtual machine (VM)**  
The course Linux machine where nginx, Slowloris, and Netdata run. Commands in the lab are meant to be run on that VM, not on your own laptop.

**RDP (Remote Desktop Protocol)**  
The remote-desktop connection used to open the VM’s graphical desktop so Firefox can reach the Netdata dashboard.

**Firefox**  
The browser on the VM. The lab uses it to open Netdata at `http://127.0.0.1:19999` and to view the live graphs while the attack runs.

**`wget`**  
A command-line tool that downloads a file from a URL. The stretch task uses it to fetch `pcap_files.zip` onto the VM.

**`unzip`**  
A command that extracts a `.zip` archive. The packet captures have to be unzipped before Wireshark can open them.
