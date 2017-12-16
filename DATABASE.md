# One Database, many services

Often when a microservices architecture is assembled, each service has its own state (which
manifests itself as a database for each service). I've worked on a couple different projects where
this architecture has prevailed and it's made tracing the effect of a given request on a system
really difficult.

In Backstroke, the database is treated like any other service. As few services are given database
access as possible, and unidirectional data flow is enforced whenever possible to minimize the
bugs related to the system's shared mutable state.

Currently, only two bits of code have database access - the `server` service and the
`operation-dispatcher` job.

The `server` service handles all of the CRUD operations for links and link operations. CRUD
intrinsically requires some sort of persistence layer in order to store the state of the resource
being acted upon. Most importantly, this is the only service that writes to the database.

The `operation-dispatcher` job handles automatic link updates. Once every ten minutes, it checks to
make sure that a link is up to date, and if it isn't, it will dispatch a new link operation into the
link queue.

## What about redis?
The Backstroke system does have one common bit of shared state: they all communicate with each other
using Redis. However, two important characteristics have been carefully designed around to minimize
any problem that this shared state would have:

1. Redis is used as a temporary cache to hold all items in the link operation queue, and to store
the response of link operations. These are both ephemeral values and expire after 24 hours. If redis
were cleared and all services restarted, the system would loose no permanant data.

2. Outside of the queue, Redis is used as an append-only data source where everything expires. This
was purposely chosen to attempt to eliminate potential sources for race conditions and other
nasty, hard to detect bugs. The queue is managed by rsmq, an external queueing system, which helps
to make this data storage mechanism safer.

# Database Schema
(coming soon)
