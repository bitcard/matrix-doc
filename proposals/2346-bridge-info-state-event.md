# MSC 2346: Bridge information state event

Many rooms across the Matrix network are currently bridged into third party networks, using bridges.
However the spec does not contain a cross-federated method to determine which networks are
bridged into a given room.

There exists a way to do this in a local setting, by using the [/thirdparty/location](https://matrix.org/docs/spec/application_service/r0.1.2#get-matrix-app-v1-thirdparty-protocol-protocol) API but this creates a splitbrain view across the
federation and is an unnacceptable situation.

Many users have taken to peeking at the list of aliases for a giveaway alias like `#freenode_` or
looking for bridge bots or users with a `@_discord_` prefix. This is an unacceptable situation,
as it assumes prior knowledge of these networks and an understanding of how bridges operate.

## Proposal

This proposal attempts to address this problem by providing a single state event for each bridge in a room
to announce which channels have been bridged into a room.

It should be noted that this MSC is intended to provide the baseline needed to display information about
a bridge, and nothing more. See the "Future MSCs" section for more information.

This proposal is heavily based upon my previous attempt [#1410](https://github.com/matrix-org/matrix-doc/issues/1410)
albeit with a notably reduced set of features. The aim of this proposal is to offer information about the
bridged network and nothing more.

### `m.bridge`

```js
{
    "state_key": "org.matrix.appservice-irc://{protocol.id}/{network.id}/{channel.id}",
    "type": "m.bridge",
    "content": {
        "creator": "@alice:matrix.org", // Optional
        "protocol": {
            "id": "irc",
            "displayname": "IRC"
        },
        "network": { // Optional
            "id": "freenode",
            "displayname": "Freenode", // Optional
            "avatar": "mxc://foo/bar", // Optional
            "external_url": "irc://chat.freenode.net" // Optional
        },
        "channel": {
            "id": "#friends",
            "displayname": "Friends", // Optional
            "external_url": "irc://chat.freenode.net/#friends" // Optional
        },
        // Custom vendor-specific keys
        "org.matrix.appservice-irc.room_mode": "+sn",
    },
    "sender": "@appservice-irc:matrix.org"
}
```

The `state_key` must be comprised of the bridge's prefix, followed by the `protocol.id`, followed by the `network.id`, followed by the `channel.id`.
Any `/`s must be escaped into `%2F`. The bridge prefix can be anything, but should uniquely identify the bridge software
that consumes the event. E.g. The matrix.org IRC bridge `matrix-org/matrix-appservice-irc` becomes `org.matrix.appservice-irc`.
This is to help distinguish two bridges on different softwares which may conflict.

The `sender` should be the MXID of the bridge bot.

The `creator` field is the name of the *user* which provisioned the bridge. In the case of alias based bridges, where the
creator is not known -- it may be omitted.

The `protocol` field describes the protocol that is being bridged. For example, it may be "IRC", "Slack", or "Discord". This
field does not describe the low level protocol the bridge is using to access the network, but a common user recongnisable
name. 

The `network` field should be information about the specific network the bridge is connected to. 
It's important to make the distinction here that this does *NOT* describe the protocol name, but the specific network
the user is on. For protocols that do not have the concept of a network, this field may be omitted.

The `channel` field should be information about the specific channel the room is connected to. 

The `id` field is case-insensitive and should be lowercase.

The `external_url` key is a optional link to a connected channel or network that works in much the same way as
`external_url` works for bridged messages in the AS spec.

In terms of hierachy, the protocol can contain many networks, which can contain many channels.

The event may contain information specific to the bridge in question, such as the mode for the room in IRC. These keys
should be prefixed by the bridge's name. Clients may be capable of displaying this extra information and are free to do so.

### Example Content

#### XMPP

An example of a straight forward messaging bridge, such as the XMPP (bifrost) bridge:

```js
{
    "state_key": "org.matrix.matrix-bifrost://xmpp/muc.xmpp.org/xsf@muc.xmpp.org",
    "type": "m.bridge",
    "content": {
        "creator": "@alice:matrix.org",
        "status": "active",
        "protocol": {
            "id": "xmpp",
            "displayname": "XMPP"
        },
        "network": {
            "id": "muc.xmpp.org",
            "displayname": "XSF",
            "external_url": "xmpp:muc.xmpp.org"
        },
        "channel": {
            "id": "xsf@muc.xmpp.org",
            "displayname": "XSF Discussion",
            "external_url": "xmpp:xsf@muc.xmpp.org"
        }
    },
    "sender": "@xmpp:matrix.org"
}
```

#### GitHub

An example of a non-messaging bridge, such as the GitHub bridge:

```js
{
    "state_key": "uk.half-shot.matrix-github://github/matrix-org%2Fmatrix-doc/2346",
    "type": "m.bridge",
    "content": {
        "creator": "@alice:matrix.org",
        "status": "active",
        "protocol": {
            "id": "github",
            "displayname": "GitHub"
        },
        "network": {
            "id": "matrix-org/matrix-doc",
            "external_url": "https://github.com/matrix-org/matrix-doc"
        },
        "channel": {
            "id": "2346",
            "displayname": "MSC2346: Bridge information state event",
            "external_url": "https://github.com/matrix-org/matrix-doc/pull/2346"
        },
        "uk.half-shot.matrix-github.merged": false,
        "uk.half-shot.matrix-github.opened_by": "Half-Shot",
    },
    "sender": "@github:matrix.org"
}
```

#### Mastodon feed

An example of a feed oriented bridge.

```js
{
    "state_key": "org.matrix-org.matrix-mastodon://mastodon/mastodon.matrix.org/@matrix",
    "type": "m.bridge",
    "content": {
        "creator": "@alice:matrix.org",
        "status": "active",
        "protocol": {
            "id": "mastodon",
            "displayname": "Mastodon"
        },
        "network": {
            "id": "mastodon.matrix.org",
            "external_url": "https://mastodon.matrix.org"
        },
        "channel": {
            "id": "@matrix",
            "displayname": "Matrix.org",
            "external_url": "https://mastodon.matrix.org/@matrix"
        },
        "org.matrix-org.matrix-mastodon.bio": "An open standard for decentralised persistent communication. Toots by @matthew, @Amandine & co.",
        "org.matrix-org.matrix-mastodon.joined": "May 2017",
    },
    "sender": "@mastodon:matrix.org"
}
```

Note the `@` in this case helps distinguish the type of channel. Here the protocol used is "Mastodon" rather than "ActivityPub".
While the underlying protocol might indeed be ActivityPub, the choice of name should be recognisable to users.

## `/_matrix/app/v1/thirdparty/location`

This proposal does NOT seek to deprecate the `location` API even though this spec effectively supercedes it in most respects.
A future MSC may choose to remove it, however.

## Potential issues

The proposal intentionally sidesteps the 'bridge type' problem: Codifing what a portal, plumbed and gatewayed bridge look
like in Matrix. For the time being, the event will not contain information about the type of bridge in a room but merely
information about what is is connected to.

## Alternatives

Some thoughts have been thought on using the third party bridge routes in the AS api to get bridge info,
by calling a specalised endpoint. There are many issues with this, such as the routes not working presently
over federation, as well as requring the bridge to be online. Using a state event ensures the data is scoped
per room, and can be synchronised and updated over federation.

## Future MSCs

(This section is for the beneift of readers to understand why this MSC doesn't contain X feature)

This proposal forms the basis for bridges to become more interactive with clients as first class citizens rather
than relying upon users having prior knowledge about which users are bridged users, or where a room is bridged to.

Future MSCs could expand the /publicRooms response format to show what network a room is bridged to before the
user attempts to join it. Another potential MSC could allow users to see which bridges they are connected to
via an accounts settings page, rather than relying on PMs to the bridge bot.

## Security considerations

Anybody with the correct PLs to post state events will be able to manipulate a room by sending a bridge
event into a room, even if the bridge is not present or does not exist. It goes without saying that if
you let people modify your room state, you need to trust them not to mess around. A future MSC may allow
users to "trust" some mxids as bridges, rather than relying on just PLs to convey trustworthiness.