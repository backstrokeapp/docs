# The operation dispatcher: How automatic link updates work

In Backstroke's system, links between repositories are kept up to date automatically. How does that work? The `operation-dispatcher` job, found [here](https://github.com/backstrokeapp/jobs/tree/master/operation-dispatcher), does the majority of that work.

In particular, the `operation-dispatcher` service is trying to answer the question: Has the upstream repository in a link changed from the last time I checked it?

## How it works
Every 10 minutes, the operation dispatcher job runs each link through the below process:
  - Given the upstream owner, repository, and branch, fetch the sha1 hash of the most recent commit on that branch.
  - Compare the fetched hash against a stored hash.
    - If the stored hash is `null`, then the link is brand new and this is the first syncing operation that has been run against it. **Therefore, add a new `AUTOMATIC` link operation to the queue.**
    - If it's the same as what's stored, we know that no new commit has been added. The process stops.
    - If it's different, then the upstream repository has had a new commit. **Therefore, add a new `AUTOMATIC` link operation to the queue.**
  -


## How is the SHA hash fetched for the upstream repository?
Within the job, there's a function called [`fetchSHAForUpstreamBranch`](https://github.com/backstrokeapp/jobs/blob/master/operation-dispatcher/fetch-sha-for-upstream-branch.js).
