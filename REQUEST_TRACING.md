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

This configuration roughly means that haproxy should generate a request id with a given format, and put it into the `X-Request-ID` header on the request before proxing it to the next service.

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


How do you trace a route from "the edge" (ie, haproxy) all the way through to the link operations
that the request has created? Let's walk through the lifecycle of some example requests.

## Simple case: request only hits one service
Let's say that I make a `GET` request to `http://api.backstroke.co/v1/links` request. This request, like all others,
will be routed through haproxy, which will tag the request with a request id. This request id will
then be proxied along with the request that haproxy makes to the `server` service on behalf of the
user.

When the `server` service receives the request, it logs the id of the request prior to handling it.
This means each request that comes into haproxy can be tied to the subsequent request made to the
`server` service.

## More complicated case: queued links
Let's say that I make a `GET` request to `http://api.backstroke.co/_linkid`. This request adds a link operation to the queue to be handled by the worker. This request is given a request id at the haproxy level, and the `server` service logs it out. In addition, when the new operation is added the the queue, its request id is also noted as a key/value pair in the root on the operation by adding a a `fromRequest` key equal to the request id that spawned it. Also, when the worker outputs the operation's status into redis, it also writes the request id on it too.

For example, let's say we make a `GET` request to `http://api.backstroke.co/_linkid`, and it's assigned a request id of `AC120001:C5A6_AC120009:0050_5A229DF8_0004:0007` at haproxy. The connection is proxied to the `server`, where it logs the request (including the request id). Then, the `server` service adds a new link operation to the link operation queue, that looks like this:

```javascript
{
  "type": "MANUAL",
  "link": { ... },
  "user" { ... },
  "fromRequest": "AC120001:C5A6_AC120009:0050_5A229DF8_0004:0007" // The id of the request that spawned the operation.
}
```

Once the worker service pops an operation off the queue, it performs the work, like normal. On completion (and at specified intervals along the way) an object is written into redis to record the status of the operation. Here's an example of one of those objects (which can be fetched with a request to `api.backstroke.co/v1/operations/:operationId`):

```javascript
{
  "status": OK,
  "type": "MANUAL",
  "startedAt": "2017-12-06T11:32:46.755Z",
  "finishedAt": "2017-12-06T11:33:46.755Z",
  "handledBy": "..."
  "output": { ... },
  "link": { ... }, // Same as the value in the link operation queue
  "fromRequest": "AC120001:C5A6_AC120009:0050_5A229DF8_0004:0007", // Forward the request id
}
```

# Notes
- Remember that in development, just hitting the endpoints of the services (and not going through
  haproxy) won't generate request ids. Ie, `curl http://localhost:8000` won't generate a request id but
  `curl -H 'Host: api.backstroke.co' http://localhost` will generate a request id.
