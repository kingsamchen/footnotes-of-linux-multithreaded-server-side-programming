# Accelerated Connection Retry for HTTP and Firefox

Not all packet loss is created equal. In particular, losing a `SYN` can really ruin your day - or at least the next 3 seconds which can feel like all day. Most Operating Systems take 3 seconds of waiting before retrying the `SYN`. Most other timeouts are dynamically scaled to the network conditions, but not the `SYN`. It is generally hardcoded. And on most of today's networks 3 seconds is an eternity.

So, in FF we took a page from Chrome's book and said if Firefox has been waiting for 250ms (configurable via network.http.connection-retry-timeout) then start a second connection in parallel with that first one. Assuming you've got .25% packet loss and general independence between packets spaced that far apart the approach turns the 3 second pause from a 1 in 400 event into a 1 in 16,000 event. That is pretty much the difference between "kinda annoying" and "didn't notice". It's a good idea - if you hate it for whatever reason just set the pref to 0 to disable it.

Taking the idea one step further, if we create two connections because of this timer and they actually both end up completing obviously only one can be used immediately. But we cache the other one as if it were a persistent connection - and then when you need it (which you probably will) you don't have to wait for the handshake at all. It is essentially a prefetched TCP connection. On my desktop, I run with an especially low timer so that any site with a > 100ms RTT benefits from this and its great!

You can see this effect below, using mnot's cool [htracr](https://github.com/mnot/htracr), on the second connection. Note how there is no request placed on it as soon as it is live (the request is the red dot at the top of the grey rectangle - the rectangle represents the connection), but one follows shortly thereafter without having to do a handshake. That's an RTT saved!

---
1. 见书 P333
2. 原文链接 https://bitsup.blogspot.com/2010/12/accelerated-connection-retry-for-http.html
