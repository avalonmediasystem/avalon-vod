# Token Authentication for Avalon Streams #

## Flow ##

Given the streaming URL:

		rtmp://example.edu/avalon/mp4:a/b/c.mp4?token=d

Engage Player will perform the following operations:

1. Connect to `rtmp://example.edu/avalon?token=d`
2. Play stream `mp4:a/b/c.mp4`

Adobe Media Server's `onConnect()` event handles operation (1), and can permit or refuse the connection. It can also change the client's read access to allow only certain content to be played.


## Issues ##

* There is no ActionScript event handler for operation (2), so the auth decision needs to be made during operation (1).
* The `onConnect()` event only receives the application URI, not the full stream location.

## Implementation ##

1. Avalon generates a `stream_token` (d) with the format `#{stream.mediapackage_id}-#{auth_token}`.
2. In `onConnect()`, Adobe Media Server calls back to Avalon with `/authorize?token=d`.
3. Avalon splits the token on the last hyphen, and ensures that `auth_token` is valid for `mediapackage_id`.
4. If the token is valid, return `mediapackage_id` to AMS with status `202 Accepted`.
5. Otherwise, return an empty document with status `403 Unauthorized`
6. AMS accepts the connection, but with client access limited to only the media package specified by the response. With the example given, a 202 response results in a connection that can _only_ stream content starting with `/a/*`, while a 403 results in a connection that has no access to anything.

## Pros ##

* Tokens can be tied to user sessions/expirable, or generated on the fly and good forever (e.g., for embedding).
* Tokens which abused can be revoked or expired.
* The same `auth_token` can be used to secure multiple streams, but a given connection will only have access to the specific stream that generated the `stream_token`.

## Cons ##

* `stream_token`s are still exploitable during their lifetimes, though in a very time- and content-limited fashion.