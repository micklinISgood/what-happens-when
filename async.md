REST and long-running jobs
16 Oct, 2014

This article describes a proper way to handle (asynchronously) a long-running job creation in the RESTful API.

Context
Consider a situation when you need to create a resource and the operation takes long time to complete.

Actually, this scenario is not that uncommon: after all, REST is not about manipulation of a couple database rows in some CRUD scenario. REST is about manipulation of arbitrary resources, and a resource might require extensive computation in order to come to existence.

So, you basically have two options:

you will force API client to wait until the resource is actually created
you can immediately return some status response, and defer creation to some later point
Example
Let’s create something non-trivial (I use HTTPie tool):

± http POST https://api.service.io/stars name='Death Star'
HTTP/1.1 201 Created
Location: /stars/12345
Here, we are trying to create a Death Star, and as you see it is created and Location is returned back.

Now, as you might imagine star creation is not an easy job by itself, let alone when we want something as equipped as the Death Star. Which means we will have to wait for a loooong time before we see that 201 Created response.

So, our aim is to initialize star creation, get acknowledgement message back, and have some fun around, up until the resource will be finally ready (we are ok if we have to poll from time to time to check for the status updates).

Why waiting is not cool
Well, because!

More seriously, there’s nothing wrong with forcing API client to wait, per se. If on server side you rely on some asynchronous loop, and can handle crazy number of connections, and if the eventual wait period is some n seconds (where n is acceptable for your purposes), then you can definitely stop reading this article and not discover the method cool kids have all been using for several years now.

Asynchronous processing (done wrong)
Your first instinct might be: “What if I return HTTP 201 Created immediately, but defer the actual creation to some later point?”.

Well, you can’t do that. If you do, you will be violating the HTTP/1.1: Semantics and Content protocol (more exactly the Section 6.3.2 of RFC 7231):

Section 6.3.2 of RFC 7231
The 201 (Created) status code indicates that the request has been fulfilled and has resulted in one or more new resources being created. The primary resource created by the request is identified by either a Location header field in the response or, if no Location field is received, by the effective request URI.
That’s HTTP 201 Created must be used when resource is actually created, not queued for creation.

Asynchronous processing (done right)
Let’s suppose we have some queue where we can put long-running jobs (to be periodically executed by some worker process).

Now, instead of 201 Created we can return 202 Accepted.

Section 6.3.3 of RFC 7231
The 202 (Accepted) status code indicates that the request has been accepted for processing, but the processing has not been completed.
As you see, that’s exactly what we are after!

We know what status code to return, what about Location header? Simple. Instead of the location to the actual resource, API will return location of the queued task that got created:

± http POST https://api.service.io/stars name='Death Star'
HTTP/1.1 202 Accepted
Location: /queue/12345
Pro Tip: It is allowed to return a payload along with 202 Accepted response, and you should use this opportunity to return something meaningful (like ETA for task completion, current state etc).

Implementation Notes
There are several implementation related questions that are frequently asked when it comes to deferred processing. Let’s review them.

Q1. How to learn when resource is finally available?

You need to query the queued task (that’s why API returned the Location header):

± http GET https://api.service.io/queue/12345
HTTP/1.1 200 Ok

<response>
    <status>PENDING</status>
    <eta>2 mins.</eta>
    <link rel="cancel" method="delete" href="/queue/12345" />
</response>
Pro Tip: Following HATEOAS constraint, we can add link to a state that will allow to cancel/delete the queued task.

Q2. What happens when resource is created, how does the queue task resource change?

Once resource is created, API should respond with 303 See Other status code on all the subsequent requests to the queued task:

± http GET https://api.service.io/queue/12345
HTTP/1.1 303 See Other
Location: /stars/97865
Q3. What to do with the task resource, when creation is completed?

While resource is being created its corresponding task is available, and you can query it for the status.

Once the originally desired resource is created, there are two alternative ways to deal with the temporary task resource:

API client must issue DELETE request, so that server purges it. Until then, server responds with 303 See Other status. Once deleted, 404 Not Found will be returned for subsequent GET /queue/12345 requests.
or, garbage collection can be a server’s job to do: once task is complete server can safely remove it and respond with the 410 Gone on subsequent GET /queue/12345 requests.
Pro Tip: Server can assign some expiry dates to all new queued tasks, and expire them regardless the completion status. That way server limits for how long (maximum) any given task can run. Such a strategy is not against the 202 Accepted declared purpose, and it is allowed for a resource to never come to existence.

References
Thijssen, J. Asynchronous operations in REST
Thijssen, J. REST Cookbook
