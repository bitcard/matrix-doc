# MSC2654: Unread counts

Homeservers currently send clients counts for missed notifications (and
highlights) in responses to `/sync` requests, which allows them to display that
information to clients. However, when it comes to unread messages, it doesn't
provide a count. This means clients have no way of telling users how many
messages they've missed since they last read a given room.

This MSC is an alternative to
[MSC2625](https://github.com/matrix-org/matrix-doc/pull/2625) which doesn't use
push rules to calculate unread counts, allowing for simpler implementations.


## Proposal


### Extended response to `/sync`

This MSC proposes the addition of an `unread_count` parameter in the `Joined
Room` structure of
[`/sync`](https://matrix.org/docs/spec/client_server/r0.6.1#get-matrix-client-r0-sync)
responses.

The value of this parameter is a count of every unread event (see below for a
list of criteria an event needs to match to be marked unread) since the latest
read receipt (i.e. the latest `m.read` read marker) for the given user in the
given room. If no read receipt exists for this user in this room, then the value
of the parameter is a count of every unread event since that user joined it.


### Unread events

An event should be counted in the unread count only if it matches at least one
of the following criteria:

* include a non-empty `body` parameter in its content
* be a state event which type is one of:
    * `m.room.name`
    * `m.room.topic`
    * `m.room.power_levels`
* be an encrypted message (i.e. have the `m.room.encrypted` type)

Additionally, an event should _not_ be counted in the unread count if it matches
at least one of the following criteria, even if it is otherwise eligible to be
included in the count:

* include a `m.relates_to` parameter in its content that includes a `rel_type`
  parameter, which value is `m.replace`. This matches the definition of an edit
  in [MSC1849](https://github.com/matrix-org/matrix-doc/pull/1849).
* include a `msgtype` parameter in its content which value is `m.notice`.

Finally, a redaction to an event that was marked as unread should exclude that
event from the unread count.


### Implementation notes on `POST /_matrix/push/v1/notify`

As homeservers weren't previously calculating unread counts, missed
notifications counts were usually used to calculate the the `unread` parameter
of the `Counts` structure of [`POST
/_matrix/push/v1/notify`](https://matrix.org/docs/spec/push_gateway/latest#post-matrix-push-v1-notify)
requests. This parameter must now show the unread count described in this
proposal.


## Potential issues

This MSC mentions edits, which are specified in
[MSC1849](https://github.com/matrix-org/matrix-doc/pull/1849). Therefore, it is
unclear whether this MSC can be merged before MSC1849.


## Alternatives

As mentioned in the introduction of this proposal,
[MSC2625](https://github.com/matrix-org/matrix-doc/pull/2625) proposes an
alternative implementation of this feature using push rules. However, given the
complexity of push rules and that MSC2625 requires to implement a new behaviour
for the `mark_unread` action, using push rules for this feature doesn't seem to
be adding much values, and seems to add unnecessary complexity to its
implementation.


## Unstable prefix

While this feature is in development, the following names will be in use:

| Proposed final name | Name while in development |
| --- | --- |
| `unread_count` | `org.matrix.msc2654.unread_count` |