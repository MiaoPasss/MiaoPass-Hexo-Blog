---
title: 网络编程与套接字(Socket)
date: 2025-02-13 09:35:34
categories: 网络模型
tags: 
- 网络编程
- 套接字
---

# Socket vs Port

- A TCP socket is an endpoint instance defined by an IP address and a port in the context of either a particular TCP connection or the listening state.
- A port is a virtualisation identifier defining a service endpoint (as distinct from a service instance endpoint aka session identifier).
- A TCP socket is not a connection, it is the endpoint of a specific connection.
- There can be concurrent connections to a service endpoint, because a connection is identified by both its local and remote endpoints, allowing traffic to be routed to a specific service instance.
- There can only be one listener socket for a given address/port combination.


Specifically, a TCP socket consists of five things:
1. transport layer protocol,
2. local address,
3. local port, 
4. remote address, 
5. remote port

A port is a number between 1 and 65535 inclusive that signifies a logical gate in a device. Every connection between a client and server requires a unique socket.

For example:
- *33123* is a port.
- *(localhost, 33123, 69.59.196.211, 80, TCP)* is a socket.

> Firefox (localhost:33123) <-----------> stackoverflow.com (69.59.196.211:80)
Chrome  (localhost:33124) <-----------> stackoverflow.com (69.59.196.211:80)

When a client device accesses a website (such as Chrome sending HTTP requests), it automatically connects to port 80 on the web server to retrieve the requested content. When the web server receives the answer, it sends a response that has 80 as source port and 33123 as destination port.

---

# Credits

网络编程与套接字：https://www.cnblogs.com/Sunbreaker/p/11318288.html

---