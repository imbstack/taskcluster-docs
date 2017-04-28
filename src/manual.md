---
filename: index.md
title: Taskcluster Manual
---

What does Taskcluster do?

Fundamentally, it executes *tasks*. A task is defined in a JSON object, and
placed on a *queue*. A *worker* later claims that task, executes it, and
updates the task with the results.

## The Taskcluster Service and Team

We provide Taskcluster as a service, rather than as an installable application.
The service is constructed as a collection of loosely-coupled microservices.
These microservices share many characteristics, making it easy to use them
together.

Taskcluster aims to be a general platform for task execution to support
software development within the Mozilla organization. It is very much a work
in progress. The current focus is to support Firefox development, without
losing the generality that will make it useful for other Mozilla projects.

Taskcluster is designed to integrate well with other services and components.
Some of those integrations are Mozilla-specific (for example, Treeherder),
while others are not. Some integrations are maintained and provided by the
Taskcluster team, such as AWS compute resources. Others are managed separately,
and may not be available to all users -- including the compute resources
assigned to Firefox development.

## Working with Taskcluster

Most people do not need to understand everything about Taskcluster! The
[Taskcluster Tutorial](/tutorial) is designed to lead you to the information
that is most relevant to you.

The [reference section](/reference) contains documentation about the
Taskcluster services and libraries. Once you have determined the services you
need to interface with, this section can provide all of the technical detail
you need to do so successfully.  The reference section begins with a [guide to
the microservices](/reference/guide) that can help to determine the services
most relevant to your work.

The remainder of the manual describes the Taskcluster platform in depth.  The
[Tasks chapter](/manual/tasks) describes the concept around which the system
revolves -- tasks.  The [Task Execution chapter](/manual/task-execution)
describes how these tasks are executed.  The next chapter, [System
Design](/manual/system-design), provides the details you will need to interact
with Taskcluster. Finally, [Using Taskcluster](/manual/using/) presents a
collection of common use cases and approaches to satisfy them.

---
filename: tasks/index.md
title: Tasks
order: 10
---

This chapter focuses on Taskcluster's core concept: tasks.

At the highest level, Taskcluster executes tasks. New tasks are added to one of
several queues, and workers consume tasks from those queues and execute them.
In the process, these services send [Pulse](/reference/design/apis/pulse) messages
about task status, and attach the results of the task -- including logs and
output artifacts -- to the task itself.

A task is described by a task definition, which is a simple data structure
usually encoded in JSON or YAML.  Each task has a unique `taskId`, assigned
randomly by the caller when the task is created.  The full schema for the task
definition is given in [XXX](XXX TODO schema link), but a simple task might
look like this:

```yaml
provisionerId:      aws-provisioner-v1
workerType:         tutorial
created:            2020-01-04T04:18.084Z
deadline:           2020-01-05T04:18.084Z
metadata:
  name:             Example Task
  description:      Eample from TC Manual
  owner:            nobody@taskcluster.net
  source:           https://github.com/taskcluster/Taskcluster-docs
payload:
  image:            ubuntu:16.04
  command:          ['echo', 'hello world']
```

