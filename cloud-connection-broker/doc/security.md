# Security

The broker does not use TLS and need to authenticate connecting clients in other ways.
It does so by using a *Authenticated Encryption with Associated Data* (AEAD) algorithm via a PSK (Pre-shared key)

Subscriber-Publisher messages are opaque to the broker and use their own PSK (end-to-end encryption).

**Privacy:** The metadata during connection establishment is encrypted with rotating keys.
A target user-id is never send in plaintext and cannot be identified and due to the rotating nature of the keys,
it is also not possible to perform long-time analysis on encrypted user-ids. 

## PSK rotation and PSK obtaining procedure

PSKs (Pre-shared keys) are generally rotated every hour.
They are derived off a 16-byte [TOTP](https://en.wikipedia.org/wiki/Time-based_One-time_Password_algorithm)
(Time-based One-Time Password), generated by an OAuth Server.
TOTPs are generated by using the private key of the OAuth server as secret.

The PSK obtaining flow looks like this in general:

1. A participant (broker or client) authenticates on an OAuth server and obtains a
   JWT based refresh and access token. The access token JWT itself is valid for one hour.
2. It extracts the "enc_key" private claim of the JWT.
   That key is valid for one hour, starting with the first minute of an hour.
   It therefore does not overlap the JWTs lifetime.
   Because the TOTP changes every hour and to understand streams that were encrypted via
   a still valid access token of the last hour, the "enc_key" private claim actually contains
   a tuple of two TOTPs: (C,L) One for the **C**urrent, one for the **L**ast hour.
3. - **Encryption**: A participant selects key **C** if the current wall time
   (for example "12:17") hour ("12") is the same as the issuing times (eg "12:01") hour ("12").
   Otherwise **L** is used. The issuing hour ("12") is send as "key_id" together with the ciphertext.
   - **Decryption**: A receptionist selects key **C** if the received "key_id" equals the wall times hour.
   It selects **L** otherwise. It decrypts the ciphertext with the selected key.

It is slightly different for client-broker connection establishment encryption
and publisher-subscriber end-to-end encryption:

* **Client-Broker Connection Establishment**:
  The broker must be authenticated to an OAuth server and will store the refresh / access token.
  That access token is expected to be a JWT and MUST include a private claim "cloud_key" which is used as PSK.
  Publisher, subscriber clients must have acquired a JWT via the OAuth scope "brokerkey" to get access to this
  "cloud_key" private claim as well.
* **End-to-end encryption** works across subscribers and publishers that are in possession of a JWT access token
  issued for the same user account. Such a JWT is expected to contain a private claim "enc_key" of 16 Bytes and a private claim "uid" containing
  the user id. The PSK is the composition of those two byte sequences.
 

## Encryption parameters

Some of the encryption parameters must accommodate the fact that especially publisher connections are established
from stateless cloud functions. The only state they have is the user bound JWT based access token and compiled
in data like public key parts of certificates.

#### Salt

The encryption *Salt* for publisher-to-broker and subscriber-to-broker connections
is the first 8 bytes of the current public OAuth key (see the create-secrets crate in this repository).
The *Salt* therefore doesn't change during the lifetime of the OHX OAuth certificate.

The encryption *Salt* for the end-to-end stream is the last 8 bytes of the user id (uid).

#### Nonce

The *Nonce* is determined by a connecting client, not by the server, to avoid roundtrips.
Because those, especially the publishers, are expected to be stateless cloud functions,
the *Nonce* can only be derived off the systems random generator or the current time.
The *Nonce* is continuously increased for each message of an established connection though.

## Encryption Weaknesses and Vulnerabilities

The first 6 bytes of a *Nonce* are random bytes.
The last two bytes encode seconds and 1/10 subseconds of the unix timestamp (UTC).
This guarantees different *Nonce*s per each hour.
This depends on the system random generator and requests per second. Connection requests per second are throttled.

ChaCha20 and Poly1305 themselves are considered safe at the moment (2019).
The implementation is based on Ring, which is as well a trusted and audited Rust library.

Weaknesses and vulnerabilities are therefore entirely depending on how the PSKs are determined
and if breaking or leaking a PSK will lead to the ability to decrypt the entire publisher-subscriber conversation.

Current TOTP based keys are valid one hour only. 

## Attack surface

The broker accepts new TCP connections on one port only. A fixed data length is expected for the first message.
A connection is immediately dropped if not authenticated successfully in the first message.

A DDOS TCP syn attack only affects the OS kernel and the broker can accept new connections again as soon
as the kernels queue of awaiting half-open TCP connections has timed out. 

A DDOS TCP connection establish attack (without sending data) will quickly fill the accept channel of the broker (100 slots).
Each connection request has a timeout of 1 second and connections are processed 5 at a time.
The OS kernels queue for incoming, unprocessed TCP connections will saturate at some point.

As soon as the DDOS stops, the broker will be able to operate normally, eg accept new connections, after about 20 seconds
plus the time to drain and process the kernels queued incoming requests.