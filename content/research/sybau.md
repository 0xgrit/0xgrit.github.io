+++
title = 'SYBAU: You Are Not Tracking Anyone With an IP Address'
date = 2025-12-18
description = "Why an IP address is not a tracking mechanism, not an identity, and not the stupid flex skids think it is."
tags = ["networking"]
+++

# Kinda random article, why?
I keep hearing a ton of misinformation about how you can supposedly use an IP address to track a person. I’ve also been a bit rusty on networking fundamentals, so this is a good excuse to refresh them.

Learning by teaching.

# What in the world is an IP address?
An IP address is a **logical addressing system** used to identify a network interface so data can be routed across networks.

“Logical” means it is software-defined. It is not tied to a physical location, a device owner, or an identity. IP addresses can be assigned dynamically or statically, reassigned, rotated, or shared between many users.

An IP address exists so packets know **where to go**, not **who you are**.

## Wait, what does “logical addressing” even mean?
It means the identifier exists purely at the network layer. Unlike MAC addresses or physical wiring, IP addresses are abstract and flexible.

Your IP can change:
- When you reconnect to a network
- When your ISP rotates addresses
- When you move between networks
- When traffic is routed through NAT or CGNAT

There is nothing inherently personal about it.

## How is an IP address useful in a real-world scenario?
Let’s say you visit `youtube.com` in your browser.

1. Your computer sends a request to a DNS server asking for the IP address associated with `youtube.com`.
2. The DNS server responds with one or more IP addresses.
3. Your computer sends an HTTP request to that IP address.
4. Routers across the internet forward your packets hop-by-hop based on the destination IP address. This process is called **routing**.
5. The YouTube server processes the request and sends the response back to your **public IP address**.

Without IP addresses, the internet literally could not function. There would be no way for responses to find their way back to the requester.

## Now you see
IP addresses are not about identity or tracking. They are about **delivery**.

They exist so packets can reach the correct endpoint and return a response.

# But can you still track location from an IP address?
Yes — but not in the way people usually imagine.

An IP address does **not** give you:
- Someone’s home address
- Their exact location
- Their identity

At best, it provides a **general location estimate**, and even that is often wrong.

# So how does IP-based location actually work?
IP lookup tools query databases that map IP address blocks to:
- Internet Service Providers
- Registered regions or cities
- Network infrastructure locations

This mapping is based on **where the ISP’s network operates**, not where the user physically is.

In practice:
- Mobile users often appear in the same city nationwide
- CGNAT users share a single public IP
- VPN users appear wherever the exit node is
- Two users in different provinces can resolve to the same city

You are seeing the location of the **network endpoint**, not the person.

# Can someone hack you just by knowing your IP address?
An IP address alone is not an exploit.

You are only at risk if:
- Your IP is publicly reachable
- You are exposing a service
- That service has a vulnerability

For example, hosting an unpatched Windows 7 server directly on the internet is a bad idea. But the average user is not doing this.

Most people:
- Are behind NAT or carrier-grade NAT (CGNAT)
- Do not expose services to the internet
- Are protected by basic firewalling

Knowing your IP address does not magically grant access.

# What about people threatening to DDoS my IP address?
These threats are almost always meaningless.

If you are behind CGNAT:
- You share a public IP with many users
- Your ISP controls traffic filtering
- Your ISP absorbs or drops malicious traffic upstream

Even without CGNAT, ISPs already implement rate limiting, filtering, and mitigation. Random people “threatening” DDoS attacks are usually just repeating words they don’t understand.

# Final thoughts
An IP address is:
- Not an identity
- Not a tracking mechanism
- Not a hacking shortcut

It is a routing label used by the internet to move packets from point A to point B.

If someone claims they can “track you” with just an IP address, what they actually mean is:
> “I don’t understand networking, but I want to sound intimidating.”

SYBAU.
```
