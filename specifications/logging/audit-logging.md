# Audit Logging Specification

Audit logging is an opinionated log stream that records important life cycle events.
These events may be of use to:

* See who is creating what and when (OPEX, especially as resources may be short lived and not display in the current view)
* See who deleted and modified what (self-service trouble shooting)
* See if anyone is trying to do something they shouldn't (policy enforcement)

The one thing in common is they reduce the burden on a platform operator and allow users to self-administer.

## Changelog

- v1.0.0 2024-07-08 (@spjmurray): Initial RFC

## Storage and Retention

Kubernetes doesn't do a good job of retaining things, be they events (1h), logs (whatever the rotation policy is).
Unikorn doesn't like state, because who wants to be a DB?

The approach we opt for is to leverage best in breed, cloud native solutions.
These exist in the form of log aggregation systems e.g. Logstash, Loki.

Our job then becomes as simple as generating logs to standard out, as is standard in Kubernetes, then delegating log collection, aggregation and storage to a 3rd party vendor.
Anything that falls into this broad category is out of scope.

## Signal to Noise Ratio

We keep this to an absolute minimum, so that what is output is of use to someone.
Logging every GET to poll updates is a vast waste of time, storage, money etc.

We make the following assumptions:

* GET, OPTIONS, HEAD have no side effects so are meaningless to most end users (operators have other means to detect DoS and intrusion attacks).
* If there's no scoping information, it cannot be tied to an organizational entity, so is dropped
* If there's no user information (i.e. unauthenticated APIs), there is no attribution so is dropped.

## Log Format

We already utilize Uber Zap structured JSON logging, because we all agree logs are data, they should be easy to parse, index and query.

To this end we will use the existing logging in Unikorn and extend it as described below to facilitate audit and action logs.

Consider the following:

```json
{
	"level": "info",
	"ts": "2024-07-08T13:01:02Z",
	"msg": "audit",
	"component": {
		"name": "unikorn-identity",
		"version": "v1.0.0"
	},
	"actor": {
		"subject": "joe.bloggs@acme.com"
	},
	"operation": {
		"verb": "DELETE"
	},
	"scope": {
		"organizationID": "e9711b20-625f-4b7a-84ee-2fb5ce66389e",
		"projectID": "d76c582f-5d06-453c-b0a3-14a628672f85"
	},
	"resource": {
		"type": "projects",
		"id": "d76c582f-5d06-453c-b0a3-14a628672f85"
	},
	"result": {
		"status": 202
	}
}
```

> [!NOTE]
> Additional fields may be present in the log entry, as injected by ancestor middleware, and should be ignored.
> Due to middleware ordering, OpenTelemetry tracing fields will be present to correlate audit events across components if desired.

* **level**: log level as defined by Zap, will be "info" unless otherwise stated.
* **ts**: RFC3339 timestamp.
* **msg**: a value of `audit` denotes this is an audit log, and should be used to filter these messages from the rest of the log stream.
* **component**: describes the service that raised the log, and can be used for filtering e.g. cluster creation spans the cluster service and the region service.
* **actor**: records information about who initiated the operation.
* **operation**: describes the operation, this will contain a HTTP method as specified in [RFC2616 section 9](https://datatracker.ietf.org/doc/html/rfc2616#section-9).
* **scope**: records any parameters from the HTTP URL path, and can be used to link operations to specific organizations, projects or resources.
* **resource**: details the exact resource that the operation has affected or would have affected.
* **result**: details the outcome of the operation, as defined by [RFC2616 section 10](https://datatracker.ietf.org/doc/html/rfc2616#section-10).
