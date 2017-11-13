# Redis
The server and all workers use a redis instance to communicate. The worker does not have database
access for a reason - making the worker as stateless as posssible makes the code in the worker much
more understandable.

## System requirements
- A queueing system for link operations.
- Store the status of a given link operation throughout its whole lifecycle (queueing, running,
  finished).
- Store the number of successful and failed link operations.
- Store the link that each operation is accociated with in such a way that looking up all operations
  that happened for a link is relatively efficient.

### Note: Link queueing system
I chose [rsmq](https://github.com/smrchy/rsmq). With hindsight, I'm not sure if it was the best idea
(it's opaque, though I've reverse engineered the library enough to interface with it from the redis cli)
but it works and hasn't caused me to tear my hair out yet.

## Enqueueing Operations
When a link operation is kicked off, either manually or automatically, it's added into the rsmq
queue with [`queue.sendMessage`](https://github.com/smrchy/rsmq#send-a-message). The payload looks
like this:

```javascript
{
  "id": "fgwhrlharhkilga", // Assigned by rsmq after a successful enqueue. Treated as a secret.
  "type": "MANUAL", // or "AUTOMATIC"
  "user": {
    "id": 123,
    "username": "my-user",
    "email": "user@example.com",
    "githubId": "user id within github",
    "accessToken": "github access token",
    "createdAt": "2017-11-10T23:50:00.236Z",
    "lastLoggedInAt": "2017-11-12T23:50:00.236Z"
  },
  "link": {
    "id": 456,
    "name": "My link name",
    "enabled": true,
    "webhookId": "q34quhvrqvmhbmeqkvdhq", // This is the secret used to enqueue a manual link operation with `curl https://api.backstroke.co/_q34quhvrqvmhbmeqkvdhq`
    "lastSyncedAt": "2017-11-12T23:50:00.236Z",

    // Upstream Information
    "upstreamType": "repo",
    "upstreamOwner": "user",
    "upstreamRepo": "repo",
    "upstreamIsFork": false,
    "upstreamBranch": "master",
    "upstreamBranches": ["master", "feature-foo"],
    "upstreamLastSHA": "812322fb64b7f04ef23a052eec8a56c50b05ba5f",

    // Fork Information
    "forkType": "repo",
    "forkOwner": "user",
    "forkRepo": "repo",
    "forkBranch": "master",
    "forkBranches": ["master", "feature-foo"]
  }
}
```

Also, after being added to the queue, an "id" is added to the payload that is assigned by rsmq. This
is used to track the opration throughout the process and henceforth will be refered to as the
operation id.

Also, it's important to mention that link statuses expire after 24 hours due to disk space concerns.
This number was chosen somewhat arbitrarily but has proven to work in production.

## Reporting the status of operations
Once an operation is eaten off the end of the queue by a worker, the worker starts processing the
link operation. Before starting work on the link operation, the worker sets a key in redis called
`webhook:status:<OPERATIONID>` to a JSON encoded body that represents the status of the operation
thusfar:

```javascript
{
  "status": "RUNNING", // or "SUCCESS" or "ERROR"
  "startedAt": "2017-11-12T23:50:00.236Z",
  "finishedAt": null, // On completion, this is set
  "output": {
    // The response from the link operation.
    // Set on completion
  },
  "link": {
    // The "link" key in the link operation payload above.
    // Set on completion
  }
}
```

So, this means that at any point, given the operation id, the server can query redis for the status
of the webhook operation. If the server can't find the key, then either the operation id is invalid
or the operaton is still in the queue. In fact, this is how the dashboard's manual `Resync` button
knows that a link operation is still queing (the first case can be eliminated because we just
created the link operation, so we know it exists).

The server's `/v1/operations/:operationId` route can return the status for any link operation (ie,
it does the `webhook:status:<OPERATIONID>` lookup).

## Storing metrics
It's great to know how many operations have succeeded or failed over a given period of time. To this
end, when a link operation completes successfully, it increments the `webhook:stats:successes` key,
and when a link operation fails, it increments the `webhook:stats:errors` key. This helps me
determine times when there are higher than average error rates, and try to get to the root cause of
why they happen.

## Attaching operations back to links
Once a link operation has been enqueued, it's impossible to recreate a list of all link operations
that were made for a given link without traversing through all link operations in redis and looking
at their `link` key. This is really inefficient.

So, a tradeoff was made to use more disk space in exchange for an increase in speed / efficiency.

When a link is created, it's "attached" to a link in redis. What this looks like in practice is a
bunch of ordered sets called `webhook:operations:<LINKID>` where each item in the set is an
operation id for the given link, ordered by timestamp. An ordered set is used so that items older
than 24 hours can be removed (remember, link statuses only exist for 24 hours at maximum). Ideally,
I'd want a set that would allow a custom TTL on each key, but the redis maintainers aren't a big fan
of those since they add a lot of complexity.

When the server wants to fetch all operations related to a link, it just needs to return the
contents of the `webhook:operations:<LINKID>` set, and then optionally fetch the status for each
operation id by fetching the `webhook:status:<OPERATIONID>` key for each link in the operation.

This still isn't _fast_, but it's faster than any of the alternatives I could think up.
