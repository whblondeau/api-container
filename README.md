# api-container
High-confidence generic container for multidestination data storage APIs.

## basic concept
This is a general container for discrete API data storage components. It supports HTTP request/response.

The container provides guaranteed-delivery semantics via:
 - local filesystem persistence of mutation requests
 - queue-based delegation to stash components for separate update destinations
 - transactional rollback of partial failures to prevent inconsistency

The design includes:
 - declarative configurability for read and update components
 - queue implementation
 
## operational description
This is a nested set of operations. In every case, a failure will result in:
 - logging of failure
 - forwarding of request to dedicated Fail Queue
 - rollback of successful updates completed for this request
 - return of failure response (HTTP 4xx or 5xx)
 
A success will result in:
 - return of success response (2xx)
 - deletion of local filesystem copy
 


### operational sequence
* **Save incoming request to local filesystem.** This returns a filename, which is used as a read-only resource for subsequent queuing operations.
  * **Enqueue request.** The request, as noted, is represented by a filename. This is enqueued in a _front queue_. The front queue returns an integer _unit of work identifier_.
    * **Fanout.** The UOW 2-tuple (UOW ID, filename) is read from the head of the front queue by a fanout component. The fanout component enqueues the UOW to all configured storage routing queues.
      * **Destination stash.** Each routing queue is read by a stash component. The stash component may:
        - _Decline_, which is a success (e.g. if the stash component is working to save an image file and there is no such attachment, the image stash component would say, "Nope," and return a 2xx message.
        - _Successfully store_ the mutation and return a 200 message, while preparing a rollback keyed to the UOW.
        - _Fail_ to store the mutation and return a 4xx or 5xx message.

Note that "returning a message" is done via callbacks. The stash components call back to the fanout component, which in turn resolves either success or failure for all.
 - In the event of any failure, the fanout issues an idempotent rollback command to the rest. The entire report is logged, and 
   - for a 5xx fail, the failure is forwarded to the fail queue.
 - In the event of either success or failure, the fanout hits the API itself.
   - For success, the API returns a 200 and deletes the original local filesystem record of the request..
   - For 5xx failure, the API returns a 5xx. The local filesystem record remains in place, and the UOW is left in the fail queue for possible retry.
   
This is the key distinction: a 4xx error is logged but not queued for retry. A 5xx error is also logged, but it goes to the fail queue. The fail queue permits retry.