You can use the [task creator](https://tools.taskcluster.net/task-creator) to
experiment with creating tasks.

The next few sections describe the many powerful features of tasks, while the
[next chapter](/manual/task-execution) covers their execution by workers.

---
filename: tasks/times.md
title: Times
order: 10
---

Each task has three associated times:
* `created` is the time the task was created;
* `deadline` is the time by which the task must be completed; and
* `expires` is the time after which all record of the task and its runs will be deleted.

The deadline exists to ensure that tasks do not remain pending forever. It is
usually set to one day after `created`, on the assumption that running the task
more than a day later is not useful.

Task expiration ensures that tasks are not stored indefinitely, thereby
controlling storage costs. Expiration is often set to one year after creation,
but for common tasks, tasks which produce large artifacts, or tasks which are
unlikely to be useful later, a much shorter expiration is appropriate.

---
filename: tasks/workertypes.md
title: Worker Types
order: 20
---

Workers execute tasks, but there are many types of workers. A task specifies a
single "worker type" by which it should be executed, using the two-part
namespace `<provisionerId>/<workerType>`.  Workers with the same worker type
all consume tasks from a single queue, as [described
later](/manual/task-execution/queues), and as such are interchangeable and have
identical configurations.

The format of a task's `payload` property is specific to the worker that will
execute it, so defining a task requires knowledge of worker type's
configuration. If given a task with an inappropriate payload, a worker will
resolve the task with the reason `malformed-payload`.

---
filename: tasks/runs.md
title: Runs
order: 30
---

A task definition is, in a sense, the input to task execution. The output,
then, comes in the form of runs and associated artifacts.

A run is an attempt to execute a task. Most tasks have a single run, but in
cases where transient errors such loss of a compute node cause a run to fail,
automatic re-runs will add additional runs to the execution.

Runs are identified by their `runId`, a small integer. While for most tasks,
the single `runId` is `0`, you should always use the latest run to avoid
reading information about a run that failed.

Each run has properties describing the worker that claimed the run (referenced
by the two-part namespace `<workerGroup>/<workerId>`), the reason the run was
created, its current status, and various timestamps.

A run is in one of a few states:

 * `pending` - ready to be claimed and executed
 * `running` - currently executing
 * `completed` - finished successfully
 * `failed` - finished unsuccessfully
 * `exception` - finished due to some reason internal to Taskcluster

Exceptions for infrastructural reasons (for example, loss of a worker) will
result in an automatic re-run if there are sufficient retries left for the
task.

A task's state is based on the state of its latest run, with the addition of an
`unscheduled` state for a task with no runs (as is the case when the task's
dependencies are not complete).

---
filename: tasks/artifacts.md
title: Artifacts
order: 35
---

But by far the most commonly-used data from a run are its artifacts. These are
HTTP entities containing output data from the task execution.

Unlike most API methods which return a JSON body, requesting an artifact from
the Queue service returns the artifact itself, possibly via one or more HTTP
redirects. This means that -- at least for public artifacts which require no
authentication -- any sufficiently robust HTTP client can download an artifact
directly.

**NOTE**: Not all clients are "sufficiently robust"! The artifact interface
makes heavy use of redirects, and artifacts may make use of other web-standard
features such as content encoding.  Like any distributed system, requests may
fail, too, and a robust client should retry. Out of the box, `curl` and `wget`
do not handle most of these cases.

Taskcluster's Queue service supports a number of artifact types, including
several storage data-storage back-ends as well as two special types: errors and
references. Error artifacts will always return an HTTP 403 (Forbidden), with
message and details supplied by the task. Reference artifacts return a 303 (See
Other) redirecting the client to another URL.

## Public Artifacts

While it is possible to create artifacts which require authorization to
download, most artifacts are public. These are easily identified by the prefix
`public/` in the artifact name.

## Log Artifacts

By convention, workers record the output of a task -- a command's output to
stdout and stderr, for example -- in an artifact named `public/logs/live.log`.  The
details of this artifact, such as its storage type and encoding, can differ
from worker to worker. You should not assume anything about it, as the
implementation may change dynamically to optimize performance.

While a task is running, some workers can relay data to the client as it is
produced, in a single HTTP request. Clients can parse and display this data as
it arrives to present a "live log" to the user.

---
filename: tasks/dependencies.md
title: Dependencies and Task Graphs
order: 40
---

Tasks can be linked together such that a dependent task does not start until
its prerequisites are complete. This functionality enables the creation of
"task graphs" containing multi-task dependency chains. For example, a task
graph to build and test an application might start with a set of library build
tasks (one for each platform), followed by application build tasks depending on
those library builds, and culminating in test tasks that depend on the
application build tasks.

Dependencies are defined by listing prerequisite taskIds in the `dependencies`
property of each task. The prerequisites must already be defined, so graphs
must be created in order. The `requires` task property controls whether the
dependent task should execute if any of its prerequisites fail.

A task can depend on itself -- this is called a "self-dependency". Such a task
will not run automatically, but can be force-scheduled as described in
[Manipulating Tasks](manipulating).

---
filename: tasks/taskgroupid-schedulerid.md
title: The taskGroupId and schedulerId properties
order: 45
---

Each task is assigned to a task group by its `taskGroupId`. Task groups
associate "related" tasks, such as those created in response to the same
version-control push. The Queue service provides an API method to list all
tasks with a specific `taskGroupId`, and task manipulation can be limited by
`taskGroupId`.

Tasks also have a `schedulerId`. All tasks in a task group must have the same
`schedulerId`. This is used for several purposes:

 * it can represent the entity that created the task;

 * it can limit addition of new tasks to a task group: the caller of
   `createTask` must have a scope related to the `schedulerId` of the task
   group (more about scopes later); and

 * it controls who can manipulate tasks, again by requiring
   `schedulerId`-related scopes.

---
filename: tasks/manipulating.md
title: Manipulating Tasks
order: 50
---

For the most part, once a task is created it is left to run to its final
resolution. However, there are a few API methods available to modify its
behavior after creation.

Permission for these methods is based on the task's `schedulerId`.

## Force Scheduling

A task which has unfinished dependencies is considered "scheduled", but not yet
"pending". A scheduled task that depends on itself will remain in that state
until its deadline, unless it is *force-scheduled* with the Queue service's
`scheduleTask` method.

## Cancelling

A task that is not yet complete can be cancelled. This is generally a
best-effort operation, useful for saving resources. Task execution may continue
for some time after an active task is cancelled, until the worker performing
the task attempts to reclaim it.

---
filename: tasks/scopes.md
title: Task Scopes
order: 60
---

Taskcluster permissions are represented by "scopes", covered in more detail in
[the design section](/manual/design/apis/hawk/scopes). For this section, it
is enough to know that a scope is a permission to perform a particular action.

A task has a set of scopes that determine what the task can do. For example, a
worker may limit access to features based on scopes. It is also possible for a task
to make arbitrary API calls using its scopes via an authenticating proxy.

Taskcluster avoids "scope escalation", where a user might gain access to scopes
they should not have, by requiring that the creator of a task have all of the
scopes associated with the task. Thus a task's scopes are a subset of the scopes
its creator has.

Creating a task also requires the caller to have scopes for various parts of
the task definition. The details are in the Queue's reference documentation,
but this includes routes, priorities, and `schedulerId`.

---
filename: tasks/messages.md
title: Messages
order: 70
---

Taskcluster publishes pulse messages when tasks change state.
[Pulse](/reference/design/apis/pulse) is a message bus with a publish / subscribe
model.  These messages are sent by default with a route containing several of
the task's ID's, but can also be sent to arbitrary routes listed in the task's
`routes` property.

This provides a powerful extension point for integrations with Taskcluster.
Taskcluster itself includes a few such integrations:

* The Index service listens for messages about task completion on routes
  beginning with `index`, storing the relevant taskIds for later retrieval.
  [More information..](/reference/core/index)

* The Notify service listens for messages beginning with `notify` and
  translates them into user notifications via email or irc.  [More
  information..](/reference/core/notify)

---
filename: task-execution/index.md
title: Task Execution
order: 20
---

We've seen how tasks are defined. Now we turn to the process of executing those
tasks.

---
filename: task-execution/queues.md
title: Queues
order: 10
---

The [queue service](/reference/platform/queue), hosted at
**`queue.taskcluster.net`**, is the centralized coordinator that is responsible
for accepting tasks, managing their state, and assigning them to workers that
claim them.

The queue service maintains the task queues. Task queues are named, with the
names having the form `<provisionerId>/<workerType>` (more on provisioners in a
later chapter). Tasks are handled in FIFO order (except for tasks of different
priority), so that tasks added earliest will be executed first.

A dependent task is not added to the queue until all of the tasks it depends on
have completed. Task deadlines ensure that no task remains in enqueued forever:
the task is deleted when the deadline has passed.

When resources permit, we prefer to have empty queues by executing all tasks
when they are submitted to the queue. Resources, of course, do not always
permit.

The Taskcluster Queue does _not_ require any configuration or programmatic
changes in order to start supporting a new worker. A task defines which
`provisionerId` and `workerType` it requires, as arbitrary string fields. So
long as a worker successfully authenticates requests to the Queue, and makes
claims against the given `provisionerId` and `workerType` combination, the
Queue will cooperate by providing Task information, and assigning tasks to the
worker. This means no Queue downtime for the roll out of a new worker. It also
means one-off workers can be written for one-off jobs, if required.

---
filename: task-execution/workers.md
title: Workers
order: 20
---

Workers are the component responsible for actually executing tasks. Taskcluster
operates on the "pull" model, where workers pull tasks from queues as they have
available capacity. The queue they pull from is named after the worker's type.

Typically workers are implemented as daemons running on dedicated instances in
the cloud. As such, they have no REST API -- that is, there is no "worker
service".

## Worker Groups and Ids

Each individual worker is identified by a `<workerGroup>/<workerId>` pair.
This information can be useful to identify errant workers.

## Claiming Tasks

Workers signal their availability for new work by calling the Queue's
`claimWork` method. If there is a task available, this method returns its
details and records a "claim" of that task by the calling worker. The claim
lasts for a short time - a matter of minutes - and must be renewed
(`reclaimTask`) before it expires. If a claim does expire, the task run is
marked as `exception` and the task may be automatically retried.

In most cases, multiple workers pull work from the same queue, so the worker
that executes a particular task is not known in advance.  In practice, this
does not matter, as all workers with the same `<provisionerId>/<workerType>`
are typically configured identically.

As mentioned previously, the format of the task payload depends on the worker
executing the task. It is up to the task author to know the payload format a
particular worker, as identified by its worker type, will expect.

## Bring Your Own Worker

Workers communicate with the Queue API using normal Taskcluster API calling
conventions. It is quite practical to run purpose-specific workers with a
configuration appropriate to that purpose.

Those workers may run one of the worker implementations developed by the
Taskcluster team, or an entirely purpose-specific implementation that uses the
same APIs. For example, the [scriptworker](XXX) was developed by release
engineering to run release-related scripts in a limited, secure context.

The Taskcluster-provided worker implementations, including their payload
formats, are described in the [workers reference](/reference/workers).

## Worker Scopes

The Queue APIs used by workers require Taskcluster credentials specific to the
worker type and ID. This prevents a worker from claiming tasks from other
worker types or from misrepresenting its identity.

When a worker claims a task, it gets a set of temporary credentials based on
the task's `scopes` property. Those credentials can be used to call Taskcluster
services on behalf of the task.

---
filename: task-execution/worker-types.md
title: Worker Types
order: 30
---

Worker types can serve a few different purposes within Taskcluster.

Different worker types may have different system configurations. For example,
one worker type may execute tasks in a Windows environment, while another
executes tasks within an iOS application context. More subtly, two worker types
might execute the same "kind" of tasks, but in instances with varying resources
such as memory and CPU, and correspondingly varying costs.

Worker types also provide a security boundary. Taskcluster's access control
model limits what can create a task for each worker type. This can control
access to workers with special powers, such as one that can upload binaries for
release. It can also serve to isolate tasks with different trust levels from
one another: if level 1 tasks use a different worker type from level 2, then
the tasks will never run on the same instance. Any operating-system or worker
vulnerability that might let two tasks on the same host affect each other would
not allow cross-contamination between the two levels.

Worker types can provide cache boundaries. Many worker implementations cache
data for re-use on later tasks. Dividing tasks into worker types based on the
caches they use can increase the cache hit rate. For example, data analysis
tasks might benefit from being separated into one worker type per dataset, to
maximize the likelihood of a task using a cached dataset already present on the
worker.

Worker types can also control costs and capacity. If a worker type is
configured with a maximum resource allocation (for example, a provisioner is
configured to run at most ten instances), then regardless of the number of
tasks submitted, it cannot exceed that capacity. Yet identically configured
workers of a different worker type might have a substantially higher maximum
capacity.

---
filename: task-execution/provisioning.md
title: Provisioning
order: 40
---

In the simplest case, a worker implementation is installed and configured on a
host by hand and executes tasks indefinitely. For most purposes, it is far more
cost-effective to dynamically adjust the pool of available workers based on the
work to be done. This process is called provisioning.

A provisioner operates by monitoring the pool of running workers and the
corresponding queues. Based on the queue length and the configuration of the
provisioner, it may create new workers or signal to running workers that they
should stop. Of course, provisioners might take into account additional
information to make better predictions of future load. A provisioner's
configuration can balance performance (many instances to execute tasks in
parallel) against cost (from idle workers).

## Provisioner IDs

Worker types are scoped within a `provisionerId`, allowing each provisioner its
own "namespace" of managed workers.

## AWS Provisioner

The Taskcluster team provides a provisioner implementation, the AWS
Provisioner, which creates EC2 spot instances within the Taskcluster AWS
account.

It is identified with `aws-provisioner-v1` as the `provisionerId` in a task
definition. Its administrative interface is available at
https://tools.taskcluster.net/aws-provisioner/.  This interface allows
monitoring of current load, as well as management of worker types.

The worker types provided by the AWS provisioner are a mix: some are
Firefox-specific, while others are available for other Mozilla uses. Since the
service runs with the Taskcluster team's AWS credentials, all costs are borne
by the Taskcluster team.

If you are setting up a new worker type with a new AMI, it is your
responsibility to ensure that a worker starts on the AMI when the AMI boots
that this worker understand the task payload and shutdown when no more tasks
are available from the queue,

---
filename: design/index.md
title: System Design
order: 30
---

This chapter starts to dive deeper into Taskcluster's implementation. The
information here is useful for those interacting directly with Taskcluster
services, and for those taking advantage of some of the more sophisticated
features available to tasks, such as scopes or routes.

---
filename: design/apis/index.md
title: Microservice APIs
order: 10
---

Taskcluster uses a microservices architecture, with each service exposing a
small set of API endpoints. Those endpoints, and other details of the
services, are documented in the [reference component](/reference).
We have already mentioned several services, especially the Queue service.

This section describes in detail how to interact with these Microservices.

---
filename: design/apis/hawk/index.md
title: API Authentication and Authorization
order: 10
---

All Taskcluster components use [Hawk](https://github.com/hueniverse/hawk) over
SSL for authentication and authorize access based on a set of "scopes"
associated with each client. Credentials and scopes are managed by the
[authentication](/reference/platform/Taskcluster-auth/references/api) component.

The details are documented here, but note that you can use one of the fine
[Taskcluster clients](/manual/tools/clients) to handle all of these details for
you. They are available in a variety of languages, and are kept up-to-date
with the latest APIs.

---
filename; design/apis/hawk/authn.md
title: Authentication
order: 21
---

## Authentication (Who Are You?)

To access protected resources, a Taskcluster client must have credentials
consisting of a `clientId` and an `accessToken` (and, if using temporary
credentials, a `certificate`).

These credentials are used with the Hawk protocol to authenticate each HTTP
request. The `clientId` is passed as the `id` parameter, and `accessToken` is
passed as the `key` parameter.

If given, the certificate is passed as `ext.certificate` to the
`Hawk.client.header` method. In JavaScript:
```js
var header = Hawk.client.header('https://foo.taskcluster.net', 'GET', {
  credentials: {
    clientId: clientId,
    accessToken: accessToken,
  },
  ext: new Buffer(JSON.stringify({certificate: certificate})).toString('base64'),
});
```

**Remark**, it is possible to [restrict authorized scopes](authorized-scopes)
using the `ext` in combination with the certificate for a set of temporary
credentials. Just include both fields in the JSON object before base64 encoding
into the `ext` property.

Given this information, the Hawk protocol (as defined by its Javascript
implementation) signs the HTTP request, and the resulting token is placed in
the `Authorization` header.

---
filename: design/apis/hawk/scopes.md
title: Scopes
order: 22
---

## Authorization

All Taskcluster endpoints are guarded by [scopes](scopes). The authorization
information passed with each API call is used to determine a set of scopes
associated with the caller. These are compared against the scopes required by
the endpoint. If they are satisfied, the request is allowed to proceed.

The scopes required for each endpoint are documented in the reference section
of this manual. Note that some API endpoints may have additional scope
requirements that depend on the body of the request; read the endpoint
documentation carefully to discover these.

## Scopes

A scope is simply a string, limited to printable ASCII characters. Scopes
generally travel in sets. For example, a typical client will have a set of a
few dozen scopes.

## Satisfaction

A set of scopes A is said to "satisfy" another set of scopes B if every scope
in B is also in A. In a practical sense, A is often the set of scopes
associated with some Taskcluster credentials, and B is the set of scopes
required by an API call. If A satisfies B, then the call is permitted.

The more mathematically inclined may like to think of this as a subset
relationship: B ⊆ A.

There is one piece of special syntax in scopes: a final `*` character acts as a
wildcard, matching any suffix. So `queue:create-task:test-provisioner/*`
satisfies `queue:create-task:test-provisioner/worker3`. The reverse is not
true. The wildcard only works at the end of a string: no more advanced
pattern-matching functionality is available.

Even so, this is an incredibly powerful technique, as all of the scopes
required by Taskcluster services have been carefully designed with
more-specific attributes on to the right of less-specific attributes.

## But I Thought Scopes Were Crazy Complicated?

Nope, that's all there is to it!

Here are a few examples:

The scope set `["queue:create-task:aws-provisioner-v1/*",
"queue:route:index.project.persona.*"]` satisfies the scope set
`["queue:create-task:aws-provisioner-v1/persona-builder",
"queue:route:index.project.persona.build.20160101.linux64"]`.

The scope set `["secrets:get:garbage/*", "queue:create-task:*"]` satisfies the
scope set `["secrets:get:garbage/my/secret",
"secrets:get:garbage/your/secret"]`.

---
filename: design/apis/hawk/clients.md
title: Clients
order: 23
---

Taskcluster authentication begins with "clients". Each client has a name
(`clientId`) and a secret access token. These can be used together to make API
requests to Taskcluster services.

Clients can be configured to expire on a specific date. An expired client is
no longer recognized by Taskcluster services. Clients can also be disabled;
this is used to prevent use of clients for which an associated user no longer
has permission. Most users do not have permission to enable a client.

Every client also has a set of [scopes](scopes). The client's scopes
control the client's access to Taskcluster resources. The scopes are
*expanded* by substituting roles, as defined in the [roles](roles) section.

The set of defined clients is visible in the [Clients
tool](http://tools.taskcluster.net/auth/clients/). This interface helpfully
shows both the scopes configured for the client, and the "expanded scopes" that
result after all roles are expanded. Note that, in keeping with the open
nature of Taskcluster, anyone can see the full list of clients.

**NOTE** Taskcluster does not identify users. All API calls are made with
Taskcluster credentials, including a `clientId`, but that identifier does not
necessarily relate to a specific person or "user account" of any sort.

---
filename: design/apis/hawk/roles.md
title: Roles
order: 24
---

A _role_ consists of a `roleId`, a set of scopes and a description. Each role
constitutes a simple _expansion rule_ that says if you have the scope
`assume:<roleId>` you get the set of scopes associated with the role named
`roleId`. Roles can refer to other roles in the same way.


## Stars in Roles

As in scopes, a final `*` in a role ID acts as a wildcard. It matches any
`assume` scope of which it is a prefix. For example, the role ID
`repo:github.com/taskcluster/*` will match
`assume:repo:github.com/taskcluster/Taskcluster-auth`.

When roles are concerned, stars expand in two ways:

 * (scope expansion) An `assume` scope ending in a star will satisfy any scope
   implied by any role of which it is a prefix. For example, if role
   `repo:github.com/taskcluster/Taskcluster-auth` has scope
   `secrets:get:auth-tests`, then credentials with scope
   `assume:repo:github.com/taskcluster/*` can get the `auth-tests` secret.
   This means that `assume:` scopes ending in a star can be very powerful!

 * (role expansion) A role ending in a star will apply to all roles of which it
   is a prefix. For example, if role `hook:Taskcluster/*` has scope
   `queue:create-task:aws-provisioner/Taskcluster-hooks`, then a credential
   with `assume:hook:Taskcluster/nightly-diagnostics` can create a task with
   the `Taskcluster-hooks` worker type.

## In Practice

In practice, roles are used in a few ways within Taskcluster:

 * As a shorthand for a commonly-used set of scopes
 * As a means of associating scopes with external things such as source-code repositories or users
 * As a way to configure scopes for Taskcluster resources like hooks or worker types
 * As a scope allowing the bearer to "assume" the named role.

See the [namespaces](/manual/design/namespaces) document for more information.

The set of defined roles is visible in the [Roles
tool](http://tools.taskcluster.net/auth/roles/). This interface helpfully
shows both the scopes configured for the role, and the "expanded scopes" for
that role. The latter value can be a little misleading for `*`-suffixed
scopes, so be careful and if in doubt, create a throwaway client to test your
assumptions.


---
filename: design/apis/hawk/authorized-scopes.md
title: Authorized Scopes
order: 25
---

If you are making API requests on behalf of a less-trusted entity that you only
know to possess a subset of your [scopes](scopes), you can specify which of
the scopes you have that a given request is authorized to rely on. If the
request cannot be authorized with the restricted set of scopes you specified,
it will fail, even though you may in fact have the scopes required to conduct
the request. In effect, you can reduce the available scopes for each API
request.

**Example**, imagine that CRAN service would like to create Taskcluster tasks
whenever an R project is updated in the archive. However, different R packages
have different levels of trust and require different scopes. The tc-cran
service runs with the superset of all scopes that might be required (perhaps
`assume:project:cran:pkg:*`), and calls `queue.createTask` with
`authorizedScopes` set to `['assume:project:cran:pkg:paleotree']` for paleotree
tasks. The scopes available for creating that task are then limited to those
assigned to the paleotree package via its role.

Authorized scopes are specified in the Hawk `ext` property with the
`authorizedScopes` property. The Taskcluster client packages all contain
support for this functionality.

```js
{
  authorizedScopes:  ['scopeA', 'scopeC']
}
```

This technique is used in the task-graph scheduler to ensure that tasks created
are only created with the set of scopes that the task-graph has.

**Note** the way Hawk works, the `ext` property is covered by the HMAC
signature. So it's not possible to modify this property on-the-fly.

---
filename: design/apis/hawk/temporary-credentials.md
title: Temporary Credentials
order: 26
---

Any client with a `clientId` and an `accessToken` can issue temporary
credentials with a subset of its [scopes](scopes). Temporary credentials consist of
a `clientId`, `accessToken` and a certificate validating them.

Temporary credentials always have an explicit start and expiry date, which can
be up to 31 days apart. A set of temporary credentials cannot be used to issue
new temporary credentials, but they be used to generate signed URLs for
untrusted sources.

Because temporary credentials can be created anywhere, there is no central
registry of temporary credentials, and they cannot be listed (or revoked)
anywhere in the Taskcluster API.

## Certificates

A certificate looks like this:

```js
ext = {
  // ...
  certificate: {
    version:          1,
    issuer:           "issuing-client-id",
    scopes:           ['ScopeA', 'ScopeB'],
    start:            1410399435102,
    expiry:           1410399497349,
    seed:             'KpJvYUNXSYeWqc0vnsAq9wJJgvWv5pTh6IYhd120YZTQ',
    signature:        'Yw1ETAM+6PGA0T65IAEShwyDLDQqw7M8qpFzLpG+Nm8='
  }
}
```

The certificate is always issued by a client, called the `issuer`, in this case `issuing-client-id`.
The issuer's `accessToken` will be used to sign the temporary credentials.

The `clientId` used with the temporary credentials may be any id for which the
issuer has the scope `auth:create-client:<clientId>`.

In the certificate the `version` must always be `1` (at least for now).

The `scopes` property is a list of scopes the temporary credentials should have
access to, this must be satisfied by the issuer's scopes. In other words, the
temporary credentials cannot have any scope that the issuer does not have.

The `start` and `expiry` properties are the validity start and expiration of the
certificate, respectively. Both `start` and `expiry` are in milliseconds since
UNIX epoch, and these cannot be more than 31 days apart. Notice that it is
allowed to issue temporary credentials that take effect in the future, useful
for delegating access to scheduled tasks.

The `seed` property is a 44 character string of random characters, it used for
construction of the `accessToken`, as detailed later.

The `signature` property validates the authenticity of the certificate and is
generated as detailed below.

## Signature Generation

The signature for a certificate is generated by the issuer, using its access
token. The issuer does this by generating a base64 encoded SHA256 HMAC
signature of the following string:
```
version:1\n
clientId:<clientId>\n
issuer:<issuer>\n
seed:<seed>\n
start:<start>\n
expiry:<expiry>\n
scopes:\n
<scopes[0]>\n
<scopes[1]>\n
...
<scopes[n]>
```

Where `<clientId>` is the id that will be used with the temporary credentials.
As described above, this can be any id that the issuer is permitted to create.

In the certificate example from previous section, the string to sign would be:

```
version:1\n
clientId:temporary-cred-client-id\n
issuer:issuing-client-id\n
seed:KpJvYUNXSYeWqc0vnsAq9wJJgvWv5pTh6IYhd120YZTQ\n
start:1410399435102\n
expiry:1410399497349\n
scopes:\n
ScopeA\n
ScopeB
```

An authenticating server will validate the certificate on every request, using
the above signature algorithm.

## Construction of `accessToken`

The `accessToken` must also be generated by the issuer. The issuer does this
by generating a SHA256 HMAC signature of the `seed` from the certificate,
signed with the issuers `accessToken`.

The signature must be encoded using URL-safe base64 encoding without `=`
padding. See [RFC 4648, sec. 5](http://tools.ietf.org/html/rfc4648#section-5)
for details on URL-safe base64. Basically it amounts to replacing `+` with `-`
and replacing `/` with `_`, and then drop `=` padding.

The resulting HMAC signature is the `accessToken`, which should be given to
client that will be using the temporary credentials. The `accessToken` will be
reconstructed on the authenticating server for each request.

## Anonymous Temporary Credentials

For backward compatibility, Taskcluster supports "anonymous" temporary
credentials. For these credentials, the temporary credentials use the issuer's
`clientId`.

In this case, the certificate does not include an `issuer` property, and the
`clientId` and `issuer` lines are omitted from the string used to generate the
signature:

```
version:1\n
seed:<seed>\n
start:<start>\n
expiry:<expiry>\n
scopes:\n
<scopes[0]>\n
<scopes[1]>\n
...
<scopes[n]>
```

We recommend that you always use named temporary credentials, with a
descriptive `clientId`, to assist in debugging and logging.

---
filename: design/apis/hawk/signed-urls.md
title: Pre-signed URLs
order: 27
---

Hawk allows you to generate a _bewit_ signature for any `GET` request. Including
this _bewit_ signature in your request will then authenticate and authorize the
request. All Taskcluster APIs support authentication of `GET` requests using
these bewit signatures. And you'll find that the official
[Taskcluster-client](https://github.com/taskcluster/Taskcluster-client)
offers an API for generating these signatures.

Pre-signed URLs are most often used to provide signed URLs allowing another
application to download a private artifact without including Taskcluster
authentication headers.

---
filename: design/apis/hawk/troubleshooting.md
title: Troubleshooting
order: 30
---

In order to avoid disclosing information to a potential hacker, a number of
errors result in the rather terse "Bad Mac" error. Fundamentally, this means
that the credentials supplied were invalid for some reason. Errors regarding
scopes will use more informative error messages.

The most common error is forgetting to include the certificate in addition to
the clientId and accessToken.  There is no need to interpret the certificate
(please don't!); just treat it as an opaque string.  Omitting the certificate
when it is required will generally result in "Bad Mac" errors.

---
filename: design/apis/pulse.md
title: Pulse
order: 20
---

Pulse is a message bus: it allows participants to publish messages, and other
participants to subscribe to those messages. Taskcluster uses this bus to
communicate between microservices, notifying interested parties when specific
events have occurred. The system is based on AMQP in a publish-subscribe
configuration.

Each service's reference documentation describes the events the service sends
(if any). Each kind of event is sent on a different exchange, and each event
has a routing key containing characteristics of the event on which consumers
might like to filter.

Pulse is a publicly accessible service, and the pulse message schemas are a
part of Taskcluster's published API, so they make a nicely decoupled
integration point for external services.

---
filename: design/apis/errors.md
title: API Errors
order: 30
---

API clients should be prepared to accept HTTP errors in response to API calls.

## Retries

HTTP Errors in the 5xx series should generally be re-tried using an exponential
backoff. The provided API clients handle this for you.

## Resiliency

Clients should also be resilient to arbitrary response content: intermediate
proxies and load balancers may produce HTML error messages rather than the
expected valid JSON.

## API-Specific Errors

Errors returned from the API implementation have a JSON body with the following format:

```js
{
  "code": "TooManyFoos",
  "message": "You can only have 3 foos\n----\nmethod: toomanyfoos\nerrorCode: TooManyFoos\nstatusCode: 472 \ntime: 2017-01-22T21:20:16.650Z",
  "requestInfo":{
    "method": "toomanyfoos",
    "params": {}, 
    "payload": {"foos":[4, 5]},
    "time": "2017-01-22T21:20:16.650Z",
  },  
}
```

The `code` property identifies the error itself.  The codes that are common to all
services (defined in [Taskcluster-lib-api](https://github.com/taskcluster/Taskcluster-lib-api))
are described below, but specific services may add additional service-specific
error codes, documented in the reference section.

The `message` is suitable for display to users. It contains a detailed error
message and some basic information about the request.

The additional `requestInfo` fields describe the request that caused the error
in more detail, and may be useful for "advanced" users. The included `payload`
may have been modified to remove potentially sensitive information, and can be
safely shown to the user in an error message.

### MalformedPayload: 400

The method expects a payload, but the request body was not JSON

### InvalidRequestArguments: 400

The provided query strings or parameters are invalid

### InputValidationError: 400

The payload was valid JSON, but did not match the documented schema

### InputError: 400

The input was invalid for some other reason (described in the message)

### AuthenticationFailed: 401

Hawk authentication failed. The corresponding message is generally very terse,
to avoid providing extra information that might help an attacker guess valid
credentials.

### InsufficientScopes: 403

The hawk authentication succeeded (or there was no Hawk header), but the scopes
associated with that authentication were not sufficient for the requested
operation.

### ResourceNotFound: 404

The requested resource wasn't found

### RequestConflict: 409

The request conflicts with server state.  This can occur when a "create"
operation occurs for a resource that already exists.  Most "create" operations
are idempotent, meaning that two create calls for exactly the same resource
will both succeed, but if the two calls specify different payloads, a
RequestConflict error will result.

### ResourceExpired: 410

The resource existed, but has expired.

### InputTooLarge: 413

The provided payload is too big.

### InternalServerError: 500

An internal error occurred. The error message is not returned, lest it expose
security-sensitive issues, but is logged and available to Taskcluster
administrators.

---
filename: design/apis/reference-format.md
title: Reference Formats
order: 40
docson: true
---

Most Taskcluster services make heavy use of JSON schemas for validation of
incoming and outgoing data, whether through APIs or AMQP exchanges. This makes
the external API surface very reliable and consistent, and helps avoid a lot of
bugs and typos.

The use of JSON schemas also makes it very easy to **generate documentation**
for all the external interfaces offered by Taskcluster components, as done on
this site. To further simplify the generation of documentation and API-clients
we have formalized formats for describing interfaces.

This document describes the formats in which references for API end-points and
AMQP exchanges are documented. This is useful for **automatic generation** of:

 * Documentation
 * Client libraries
 * Dummy mock servers

**Note**, these formats are not completely stable and may change over time, or
merge into one.

## API References

Our API end-points all have a simple URL made up of a `baseUrl + route` where
a few argument have substituted into the `route`. The API end-point takes
JSON and input and returns JSON, input is always validate before it's accepted
and output is validated before it is returned.

This makes it easy to describe the API end-points in JSON with references to
JSON schema files. The reference format looks as follows:

<div data-render-schema="http://schemas.taskcluster.net/base/v1/api-reference.json">
</div>

The JSON schema for the API reference format is
`http://schemas.taskcluster.net/base/v1/api-reference.json` and references are
validated prior to publication.

## AMQP Exchange References

All of our AMQP exchanges are `topic` exchanges, and the messages are always in
JSON. Which makes it easy to validate all messages against a declared JSON
schema prior to publication. Note, we do **not** recommend validation of
messages upon consumption, as publishers may upgrade the schema in a
backwards compatible way in the future.

Usually, we prefix all exchanges from the same component with a common
`exchangePrefix` to ensure uniqueness. For routing keys we strive to always
have the same entries for messages on a given exchange and use `_` if no
value makes sense for the given routing key entry with a specific message.

These conventions makes it easy to describe all exchanges in JSON with
references to JSON schema files. The reference format looks as follows:

<div data-render-schema="http://schemas.taskcluster.net/base/v1/exchanges-reference.json">
</div>

The JSON schema for the exchanges reference format is
`http://schemas.taskcluster.net/base/v1/exchanges-reference.json` and references
are validated prior to publication.

---
filename: design/devel/index.md
title: Development
order: 20
---

This section of the manual contains information useful for people working on
the Taskcluster codebase.

The [Taskcluster organization](https://github.com/taskcluster) on Github
contains the source code for all of the microservices, a collection of
supporting libraries, and more.

You can find a list of [open pull
requests](https://github.com/search?utf8=%E2%9C%93&q=user%3ATaskcluster+is%3Aopen&type=Issues&ref=searchresults),
or the aggregated [Travis CI Status](https://travis-ci.org/Taskcluster) to see
how we are doing.

---
filename: design/devel/principles.md
title: Guiding Design Principles for Taskcluster
order: 10
---

At the [2016 tc-worker workweek](http://www.chesnok.com/daily/2016/03/11/workweek-tc-worker-workweek-recap/) the Taskcluster Platform team laid out our _core design principles_. The four key principles are:

* Self-service
* Robustness
* Enable rapid change
* Community friendliness

These are all under an umbrella we call Getting Things Built&#8482;. None of our work matters unless __it works__! Read further for a slightly expanded list of principles!

### Getting Things Built™

#### Self-service

- Task Isolation
- API-driven UI Tools
- Extensible
- Granular Security
- Clearly-defined interfaces
- Separation of concerns

#### Robustness

- Scalability
- Correctness
  - Idempotent APIs
- Minimal Self-hosting
  - Use managed services, e.g. S3, Azure Storage
  - Don't self-host mutable services
- Stateless services
- 12-factor applications

#### Enable Rapid Change

- Agility
- Clearly-defined interfaces
- Microservices
- Separation of concerns

#### Community Friendly

- Transparency
  - Granular Security
- Public by Default
- Self-Service

<img src="/assets/principles.svg" alt="Taskcluster Principles Diagram" width="100%" />

---
filename: design/devel/rfcs.md
title: RFCs
order: 20
---

Taskcluster manages major changes to the platform through "requests for
comment", known as RFCs.  These provide an open, transparent decision-making
process and a way to track ideas from initial proposal through decision and
implementation.

Taskcluster's RFCs are in the [taskcluster-rfcs repository](https://github.com/taskcluster/taskclutser-rfcs).

---
filename: design/devel/idempotency.md
title: Idempotency
order: 30
---

All Taskcluster API operations are idempotent. This means that repeating a
request has no effect, as long as it occurs within a reasonable time.

For example, calling `queue.createTask` twice with exactly the same task
creates only one task. On the other hand, calling the method twice with the
same task ID but different task definitions is an error.

Ensuring idempotency requires some careful design, but is critical for an HTTP
API, as clients may re-try a successful request if the connection fails before
that success is communicated back to the client.

In particular:

 * Identifiers should always be specified by the client, not generated by
   the API endpoint.  Slug IDs provide a simple way to ensure
   (probabilistically) unique IDs without any central coordination.

 * Creation operations should explicitly handle the case where the given
   identifier is already present in the backend storage, comparing the rest of
   the object and only returning an error if the two differ.

 * Update operations, too, should treat updates that make no changes as a
   success condition. For example, it is not an error to resolve an
   already-resolved error, as long as the resolution reason is the same.

 * Over long time durations, idempotency is not required. For example,
   updating a client's scopes after it has expired will fail.

---
filename: design/devel/best-practices/index.md
title: Best Practices
order: 40
---

This section highlights some best practices for developing Taskcluster components and integrations.

---
filename: design/devel/best-practices/scopes.md
title: Scopes and Roles
order: 10
---

## When To Require A Role

When verifying the scopes for an API call or some other operation, the default behavior should be to check for appropriately-parameterized scopes.
These answer the question, "Can the caller do X".

However, when the operation modifies an object which will later perform actions using a role, then it is appropriate to require that the caller possess that role.
For example, hooks trigger tasks using role `hook-id:<hookGroupId>/<hookId>`, so the API calls to create and modify the hook require `assume:hook-id:<hookGroupId>/<hookId>` directly.
It is not enough simply to possess the role's extended scopes -- the caller must possess the assumed role itself.

## Persisting Scopes in Resource

When modifying or creating a resource that will later perform actions on behavor of the _resource creator_,
the scopes delegated by the resource creator should be explicitly specified and saved in the resource.
For example, a task can perform actions on behalf of the _task creator_, however, the scopes delegated to the
task must be explicitly given in `task.scopes`. The service (in the case the queue) is naturally responsible
verifying that the task creator possesses the scopes specified in `task.scopes`.

Services should generally prefer to avoid storing lists of scopes in resources. It makes sense to store a list of
scopes in a temporary resources, such as a task which has a fixed deadline. For permanent resources like workers
or hooks should not contain a list of assigned scopes. Instead they should encode their identity in a role and
assume this role when performing actions. For example, a `workerType` in the aws-provisioner doesn't contain a
list of assigned scopes, instead the workers assumes the role `worker-type:<provisionerId>/<workerType>`.

Following this pattern for permanent resources ensures that any permanent grant of authority can be inspected through roles.

## Encoding Information In Roles

Scopes should only ever be used to evaluate scope satisfaction; never pattern match scopes to try to extract information from them.

A common example of this error is in trying to determine a user's identity based on their credentials.
Since the `Taskcluster-login` service helpfully adds scopes like `assume:mozilla-user:jrainer@ranierfamily.net`, it is tempting to look for a scope matching that pattern and extract the email address.

This has a few awkward failure modes, though.
Administrative users may have multiple matching scopes, or even `assume:mozilla-user:*`.
Even if those administrative users should avoid using your service with such powerful credentials, it's easy to do accidentally and incautious code may assume the user is named `*`.
Other credentials may have no matching scope, but still posess the scopes to authorize the bearer to perform an operation.
Basically, scopes are not a great way to communicate this information.

The appropriate way to determine a user's identity (as described in [Third Party Integration](/manual/integrations/apis/3rdparty)) is to find an email from some less trustworthy source such as the clientId, and then *verify* that email against the scopes, by asking "is `assume:mozilla-user:<email>` satisfied?"


---
filename: design/devel/best-practices/microservices.md
title: Building Microservices
order: 20
---

The Taskcluster microservices are all maintained independently, but we share responsibility for maintaining them.
This shared responsibility is easier for everyone if the implementations are similar, avoiding surprises when moving from one service to another.
This document aims to collect the practices and standards we've agreed on.

These conventions are strongly encouraged for new services and contributions updating existing services to follow them are always welcome.
When we have a good reason to not follow the convention for a specific service, we document why.

## Naming

The name of the service should begin with "Taskcluster", followed by something brief and accurate.
Use this name as the identifier for the project everywhere -- repository, deployment, logging, monitoring, docs, etc.
This makes it easy to predict the URL for the project when we're in a hurry.

It's OK to refer to a service more casually in prose, e.g., "the hooks service" instead of "Taskcluster-hooks".

## Package Mechanics

### Repository

Services should be in a Github repository in the `Taskcluster` organization, with the repo name having the prefix `Taskcluster-`.

### Source Layout

Include all source in the `src/` directory, and all tests in `test/`.
The babel-compile configuration will compile those to `lib/` and `.test/`, respectively.

Within the `src` directory, the main script should be named `main.js`.
This file should use `Taskcluster-lib-loader` as described below, and should serve as the main entry point to the service.
This file will be transpiled to `lib/main.js` and should be set as the `main` property in `package.json`.

### Node

Prefer to use the latest stable node version, and corresponding yarn version. These should both be locked and not specified
as a range. This means in particular no `^` in front of either version spec.
Encode this version both in `package.json` and in any CI configurations such as `.Taskcluster.yml`.

`package.json` should have the `"engine-strict": true` flag set. Preferably directly above the `engines` stanza.

### Compiling

Use [babel-compile](https://github.com/taskcluster/babel-compile) with [babel-preset-Taskcluster](https://github.com/taskcluster/babel-preset-Taskcluster) to compile your application.
Its README describes how to set it up generally, but for a Taskcluster service, you will need to install `babel-compile` and `babel-preset-Taskcluster`.
Include the following in your `package.json`:

```js
  "scripts": {
    "compile": "babel-compile -p Taskcluster src:lib test:.test",
    "pretest": "npm run compile",
    "install": "npm run compile",
  }
```

## Managing Dependencies

Try to keep up-to-date with the latest versions of all Taskcluster libraries.
In general, the implications of updating these libraries should be clear, and the authors are easy to find when things go badly.

Other dependencies should also be kept up-to-date as much as possible.
Tools like [Greenkeeper](https://greenkeeper.io/) can be very useful for this purpose.

### Yarn

We have moved from [npm](https://docs.npmjs.com/cli/npm) to [yarn](https://yarnpkg.com/) for as much as possible. This means that
you should not `npm install` anything and there is no `npm-shrinkwrap.json`. Generally you'll want `yarn install` and `yarn add` in
place of `npm install` and `npm install <package>`. Yarn should keep everything in a `yarn.lock` file that is committed to version
control.

When changing a service's dependencies, use the `yarn` comands.
This will update both `package.json` and `yarn.lock` automatically.

 * `yarn add some-lib`
 * `yarn remove some-lib`
 * `yarn add some-lib@^2.3.0`  (to update an existing dependency's version)

It is the service owner's responsibility to keep dependencies up to date.
The `yarn outdated` command gives a useful overview of available updates.
In general, try to keep packages up to date within semver constraints (so, fix things displayed in red in `yarn outdated`), but be cautious that the new code you are incorporating into your service is trustworthy.
In an ideal world, that means a thorough security review.
In the real world, that probably means a lot less.

## Testing

### Test Setup

Use Mocha to run unit tests, in the `test/` directory.
In order to get useful stacktraces from unit tests, you should `yarn add source-map-support`.
Include the following in `mocha.opts`:

```
--ui tdd
--timeout 30s
--reporter spec
--require source-map-support/register
```

Name the test files `test/*_test.js`, so they will be matched by the `npm test` script given above.
Because we transpile the tests to different directory than Mocha's default (`.test` instead of `test`), we need to be careful that the bash globbing will match all test files.
The easiest way to ensure this is to have all test files directly in the `test` directory and not in sub-directories 
These files should require production code from `../lib/`: `foo = require('../lib/foo')`.

### Helpers

Include any shared test-specific code in `test/helpers.js`.

### ESLint

Use [eslint-config-Taskcluster](https://github.com/taskcluster/eslint-config-Taskcluster) and [eslint-plugin-Taskcluster](https://github.com/taskcluster/eslint-plugin-Taskcluster) along with `mocha-eslint` to check for lint.

To do so, install `eslint`, `babel-eslint`, `mocha-eslint`, `eslint-config-Taskcluster`, and `eslint-plugin-Taskcluster`.
Add the following to the `scripts` section of `package.json`: `"test": "mocha .test/lint.js .test/*_test.js"`.

Create `.eslintrc` in the root of the repository with
```js
{
  "extends": "eslint-config-Taskcluster"
}
```

And the following in `test/lint.js`:
```js
var lint = require('mocha-eslint');

var paths = [
  'src/*.js',
  'test/*.js',
];

lint(paths);
```

### Test Requirements

A simple `git clone` .. `yarn install` .. `yarn test` should run successfully for new contributors.
Anything else dramatically increases the difficulty in getting started.

Where possible, try to write tests that do not require any credentials or access to external services.
Liberal use of fakes, mocks, stubs, etc. allows most application logic to be tested in isolation.
Note that `Taskcluster-lib-loader` allows dependency injection by means of overwrites:

```js
let server = await load('server', {
  profile: 'test',
  dependency1: fakeDep1,
  dependency2: fakeDep2,
});
```

For tests that must have credentials, check for the presence of credentials and mark the suite as pending if they are not available:

```js
suite("things", function() {
  if (!helper.haveRealCredentials) {
    this.pending = true;
  }
});
```

This will generate clear output for anyone running the tests without credentials, showing that many tests were not run.
If they make a pull request, then the full suite will run in automation, and any issues not detected locally will be revealed.

### Configuration

For services which require credentials, it should be possible to supply them in a `user-config.yml` file.
The `typed-env-config` library makes this easy.

The service repository should have a `user-config-example.yml` which has all the necessary settings filled with an illustrative example value or the string '...'.
This helps people to know which credentials they need and how to set them up.
The `user-config.yml` should be included in `.gitignore` to avoid checking in credentials.

## Deployment

### Verification Steps

Somewhere in the README, describe how to deploy the service, if it's anything more complicated than a Heroku app or pipeline.

In any case, include a short description of how to verify that the service is up and running after a deployment.
This may be as simple as loading the relevant tools page and seeing content from the service.

### Logging

Connect the service to the Taskcluster papertrail account.
For Heroku services, follow [the standalone method](http://help.papertrailapp.com/kb/hosting-services/heroku/).
Name the service in papertrail to match the repository name.

## Taskcluster Libraries

### General

Do not use `Taskcluster-base`.
Instead, depend directly on the `Taskcluster-lib-*` libraries the service requires.

The following sections describe best practices for specific platform libraries.

### Taskcluster-lib-loader

The main entry-point for the service should be `src/main.js`, which should use [Taskcluster-lib-loader](https://github.com/taskcluster/Taskcluster-lib-loader) and have the following initialization code:

```js
// If this file is executed launch component from first argument
if (!module.parent) {
  load(process.argv[2], {
    process: process.argv[2],
    profile: process.env.NODE_ENV,
  }).catch(err => {
    console.log(err.stack);
    process.exit(1);
  });
}

// Export load for tests
module.exports = load;
```

Entries in `Procfile`, then, look like `web: node lib/main.js server`.
The web service should always be the component named `server`.
All services, including those run from the Heroku scheduler, should start like this, via `lib/main.js`.

### azure-entities

Each Azure table should be defined in `src/data.js` using a cascade of `configure` calls, one for each version:

```js
var MyEntity = Entity.configure({
  version:              1,
}).configure({
  version:              2,
  migrate: function(item) {
    // ...
  },
}).configure({
  version:              3,
  migrate: function(item) {
    // ...
  },
});
```

The result is a `MyEntity` class that can be setup with additional configuration in a loader component, in `src/main.js`:

```js
{
  MyEntity: {
    requires: ['cfg', 'process', 'monitor'],
    setup: ({cfg, process, monitor}) => {
      return data.MyEntity.setup(_.defaults({
        table:        cfg.app.hookTable,
        monitor:      monitor.prefix(cfg.app.hookTable.toLowerCase()),
        component:    cfg.app.component,
        process,
      }, cfg.azureTable, cfg.Taskcluster));
    },
  },
}
```

In `src/data.js`, it is common to add utility methods for the entity type.
For entities which can be fetched via an API, a `json` method is common:

```js
MyEntity.prototype.json = () => {
  return {
    foo: this.foo,
  };
};
```

### Taskcluster-lib-api

The API definition should be in `src/v1.js` or `src/api.js`:

```js
var api = new API({
  // ...
});

// Export api
module.exports = api;

/** Get hook groups **/
api.declare({
  // ...
});
// ...
```

This is then imported and set up in `src/main.js`:

```js
{
  router: {
    requires: ['cfg', 'profile', 'validator', 'monitor'],
    setup: ({cfg, profile, validator, monitor}) => {
      return v1.setup({
        context: {},
        authBaseUrl:      cfg.Taskcluster.authBaseUrl,
        publish:          profile === 'production',
        baseUrl:          cfg.server.publicUrl + '/v1',
        referencePrefix:  'myservice/v1/api.json',
        aws:              cfg.aws,
        validator,
        monitor,
      });
    },
  },
}
```

#### Error Handling

Do not use `res.status(..)` to return error messages.
Instead, use `res.reportError(code, message, details)`.
The `Taskcluster-lib-api` library provides most of the codes you will need, specifically `InvalidInput`, `ResourceNotFound`, and `ResourceConflict`.

Prefer to use these built-in codes.
If you have a case where you must return a different HTTP code, or clients need to be able to distinguish the errors programmatically, add a new error code:

```js
var api = new API({
  description: [
    // ...
    '',
    '## Error Codes',
    '',
    '* `SomethingReallyBad` (472) - you\'re really not going to like this',
  ].join('\n'),
  errorCodes: {
    SomethingReallyBad: 472,
  },
});
// ...
res.reportError('SomethingReallyBad',
  'Something awful happened: {{awfulthing}}',
  {awfulThing: result.awfulness});
```

Be friendly and document the errors in the API's `description` property, as they are not automatically documented.

### Taskcluster-lib-monitor

*Do not use* `Taskcluster-lib-stats` or `raven`.
Instead, use `Taskcluster-lib-monitor` as described in its documentation.

### Taskcluster-lib-docs

All services should use `Taskcluster-lib-docs` as directed to upload documentation.

The service will include substantial documentation in Markdown format in its `docs/` directory.
The docs library will automatically include the service's `README.md`, as well, and that is a good place to include an overview and development/deployment instructions.

If the service provides an API or pulse exchanges, set it up to publish that information as directed.

---
filename: design/devel/best-practices/libraries.md
title: Building Libraries
order: 30
---

Taskcluster's libraries show a great deal more variety than the microservices.
Some aim to be useful outside of Taskcluster, while others are designed specifically for use in our own services.
As such, they share less in common.
However, we can identify a few best practices.

## Repository

Libraries should be in the `Taskcluster` Github org.
Those specific to Taskcluster should have the repo prefix `Taskcluster-lib-`, while those intended for general use can have arbitrary names.

## Packaging

Use the `files` property in `package.json` to whitelist the files that should be included in the library distribution.
This avoids packaging unnecessary files into the distribution, and more importantly avoids accidentally picking up files containing credentials!
Do *not* include a `.npmignore` file.

This is often as simple as

```js
{
  "files": [
    "lib"
  ]
}
```

You can check what files are being included with `tar -ztf $(npm pack)`.

If the library is transpiled, be sure to include both the ES6/7 input files and the transpiled output.

## Managing Dependencies

We have moved from [npm](https://docs.npmjs.com/cli/npm) to [yarn](https://yarnpkg.com/) for as much as possible. This means that
you should not `npm install` anything and there is no `npm-shrinkwrap.json`. Generally you'll want `yarn install` and `yarn add` in
place of `npm install` and `npm install <package>`. Yarn should keep everything in a `yarn.lock` file that is committed to version
control.

## Part Of The @Taskcluster NPM Org

Ensure that your npm library is part of the @Taskcluster team.

## Publish On Tag

Set up libraries to publish on a tag using [Travis-CI](https://docs.travis-ci.com/user/deployment/npm/).
The appropriate API token is for the 'Taskcluster-bot' user, and the the token itself is available in the password store.
Use email `Taskcluster-accounts@mozilla.com`.

To publish, run `npm version patch` (or `minor` or `major` depending on the semver changes).
This will update the version, create a new git commit, and create a tag.
So you only need to push to upstream with and without `--tags`:

```
git push upsream
git push upsream --tags
```

When the Travis job for that push completes, the new version should be deployed.
Note that the npmjs.com browser UI is cached and may not immediately represent the updated version.

---
filename: design/namespaces.md
title: Namespaces
order: 30
---

Taskcluster has a number of namespaces defined to allow multiple users to get
along without interfering with one another.  The platform itself is agnostic to
the structure of these namespaces, but infrastructure that interacts with the
platform is dependent on the namespaces for security and correctness.

This document necessarily contains information that is specific to users of the
Taskcluster platform.  As such, it is open to contributions from all users who
wish to carve out a section of a namespace -- just [submit a pull
request](https://github.com/taskcluster/Taskcluster-docs).

--- ---

## Projects

Most work at Mozilla falls into "projects", and these provide a nice organizational boundary for controlling access.
We use a consistent name for each project in the various namespaces below -- something simple and without punctuation.
The known projects are:

 * `Taskcluster` -- The Taskcluster platform itself
 * `releng` -- Mozilla release engineering (build, test, and release processes)
 * `gaia` -- B2G/Firefox OS build system
 * `rust` -- Rust development + servo
 * `ateam` -- The engineering productivity team

Please file a pull request to add your project!

--- ---

## Scopes

Many scopes reflect the namespaces given elsewhere in this document, as described in the API documentation for the component.

* `<component>:<action>:<details>` -
   Scopes for most API actions follow this pattern.
   For example, the `queue.defineTask` call is governed by a scope beginning with `queue:define-task:<details>`, where the details describe a hierarchy of task attributes.
   In cases where an action may be limited along any of several dimensions, each of those dimensions should be a separate scope.

* `project:<project>:…` -
   Individual projects should use scopes with this prefix.
   Projects are free to document the contained namespace in this document, link to another document, or leave it undocumented.

--- ---

## Clients

Client names describe the entity making an API call.
They are used for logging and auditing, but not for access control -- all access control is performed with scopes.
ClientIds have the following forms:

 * `mozilla-ldap/<email>` -
   Clients with this name belong to users who have been authenticated against the Mozilla LDAP database.
   These are temporary credentials issued by [Taskcluster-Login](https://github.com/taskcluster/Taskcluster-login).

 * `persona/<email>` -
   Clients with this name belong to users who have been authenticated by Persona, conferring a lower level of trust than Mozilla LDAP.
   These are temporary credentials issued by [Taskcluster-Login](https://github.com/taskcluster/Taskcluster-login).

 * `mozilla-ldap/<email>/*`,
   `persona/<email>/*` -
   Clients with this form are managed by the user identified by the prefix.
   The portion of the name matching `*` is at the discretion of the user.

 * `<component>/*` -
   Taskcluster Platform services generate clientIds with this form.
   The service itself will generally have a clientId of `<component>`.

 * `queue/task/<taskId>/<runId>` -
   Clients of this form represent specific tasks, and are issued by the queue in the form of temporary credentials.

 * `aws-provisioner/worker/<workerGroupId>/<workerId>` -
   Clients of this form represent specific wokers, and are issued by the AWS provisioner in the form of temporary credentials.

 * `project/<project>/*` -
   Clients for a specific project have this prefix.
   Administrators for the project are granted control over this namespace, and may further subdivide it as they see fit.
   They are welcome to document those subdivisions here.

 * `garbage/*` -
   Playground for testing; clients here should not be active in the wild.
   Likewise, deleting or modifying clients with this prefix will not cause production failures.

--- ---

## Roles

Most roles are defined by some kind of automatic usage in a Taskcluster component.
However, some are defined by convention.
Both are listed here:

* `client-id:<clientId>` -
   Roles with this prefix give the scopes for client credentials.
   In general, scopes should be assigned directly to clients, instead.

* `hook-id:<hookGroupId>/<hookId>` -
   Roles of this form give the scopes used to create tasks on behalf of hooks.

* `moz-tree:level:<level>` -
   Roles of this form include the basic scopes available to version-control trees at each of the three Mozilla source-code managament levels.
   They are useful as shorthand to configure `repo:*` roles.
   See [Mozilla Commit Access Policy](https://www.mozilla.org/en-US/about/governance/policies/commit/access-policy/) for information on levels.

* `mozilla-group:<groupName>` -
   Roles of this form represent the scopes available to members of the given Mozilla LDAP group via the login service.

* `mozilla-user:<userName>` -
   Roles of this form represent the scopes available to the given Mozilla LDAP user (email) via the login service.

* `mozillians-group:<groupName>` -
   Roles of this form represent the scopes available to members of the given Mozillians group via the login service.

* `mozillians-user:<userName>` -
   Roles of this form represent the scopes available to the given Mozillians user via the login service.

* `project:<project>:…` -
   Roles of this form are controlled by the corresponding project.

* `project-member:<project>` -
   Roles of this form represent the scopes accorded to members of the given project.
   This role is then be assumed by the appropriate groups.
   The scopes associated with a `project-member` role are:

   * `auth:{crud}-client:project/<project>/*` - manage project-specific clients
   * `auth:{crud}-role:project:<project>:*` - manage project-specific clients
   * `auth:{crud}-role:hook-id:project-<project>/*` - manage scopes for project-specific hooks
   * `project:<project>:*` - all project-specific scopes
   * `queue:get-artifact:project/<project/*` - create project-specific (non-public) artifacts
   * `hooks:modify-hook:project-<project>/*` - manage project-specific hooks
   * `secrets:<verb>:project/<project>/*` - manage project-specific secrets
   * `queue:route:index.project.<project>.*` - manage project routes

* `repo:<host>/<path>:branch:<branch>`,
* `repo:<host>/<path>:pull-request` -
   Roles of this form represent scopes available to version-control pushes and pull requests.

* `repo:<host>/<path>:cron:<jobName>` -
   Roles of this form are used for cron jobs by the periodic task-graph generation support in Gecko.

* `worker-type:<provisionerId>/<workerType>` -
   Roles of this form represent scopes available to workers of the given type.

--- ---

## Artifacts

Artifacts are named objects attached to tasks, and available from the queue component.
Artifact names are, by convention, slash-separated.

* `public/…` -
   The queue allows access to any artifact that begins with `public/` without any kind of authentication.
   Public names are not further namespaced: tasks can create any public artifacts they like.
   As such, users should not assume that an artifact with this prefix was created by a known process.
   In other words, any task can create an artifact named `public/build/firefox.exe` , so do not trust such a file without verifying the trustworthiness of the task.

* `private/…` -
   Artifact names with this prefix are considered non-public, but access to them is otherwise quite broadly allowed (e.g., to all Mozilla employees, contractors and community members under NDA).
   In general, users with narrower requirements than "not public" should select a different prefix and add it to this document.

* `private/docker-worker/…` -
   Artifact names with this prefix are considered non-public, but access to them is otherwise quite broadly allowed to everybody with commit-level 1 access, regardless of NDA state.

* `project/<project>/…` -
   Artifact names with this prefix are the responsibility of the project, which may have further namespace conventions.

--- ---

## Hooks

Hooks are divided into "hook groups", for which the namespace is defined here.
Within a hook group, the names are arbitrary (or defined by the project).

* `Taskcluster` - hooks used internally for Taskcluster maintenance
* `project-<project>` - hooks for a specific project
* `garbage` - playground for testing; hooks can be created here, but anyone can modify or delete them!

--- ---

## Worker Types

Worker types are broken down into `<provisionerId>` and `<workerType>`.
Provisioner IDs are issued individually, with no namespacing.
Worker types are specific to the provisioner ID, but provisioners that provide general services (currently that means `aws-provisioner-v1`) should follow the following guidelines:

* `<project>-*` - worker types designed for a specific project; the suffix is arbitrary and up to the project
* `gecko-t-*` - worker types for gecko tests; the suffix is arbitrary
* `gecko-L-b-*` - worker types for gecko builds, with `L` replaced with the SCM level; the suffix is arbitrary
* `ami-test*` - worker types for testing deployment of new AMIs
* `tutorial` - default worker type for the getting-started tutorial in this documentation
* `github-worker` - default worker type for Github-triggered jobs
* `hg-worker` - default worker type for Mercurial-triggered jobs

Note that there are many worker types not following this convention, as worker types must be kept around for a long time to run old jobs.

--- ---

## Worker IDs

Worker IDs are broken down into `<workerGroup>` and `<workerId>`.
In the present implementation, both of these are arbitrary strings.
For workers started by the AWS provisioner, they are avaiability zone and instance ID, respectively.
For other worker types, anything goes.

--- ---

## Provisioner IDs

Provisioner IDs are limited to 22 characters.
We do not subdivide namespaces; instead, they are enumerated here:

 * `aws-provisioner-v1` -- the AWS provisioner
 * `buildbot-bridge` -- the AWS provisioner
 * `scriptworker-prov-v1` -- the scriptworker provisioner

--- ---

## Scheduler IDs

Scheduler IDs are limited to 22 characters.
We do not subdivide namespaces; instead, they are enumerated here:

 * `tc-diagnostics` -- the Taskcluster diagnostics tool

--- ---

## Docker-Worker Caches

Docker-worker caches are located on individual host machines, and thus may be shared among tasks with the same workerType.
The namespaces for these caches help to avoid collisions and prevent cache-poisoning attacks.

Cache names do not contain directory separators.

* `gaia-…` -
  Caches with this prefix are used by gaia builds, limited to the https://github.com/mozilla-b2g/gaia repository

* `tooltool-cache` -
  This cache contains cached downloads from tooltool.
  Since tooltool is content-addressible, and verifies hashes on files in the cache, there is no risk of cache poisoning or collisions.

* `level-<level>-<tree>-…` -
  Caches with these prefixes correspond to tasks at the corresponding SCM levels.
  See [Mozilla Commit Access Policy](https://www.mozilla.org/en-US/about/governance/policies/commit/access-policy/) for information on levels.
  The rest of this namespace is free-form and generally divided by task type, with the following common cases:

  * `level-<level>-<tree>-decision` - decision task workspace
  * `level-<level>-<tree>-tc-vcs` `level-<level>-<tree>-tc-vcs-public-sources` - Taskcluster-vcs caches
  * `level-<level>-<tree>-linux-cache` - cache of `~/.cache`, containing Python packages among other things
  * `level-<level>-<tree>-build-<platform>` - workspace cache for builds for the given platform

--- ---

## Secrets

Secrets provide key-value storage governed by scopes.
As such, it's very important that secrets not be unexpectedly made accessible to users who should not see them.
Secret names have the following structure:

* `garbage/<ircnick>/` -
  Secrets with this prefix are not actually secret - lots of people have access to them.
  This is a place to test out interfaces to the secrets API, but do not store anything important here.

* `project/<project>/` -
  Secrets with this prefix are the exclusive domain of the given project.
  Users not associated with a project should not be given scopes associated with the project's secrets!

* absolute name `repo:<host>/<path>:branch:<branch>` or prefix `repo:<host>/<path>:branch:<branch>:` -
  For secrets that should only be available to a single branch of a repository, and not other branches or forks of that repository.
  The first form is for a single secret object containing all secrets; the second form is for using multiple secret objects.
* `repo:<host>/<path>:pull-request` -
  Secrets named in this manner will be available to repository forks, branches, and pull requests via the corresponding roles.

--- ---

## Indexes

The index provides a nice, dot-separated hierarchy of names. When using these as AMQP routes, they are prefixed with `index.`, so for example the project 'foo' might route messages about its level 3 repository tasks to routes starting with `index.project.foo.level-3.…`

* `buildbot` - the "old" index for Buildbot builds; do not use

* `funsize.v1` -
   Tasks indexed under this tree represent funsize tasks.
   These are the responsibility of the release engineering team.

* `gaia.npm_cache.<nodever>.<platform>.<revision>` -
   Tasks indexed here have generated the `node_modules` directory required for the given revision.
   These are the responsibility of the B2G automation team.

* `garbage.<ircnick>` -
   Anything goes under this index path.
   Use this for development and experimentation.

* `gecko.v1` - another "old" index for builds; do not use

* `gecko.v2.<tree>.revision.<revision>.<platform>.<build>`,
* `gecko.v2.<tree>.latest.<platform>.<build>` -
   Index for Gecko build jobs, either by revision or for the latest job with the given platform and build.
   These are the responsibility of the release engineering team.

* `tc-vcs.v1` -
   Tasks indexed under this prefix represent caches used by the Taskcluster VCS tool to avoid crushing version-control hosts.
   These are the responsibility of the Taskcluster team.

* `project.<project>.…` -
  Tasks indexed under this prefix are the domain of the respective project.

---
filename: using/index.md
title: Using Taskcluster
order: 50
---

There's so much more of Taskcluster to explore! The details are all in the
[reference](/reference), but it's not always clear where to start to solve a
particular problem.

This section addresses a series of common use cases, sketching recommended
solutions that will help you figure out where to begin exploring to learn more.

---
filename: using/github.md
title: Integrating with Github
order: 10
---

Taskcluster provides a convenient integration with Github to enable starting
tasks when events occur on a github repository.  Setting up the integration is
simple: add the integration to the Github organization and repository
(requiring administrator rights on the organization), then add a file called
`.taskcluster.yml` to the root of the repository.

We recommend using [The quickstart
tool](https://tools.taskcluster.net/quickstart/) to get started.  For more
in-depth documentation, [the reference
pages](https://docs.taskcluster.net/reference/integrations/github/docs/usage)
provide detailed information.

## Task Security

The github integration provides per-repository configuration for building pull
requests, with the configuration controlled from the default (`master`) branch.
It is convenient for contributors to run tasks on pull requests, but those
tasks then run arbitrary, un-reviewed code. If the tasks have access to
resources that should not be publicly available, then a pull request can
provide a route for a malicious contributor to abuse those resources.

With a little extra care in setting up the `repo:github.com/<org>/<repo>`
scopes, it is possible to provide scopes for branches (containing trusted,
reviewed code) that differ from the scopes given to pull requests.

---
filename: using/secrets.md
title: Handling Secrets
order: 20
---

Taskcluster is a very open platform, so a secret embedded in a task definition
or a task artifact is not very secret.

Ideally, tasks would not need access to any secrets, since most tasks run
source code from a version-control repository -- even un-trusted code in the
case of a Github pull request. It's all too easy to accidentally or maliciously
output a secret to the task logs, and such a disclosure is unlikely to be
noticed quickly.

The [Secrets service](/reference/core/secrets) provides a simple way to store
JSON-formatted secrets in a secure fashion. Access to secrets can be controlled
by [scopes](/manual/design/apis/hawk/scopes).

The most common approach is to use the Tools site to create secrets named
according to the [namespaces document](/manual/design/namespaces), then read
those secrets in tasks via the taskcluster proxy. Access to the secrets is
granted by adding a scope to the repository's role.

For example, generating this documentation site requires access to the
Mozillians API to download information for the [people](/people) page, and that
API key is stored in a secret named `project/taskcluster/tc-docs/mozillians`.
The task definition in `.taskcluster.yml` has a scope to read this secret and
enables the docker-worker taskclusterProxy feature:

```yaml
scopes:
  - "secrets:get:project/taskcluster/tc-docs/mozillians"
payload:
  features:
    taskclusterProxy: true
# ..
```

The role for the master branch of the repository contains the same scope:

```yaml
"role:github.com/taskcluster/taskcluster-docs:branch:master"
  - "secrets:get:project/taskcluster/tc-docs/mozillians"
```

Note that the `..:pull-request` role does *not* have this scope, so pull
requests cannot access the Mozillians API key.

The script that runs in the task calls the secrets API using the Taskcluster
client and extracts the API key from the resulting JSON object:

```js
var getApiKey = () => {
  if (process.env.MOZILLIANS_SECRET) {
    var secrets = new taskcluster.Secrets({baseUrl: 'http://taskcluster/secrets/v1/'});
    return secrets.get(process.env.MOZILLIANS_SECRET).then(secret => secret.secret['api-key']);
  }
```

There are taskcluster clients available in many languages - use the one most
comfortable for you.

_Note:_ the `garbage/` namespace is provided for experimentation with the API,
but it is what it says on the tin: garbage, and more importantly, readable by
anyone.  Never put important information in a secret under `garbage/`!

---
filename: using/indexing.md
title: Indexing Tasks
order: 30
---

Since task IDs are random, it can be difficult to find a task after it has been
created without recording that task ID somewhere. The Github integration will
link to tasks from pull requests and commits, and services like Treeherder
gather and link to tasks for specific projects, but neither of these solutions
is especially flexible.

The [Index service](/reference/core/index) stores references to completed tasks
in a hierarchical naming structure, similar to a directory tree. Careful naming
allows a more flexible approach.

## Finding the Latest Build

A very common case is to provide a stable link to the most recent build of a
repository. Suppose that the Amazing Cats build process produces an artifact
named `public/Amazing-Cats.dmg`, and we wish to create a URL that will always
point to the most recent DMG.

We will use the index path `project.amazingcats.builds.latest`. We add a route
to the task with the prefix `index.`:

```yaml
routes:
 - index.project.amazingcats.builds.latest
```

This will require a scope when creating the task. As part of the project setup,
the Github repo role `repo:github.com/amazing/*` was granted
`queue:route:index.project.amazingcats.*`, so this is already in place.

With this change, each *successful* build task will be indexed at the selected
path. We can find the task through the index browser in the tools site, and
find the DMG in the artifacts of the latest run

The Index service provides a convenient `index.findArtifactFromTask` method
that will do all of that in one go, redirecting to the artifact itself. For
public artifacts, the method requires no authentication, so it can be called by
any HTTP client. Based on the documentation for the service, we can construct a
URL to the artifact:
`https://index.taskcluster.net/task/project.amazingcats.builds.latest/artifacts/Amazing-Cats.dmg`.

This link could be embedded in the Amazing Cats README or documentation site
for download.

_Note:_ A path like this might point to a new task at any moment. While it is
safe to download a single artifact like this, if your use-case involves
downloading a collection of related artifacts, it may be problematic. Suppose
the Linux version of Amazing-Cats contains both an executable binary and an
associated file containing debug symbols. Downloading those two files via the
index path may result in binary and symbols from different tasks if a task
completes during the download process.

There are two ways to avoid this issue. The simplest is to bundle all important
results into a single artifact, but for large artifacts this can be wasteful.
The more complex option is to script use of the `index.findTask` method to
determine the latest task ID, then download all artifacts from that single task
using `queue.getLatestArtifact`.

---
filename: using/scheduled-tasks.md
title: Running Periodic Tasks
order: 40
---

It is common to build a project on a nightly basis, or to "refresh" some output
periodically. For example, this documentation site is regenerated several times
per hour, combining documentation from all of the taskcluster repositories.

The [Hooks service](/reference/core/hooks/) is responsible for creating
pre-defined tasks in response to external events.  At the moment, it only
supports creating tasks at particular times, but this is exactly the
functionality we need. For the documentation website, a hook is configured to
run every 15 minutes. Its task body contains a docker-worker payload that runs
the same script as for a push to the master branch, and provides the necessary
scopes.

## Using Hooks

Hooks are named with a `hookGroupId` and a `hookId`. The group IDs follow a
pattern given in the [namespaces document](/manual/design/namespaces). The
`hookId` is arbitrary, although it is a good idea to think carefully about the
names and use long, hierarchical names.

The scopes available to a hook are given by a role. This allows separation of
hook management from scope management, and the full generality of scope
expansion. The role for a hook is named `hook-id:<hookGroupId>/<hookId>`. The
role must include all of the scopes required to create the task, including
`queue:create-task:<provisionerId>/<workerType>`.

The scopes actually used by the hook's task are, of course, defined in
`task.scopes`, which must be satisfied by the hook's role. These need not
include `queue:create-task:<provisionerId>/<workerType>` unless the task will
be creating more tasks (for example, a [decision task](/manual/using/task-graph)).

For the documentation repository, the hook is named `taskcluster/docs-update`.
The task has scopes

```yaml
task:
  scopes:
    - "auth:aws-s3:read-only:taskcluster-raw-docs/*"
    - "auth:aws-s3:read-write:docs-taskcluster-net/"
    - "secrets:get:project/taskcluster/tc-docs/mozillians"
```

and the hook role has those scopes plus the required `create-task` scope:

```yaml
"hook-id:taskcluster/docs-update":
  - "auth:aws-s3:read-only:taskcluster-raw-docs/*"
  - "auth:aws-s3:read-write:docs-taskcluster-net/"
  - "secrets:get:project/taskcluster/tc-docs/mozillians"
  - "queue:create-task:aws-provisioner-v1/github-worker"
```

## Advice

Hooks are not easy to manage directly, and exist far from the rest of the
infrastructure for your project. Try to avoid embedding too much detail into
the hook definition.

For simple work (for example, a periodic cache refresh), create a shell script
in your code repository, and write a hook that will check out the latest source
and run that script. Then any modifications to the cache-refresh process can be
handled using your usual development processes, instead of an update in the
hooks API.

For more complex purposes, invoke a [decision task](/manual/using/task-graph)
thad runs within a source checkout and draws the details of what to achieve out
of that source checkout.

---
filename: using/task-graph.md
title: Building Task Graphs
order: 50
---

Useful work often requires more than one task. For example, new source code
might be built on several platforms, or slow tests might be split up to run in
parallel.

Taskcluster's developers and users have established a convention for
accomplishing this, called a "decision task". This is a single task which runs
first, and creates all of the required tasks directly by calling the
`queue.createTask` endpoint.

This has a number of advantages over other options:

 * The set of tasks to run can be specified in the same source tree as the code
   being built, allowing unlimited flexibility in what runs and how.
 * Several event sources can all create similar decision tasks. For example, a
   push, a new pull request, and a "nightly build" hook can all create decision
   tasks with only slightly different parameters, avoiding repetition of complex
   task-definition logic in all of the related services.

The disadvantage being Taskcluster does not provide an easy way to design decision
tasks. The Gecko (Firefox) project has a sophisticated implementation, but it
is not designed to be used outside of the Gecko source tree. Other projects
are left to implement decision tasks on their own.

We on the Taskcluster team would like to remedy this shortcoming, but it is not
an active project.  Contributors are welcome!

## Conventions

We have established a few conventions about decision tasks. These are based on
our experience with Gecko, and will help avoid some pitfalls we encountered.
They will also ensure that your decision tasks are compatible with any later
formalisms we may add around decision tasks.

 * A decision task is the first task in a task group, and that task group's
   `taskGroupId` is identical to its `taskId`. As a corollary, it is easy to
   find the decision task for a subtask: simply treat its `taskGroupId` as a
   `taskId`.

 * Decision tasks call `queue.createTask` using the TaskclusterProxy feature,
   meaning that no Taskcluster credentials are required, and the scopes
   available for the `createTask` call are those afforded to the decision task
   itself.

 * A decision task runs with all of the scopes that any task it creates might
   need. It calculates precisely the scopes each subtask requires, and supplies
   those to the `queue.createTask` call.
 
 * All subtasks depend on the decision task. This ensures that, if the decision
   task crashes after having created only some of the subtasks, none of those
   tasks run and the decision task can simply be re-triggered.

---
filename: using/artifacts.md
title: Working With Artifacts
order: 60
---

XXX

---
filename: using/s3-uploads.md
title: Uploading To S3
order: 70
---

XXX

---
filename: using/task-notifications.md
title: Task Notifications
order: 80
---

XXX

---
filename: using/administration.md
title: Administration
order: 90
---

XXX

---
filename: using/integration/index.md
title: Integration with Other Applications
order: 200
---

Taskcluster is built to mix in with other applications in a larger CI
environment. Aside from the platform services, no Taskcluster component is
privileged and other implementations are always possible.

For example, the Taskcluster tools site is built to be generally useful and to
provide a clear view of the Taskcluster system, but may not be suited to
providing the information your developers need. It's easy to write a new
frontend that uses the Taskcluster services to generate a dashboard specific to
your project, and even to authenticate users and allow them to manipulate tasks
and other APIs via your site.

Backend applications can create tasks (on pushes to that old CVS repository?)
and call other Taskcluster API methods. They can also listen for
Taskcluster-related events via Pulse and react appropriately, perhaps recording
test performance statistics into a database for later analysis.

---
filename: using/integration/frontend.md
title: Frontend Applications
order: 10
---

If you are building a frontend application which would like to interact with
Taskcluster APIs on behalf of users, Taskcluster currently does not provide a
great experience. The major issue is that Taskcluster does not provide user
authentication, but just provides a set of Taskcluster credentials.

This situation will change soon, though -- see [this
RFC](https://github.com/taskcluster/taskcluster-rfcs/issues/9) for details.

## Guidelines

Before jumping in to the technical details, a few words of caution are
required.  When a user clicks "Grant" for your service, they are trusting your
service with their credentials.  Even for a trivial service, this can be a
heavy burden!

Limit the places you copy these credentials:

 * Do not send them to your backend, if possible.
 * Do not log them in your backend.
 * Redirect, or rewrite `window.location`, to remove them from the browser's location bar.

*Do not* use `clientId`s for authentication.  First, because mere possession of
a credential with a clientId and some 44-character accessToken does not prove
anything until you have validated that the accessToken is valid.  Second, even
if you validate the accessToken, Taskcluster is fairly permissive in creation
of temporary credentials with arbitrary `clientId`s, by design.  The
information you may rely on for authorization is contained in the list of
scopes returned from the
[`auth.authenticateHawk`](/reference/platform/auth/reference/api-docs#authenticateHawk)
method.

*CAUTION:* remember that you are dealing with powerful credentials belonging to
real users.  Think carefully about how you handle those credentials, and how
you can minimize the handling that you do.  Beyond that, what plans you have
for detecting and handling credential disclosure?

## Getting Credentials

You can interact with Taskcluster by redirecting your users to a Taskcluster
URL as described below.  If the user authenticates correctly and grants your
service access, they are redirected back to your site with a set of [temporary
credentials](temporary-credentials) based on their assigned scopes.

Then store the resulting credentials in the JavaScript heap or (to survive
reloads) LocalStorage.  Simply use those credentials along with the Taskcluster
client to make calls to Taskcluster APIs.

If you need to display some identifying information for the user, such as a
"you are logged in as.." tooltip, you may use the clientId.  However, as
mentioned above, the clientId should not be used alone to authenticate a user,
as it can be forged. If you need to definitively identify a user, contact the
Taskcluster team to talk about available options and plans.

If you would like to perform additional error-checking, you can use those
credentials to call
[`auth.currentScopes`](/reference/platform/auth/reference/api-docs#currentScopes).

## Don't be a [Confused Deputy](https://en.wikipedia.org/wiki/Confused_deputy_problem)

If the service you are building acts on behalf of users, but uses its own
Taskcluster credentials (for example, on the backend, to avoid storing users'
credentials), you must be very careful to avoid allowing malicious users to
abuse your privileges through scope escalation.  Scope escalation is when a
user can cause some action for which they do not have the appropriate scopes.

For example, your service might create tasks based on a user's selections in a
browser form.  If the service has the scopes to create tasks that can read
secrets, but does not verify that the user has such permission, then the
service would provide a way for malicious users to create tasks that display
those secrets.  The user has escalated their access to include those scopes
which they did not already possess.

The phrase "confused deputy" refers to the case where a service performs some
actions on a user's behalf (as a deputy), but allows scope escalation
(confused).

### Don't be a Deputy

The best way to avoid this issue is to not act as a deputy.  This means using
the user's own Taskcluster credentials to create the tasks, rather than using
credentials assigned to the service.  In the example above, ideally the user's
credentials would be stored locally in the browser, and the client-side code
would call the queue's `createTask` method directly.  A less optimal solution
would involve sending the user's credentials to the backend and using those
credentials to call `createTask` on the backend.

### Deputy Tools

If you must act as a deputy -- for example, running tasks without a browser
involved -- Taskcluster provides a tool to prevent confusion.

This tool is [Authorized Scopes](authorized-scopes), which are used with an API
call to reduce the scopes available to that call.  For example, a service which
creates tasks during low-load times might have a `createDelayedTask` API method
taking a time and a task definition.

The obvious, but incorrect, way to authenticate this would be to duplicate the
`queue.createTask` permissions model, verifying the caller possess the scopes
in `task.scopes`.  When the system load fell and it was time to run the task,
the service would call `queue.createTask` using its own credentials.  But there
are already some subtleties in queue's permissions model, and that model may
change over time, introducing a scope-escalation vulnerability.

The better answer is to capture the scopes of the credentials used to call
`createDelayedTask`.  When calling `queue.createTask`, pass those scopes as
`authorizedScopes`.  This method avoids any interpretation of scopes by the
delayed-task service, so there is no possibility of a scope escalation.

This better answer does lose the advantage of error-checking:
`createDelayedTask` will happily accept a task for which the user does not have
scopes, but will fail when the service calls `queue.createTask`.  It's safe to
fix this with an approximation to the queue permissions model, as long as the
`authorizedScopes` are still enforced.  The failure modes for this check are
acceptable: either `createDelayedTask` refuses to create a delayed task which
should be accepted, or it accepts a task which will later fail due to the
`authorizedScopes`.

---
filename: using/integration/backend.md
title: Backend Services
order: 20
---

If you are building a CI-related service, it is sensible to design it to accept
Taskcluster credentials.

This is quite simple: call
[`auth.authenticateHawk`](/reference/platform/auth/reference/api-docs#authenticateHawk)
from your backend with the appropriate parts of the HTTP request.  Then verify
that the returned scopes satisfy the scopes required for the operation being
protected.  There is no need to "register" the scopes you would like to use,
but see the [namespaces document](/manual/design/namespaces) for guidance on
selecting appropriate names.

The advantage of this approach is that it facilitates service re-use: anyone
who is familiar with Taskcluster APIs can call your API, whether from a task,
the command line, the browser, or another service.  Furthermore, the backend
never sees the credentials, just the Hawk signature.

If you build a user interface around this approach, it is safe to display the
clientId to the user so they can recognize the login.  Just be cautious of the
warning in the previous section regarding using `clientId`s for authentication.

---
filename: using/integration/pulse.md
title: Pulse Integrations
order: 40
---

XXX

---
filename: using/integration/libraries.md
title: Client Libraries
order: 50
---

The Taskcluster client libraries enable you to interface with Taskcluster in your automation.

### Node.js Client Library

* Package and Documentation on [npm](https://www.npmjs.com/package/Taskcluster-client)
* Source and Documentation on [github](https://github.com/taskcluster/Taskcluster-client)

### Python Client Library

* Package on [pypi](https://pypi.python.org/pypi/Taskcluster)
* Documentation and Source on [github](https://github.com/taskcluster/Taskcluster-client.py)

### Go (golang) Client Library

* Source and Usage Documentation on [github](http://Taskcluster.github.io/Taskcluster-client-go)
* API Documentation on [godoc](https://godoc.org/github.com/taskcluster/Taskcluster-client-go)

_(Go checks out libraries directly from source control, so no need for a package. Yay!)_

### Java Client Library

* Package on [maven central](http://search.maven.org/#search|gav|1|g%3A%22org.mozilla.Taskcluster%22%20AND%20a%3A%22Taskcluster-client%22)
* Source and Usage Documentation on [github](http://Taskcluster.github.io/Taskcluster-client-java)
* API Documentation [javadocs](http://Taskcluster.github.io/Taskcluster-client-java/apidocs/)

### Taskcluster CLI

* Package and Documentation on [npm](https://www.npmjs.com/package/Taskcluster-cli)
* Source and Documentation on [github](https://github.com/taskcluster/Taskcluster-cli)
