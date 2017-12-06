# Request Tracing

In distributed systems, it's often difficult to follow a single request through a system to troubleshoot an issue. Backstroke solves this issue by tagging each request with an identifier.

Haproxy has a section of configuration that will automatically assign an id to each request that is passed into it:
```
# Add request ids for tracking requests through the whole system
# clientip:port_listeningip:port_timestamp_counter:haproxypid
# AC120001:C5A6_AC120009:0050_5A229DF8_0004:0007
unique-id-format %{+X}o\ %ci:%cp_%fi:%fp_%Ts_%rt:%pid
unique-id-header X-Request-ID
```

Given that the request id can be whatever data we want, a format was chosen that contains a bunch of information pertenent to the request. The identifier could be random or sequential, but putting helpful information into the request id means that it has inherint meaning, and given some information about the request, an approximate request id can be calculated (which is handy when grepping through logs).

Here's the format, explained:
`AC120001:C5A6_AC120009:0050_5A229DF8_0004:0007`
- `AC120001` is the client ip address. You might ask - this isn't an ip address, it's hex! Because these logs are stored as ascii, it's much more efficient to encode the 32 bit number behind the ipv4 address into hex. When this value is converted into decimal, it's equivelent to `2886860801`, and when piped into an online converter such as [this](https://www.browserling.com/tools/dec-to-ip), its value is `172.18.0.1`. This is the ip address of the user that made the request!
- `C5A6` is the source port of the connection, also encoded in hex. (`50598`)
- `AC120009` is the listening ip address, which in this case, converts into `172.18.0.9`. This might seem irrelevant because we have only one edge node, but if Backstroke scales in the future its nice to have this already baked in to the request id.
- `0050` is the listening port of the connection, which is `80` (https is not handled by haproxy in this system)
- `5A229DF8` is the unix epoch (in seconds) that the request was initiated. In the above, the value is `1512218104` - or `Sat, 2 Dec 2017 12:35:04 +00:00`
- `0004` is a sequential counter of requests. This means that we can handle a max of `0xFFFF` requests per second (`65535`)
- `0007` is the pid of the haproxy instance that handled this request.


How do you trace a route from "the edge" (ie, haproxy) all the way through to the link operations that the reqest has created?
Coming soon!
