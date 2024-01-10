# Lay v1 [DRAFT]
## July 2023

### Introduction

Lay is a protocol for internet-based communication for platforms involving text chat, forum posts, etc. The protocol follows a REST API structure on top of **HTTPS** over the **HTTP/2** protocol.
Lay can be implemented as a standard HTTPS server hosting on any port (usually 443).

### Structure

Lay implementations define interfaces through typical HTTP routes. All communication is done exclusively through GET, POST, and DELETE requests sent to these routes.
Each request uses JSON for the request data, in which the structure is specified further in this document. This means that there is no need for an open continuous connection with a server.

### Error Handling

Lay uses the following HTTP response codes:
 - OK (200) - Signaling Success (response JSON not necessary)
 - Bad Request (400) - Signaling Client Error (response JSON necessary)
 - Internal Server Error (500) - Signaling Server Error (response JSON necessary)

The server must reply with one of these HTTP codes for every client request. The server must also provide information about errors if they arise from a client's request.
The response JSON must contain a Lay error code (string, case sensitive) and a user-friendly message.

Lay has several universal error codes. Each standard Lay route also has its own error codes associated with it.

Univeral Error Codes:
 - LIMIT_REQUEST - *the request was too large*
 - INVALID_KEY - *the provided public key is not a valid Ed25519 key*
 - INVALID_SIGNATURE - *the provided signature is not a valid Ed25519 signature*
 - FAILED_VERIFY_SIGNATURE - *the provided signature does not match the provided public key*
 - IMPOSSIBLE_TIMESTAMP - *the provided timestamp is not possible*

### Client/Server Authorization

Every client and server will maintain their own public and private key used for verifying and signing respectively. Lay standardizes Ed25519 for public key cryptography.

Public keys and signatures are serialized as standard Base64 strings when transmitting via requests.
A public key should be 32 bytes, and a signature should be 64 (although the Base64 strings will likely be larger).

### Channels

Lay defines a channel as a centralized group of posted content. Channels allow servers to have separate places for clients to post/poll content to/from.
Servers can also choose to restrict channels to specific content or clients using allowlists or blocklists.

Channels are identified by their unique public key, which is used when clients’ send messages to a specific channel.
User identifiers (their client public key) can also be used as a channel ID to facilitate private messaging.
Users can poll their private messages by sending a GET request with their own public key as the channel ID.
Other users can send messages privately to another user by sending a POST request to the target user’s public key as the channel ID.

### Resources

Lay defines a resource as a non-primitive piece of data (mainly files, such as images, large text documents, executable files, etc) that can be uploaded and downloaded by clients.
Every resource has an associated resource ID, which is simply the SHA-512 hash of the bytes of the data. Servers can also store other metadata for resources.

**NOTE:** Resources in Lay are optional, and therefore it is up to the client/server implementation to support the resource system.
The reasoning behind this is that not all server maintainers may have the physical resources to store large amounts of files for resources created by users.

### Side Notes

Implementation of anything below marked with a \* before it is **optional** for *clients* but **required** for *servers*.

Implementation of anything below marked with a \*\* before it is **optional** for both *clients* and *servers*.

### /text Route

This route is the core interface for Lay. All user and server text messages are sent through this interface via POST requests. Clients also poll messages via GET requests to this route.

This route expects a specific structure to the provided JSON data.

GET requests sent to this route should contain the following data formatted in standard (preferably minified) JSON:

* Public Key (client’s public key)
* Timestamp (milliseconds since UNIX epoch)
* Channel ID (channel public key)
* \*\* ... <metadata> ...
* Signature (previous data signed by client’s private key)

The amount of historical messages returned by the server in response is implementation defined.

Server implementations can track the last time a specific client polled messages, and only poll the historical messages since that point in time.

POST requests sent to this route should contain the following data formatted in standard JSON:

* Public Key (client’s public key)
* Timestamp (milliseconds since UNIX epoch)
* Channel ID (public key)
* Content (UTF-8 text)
* \*\* ... <metadata> ...
* Signature (previous data signed by client’s private key)

For example, the server can authorize the message based on the sender's public key or IP address, for example, to implement allowlist or a ban mechanism.

The timestamp should be used for ordering messages on both the client (for displaying messages in order) and server (for determining which messages to poll for a client).

**NOTE:** The timestamp can not be meaningfully verified, but a tampered or impossible-timestamp should not cause any security or implementation issues.
For example, server implementations can also track their own timestamp of “when the message was received server side”, but should use client timestamps for ordering.

### /profile Route

This route is used for the standard profile system in Lay. A profile is defined by one main characteristic with a few optional ones:

* Display name (UTF-8)
* \*\* ... <metadata> ...

The profile interface is used by clients to retrieve profile information for other connected clients and channels, as well as for clients to send their profiles to the server to store for other clients to retrieve.

GET requests sent to this route should contain the following data formatted in standard JSON:

* Public Key (client’s public key)
* Timestamp (milliseconds since UNIX epoch)
* Target Public Key (public key of desired client, channel, or server profile)
* Signature (previous data signed by client’s private key)

The server should reply with the profile data in JSON described in sections above, as well as any optional server metadata.

POST requests sent to this route should contain the following data formatted in standard JSON:

* Public Key (client’s public key)
* Timestamp (milliseconds since UNIX epoch)
* Profile (client profile data)
* Signature (previous data signed by client’s private key)

The server should reply with success if the profile was successfully authorized and stored.

### /channel Route

This route is used for clients to discover available channels on the server.

GET requests sent to this route should contain the following data formatted in standard JSON:

* Public Key (client’s public key)
* Timestamp (milliseconds since UNIX epoch)
* Signature (previous data signed by client’s private key)

The server should reply with a json array of channel public keys available to the client.
The server can also use this to allow for user discovery by returning user public keys as well, where the client can get the profile and distinguish channels and users.

### /resource Route

This route is used for clients to send and retrieve resources (such as images or files).

GET requests sent to this route should contain the following data formatted in standard JSON:

* Public Key (client’s public key)
* Timestamp (milliseconds since UNIX epoch)
* Resource ID (resource data hash)
* Signature (previous data signed by client’s private key)

The server should reply with json containing any metadata the server wants to provide the client.
The server should also respond via HTTP multipart with the entire byte contents of the resource.

POST requests sent to this route should contain the following data formatted in standard JSON:

* Public Key (client’s public key)
* Timestamp (milliseconds since UNIX epoch)
* Resource ID
* \*Metadata
* Signature (previous data signed by client’s private key)

POST requests should use HTTP multipart to send the entire byte contents.

### \*\*Guest Clients

Servers can optionally define a guest profile, that is, a profile that all anonymous clients’ share as to provide other clients’ with an indicator that the client is choosing to remain anonymous. This can be the profile that the server returns if a client requested the profile for a specific public key that does not exist.

Servers should preferably NOT use/expose clients’ IP addresses as identifiers or display names for clients without a profile.
