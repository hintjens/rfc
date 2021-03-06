---
domain: rfc.zeromq.org
shortname: 26/CURVEZMQ
name: CurveZMQ
status: stable
editor: Pieter Hintjens <ph@imatix.com>
---

This document describes CurveZMQ, a protocol for secure messaging across the Internet. CurveZMQ is closely based on Daniel J. Bernstein's [CurveCP](http://curvecp.org), adapted for use in ZeroMQ over TCP. A reference implementation of CurveZMQ is provided at [curvezmq.org](http://curvezmq.org). This document describes version 1.0 of CurveZMQ.

See also: http://rfc.zeromq.org/spec:23/ZMTP, http://rfc.zeromq.org/spec:25/ZMTP-CURVE.

## Preamble

Copyright (c) 2013 iMatix Corporation.

This Specification is free software; you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation; either version 3 of the License, or (at your option) any later version. This Specification is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details. You should have received a copy of the GNU General Public License along with this program; if not, see <http://www.gnu.org/licenses>.

This Specification is a [free and open standard](http://www.digistan.org/open-standard:definition) and is governed by the Digital Standards Organization's [Consensus-Oriented Specification System](http://www.digistan.org/spec:1/COSS).

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](http://tools.ietf.org/html/rfc2119).

## CurveZMQ Specification

CurveZMQ is a protocol for secure messaging across the Internet that closely follows the [CurveCP security handshake](http://curvecp.org). We built CurveZMQ primarily to provide security for ZeroMQ applications but it can be used more widely. The CurveZMQ protocol can work either on top of the <tt>libzmq</tt> API, at the application layer, or inside <tt>libzmq</tt>, as a security mechanism for a [ZMTP 3.0 transport layer](http://rfc.zeromq.org/spec:23).

### Goals

CurveZMQ aims to provide the same level of security as CurveCP, differences between UDP and TCP notwithstanding. That is, it aims to prevent eavesdropping, fraudulent data, altered data, replay attacks, amplification attacks, man-in-the-middle attacks, key theft attacks, identity attacks, and certain denial-of-service attacks.

Additionally, it aims to:

* Provide interoperability between implementations written in arbitrary languages, by defining a specific command syntax and semantics that guarantee a command sent by one implementation can be correctly processed by another implementation.

* Be usable as a plug-in mechanism at different levels in the stack, rather than serving only as transport-layer security. This means CurveZMQ can be used with any transport, including older versions of ZMTP.

* Be compatible with ZMTP 3.0's extensible security model, which is based on the IETF's [Simple Authentication and Security Layer](http://tools.ietf.org/html/rfc4422) (SASL).

* Be simple to implement, given NaCl or libsodium support in a given language.

### Security

CurveZMQ uses the Curve25519 elliptic curve, which was designed by Daniel J. Bernstein to achieve good performance with short key sizes (256 bits). The protocol establishes short-term session keys for every connection to achieve perfect forward security. Session keys are held in memory and destroyed when the connection is closed. CurveZMQ also addresses replay attacks, amplification attacks, MIM attacks, key thefts, client identification, and various denial-of-service attacks. These are inherited from CurveCP, and are explained later.

### Use Cases

There are two main use cases for CurveZMQ:

* To secure a single hop between client and server, which is the CurveCP use case. For this use case we would embed CurveZMQ in the transport layer so that it can work for all patterns (publish-subscribe, pipeline, and so on).

* To secure a client and server end-to-end across one or more untrusted peers, where transport-layer security is not sufficient. For this use case we would build CurveZMQ into our application-layer protocols using an asynchronous request-reply pattern.

Additionally, CurveZMQ could be used for data sent from one peer to another over out-of-band transports, such as email or file transfer, assuming both peers first execute the initial handshake over a suitable transport.

### Overall Operation of CurveZMQ

Clients and servers have long-term permanent keys, and for each connection, they create and securely exchange short-term transient keys. Each key is a public/secret keypair, following the elliptic curve security model.

To start a secure connection the client needs the server permanent public key. It then generates a transient key pair and sends a HELLO command to the server that contains its short term public key. The HELLO command is worthless to an attacker; it doesn't identify the client.

The server, when it gets a HELLO, generates its own short term key pair (one connection uses four keys in total), and encodes this new private key in a "cookie", which it sends back to the client as a WELCOME command. It also sends its short term public key, encrypted so only the client can read it. It then discards this short term key pair.

At this stage, the server hasn't stored any state for the client. It's generated a keypair, sent that back to the client in a way only the client can read, and thrown it away.

The client comes back with an INITIATE command that provides the server with its cookie back, and the client permanent public key, encrypted as a "vouch" so only the server can read it. As far as the client is concerned, the server is now authenticated, so it can also send back metadata in the command.

The server reads the INITIATE and can now authenticate the client permanent public key. It also unpacks the cookie and gets its short term key pair for the connection. As far as the server is now concerned, the client is now authenticated, so the server can send its metadata safely. Both sides can then send messages and commands.

This handshake provides a number of protections but mainly, *perfect forward security* (you cannot crack data encrypted using the short term keys even if you record it, and then later get access to the permanent keys) and *protection of client identity* (the client permanent public key is not sent in clear-text).

### General Design

The CurveZMQ protocol has a synchronous handshake, followed by an asynchronous exchange of messages in either direction. Each operation in the protocol is a "command", which consists of a command name followed by a command body. Command names are 8 octets, padded with spaces, while command bodies have a binary encoding that depends on the command.

CurveZMQ has these commands: HELLO, WELCOME, INITIATE, READY, MESSAGE, and ERROR. A connection always starts with the client sending HELLO to the server, which responds with WELCOME:

```
C:HELLO
S:WELCOME
```

After this exchange each side has the other's public transient key. The client then sends an INITIATE command, and the server responds with READY:

```
C:INITIATE
S:READY
```

After this exchange each side has authenticated the other and exchanged metadata for the connection. They can now send messages as MESSAGE commands, in any order and without any further synchronization:

```
C:MESSAGE | S:MESSAGE
```

Either peer can silently close their side of the connection at any time, and this can also happen due to network behavior. This is considered a soft error, and the disconnected peer MAY retry a connection after some suitable interval.

To reject a client connection explicitly, a server SHALL send an ERROR command:

```
S:ERROR
```

This is considered a hard error, and the client SHALL NOT try to reconnect using the same credentials. A client peer SHALL NOT send ERROR to the server.

CurveZMQ is designed to be implemented a black box that accepts and produces correctly-formatted commands, and outputs data messages on the side, and runs a state machine internally for each connection.

In a CurveZMQ architecture, the clients MUST know the server public key before they can connect. Servers MAY know clients' public keys, and MAY distinguish different clients based on their keys. This gives us three possible security models:

* Where the server does not check client keys at all. In this case the clients can be certain they are talking securely to the correct server, but the server will accept connections from any client. This fits the conventional Internet model where a browser talks securely to a website to place and order and send credit card information.

* Where all clients share the same public key, that the server checks. In this case access to the server will be restricted to authorized clients. This fits the model of a private network over public infrastructure. Note that the client public key can be stolen, but cannot be used unless an attacker also steals the client secret key.

* Where each client has its own key, that the server checks. In this case the server can grant access to clients according to their authenticated identity. Again, an attacker may steal the client public key but cannot do anything with this unless it can also steal the client's secret key.

Elliptic curve encryption depends on mixing a unique number into each encryption operation. This "number used once", or "nonce", is chosen by the sender, stops replay attacks, and protects the keys from cryptoanalysis. We use two kinds of nonces in CurveZMQ. A long nonce protects permanent keys, and is 16 octets from a good random number generator. A short nonce protects transient keys and is an 8-octet sequential number.

### High-level Grammar

The following ABNF grammar defines the CurveZMQ protocol from a high level:

```
curvezmq = C:hello ( S:welcome | S:error )
           C:initiate ( S:ready | S:error )
           *message

;   HELLO command, 200 octets
hello = %d5 "HELLO" version padding hello-client hello-nonce hello-box
hello-version = %x1 %x0     ; CurveZMQ major-minor version
hello-padding = 72%x00      ; Anti-amplification padding
hello-client = 32OCTET      ; Client public transient key C'
hello-nonce = 8OCTET        ; Short nonce, prefixed by "CurveZMQHELLO---"
hello-box = 80OCTET         ; Signature, Box [64 * %x0](C'->S)

;   WELCOME command, 168 octets
welcome = %d7 "WELCOME" welcome-nonce welcome-box
welcome-nonce = 16OCTET     ; Long nonce, prefixed by "WELCOME-"
welcome-box = 144OCTET      ; Box [S' + cookie](S->C')
;   This is the text sent encrypted in the box
cookie = cookie-nonce cookie-box
cookie-nonce = 16OCTET      ; Long nonce, prefixed by "COOKIE--"
cookie-box = 80OCTET        ; Box [C' + s'](K)

;   INITIATE command, 257+ octets
initiate = %d8 "INITIATE" cookie initiate-nonce initiate-box
initiate-cookie = cookie    ; Server-provided cookie
initiate-nonce = 8OCTET     ; Short nonce, prefixed by "CurveZMQINITIATE"
initiate-box = 144*OCTET    ; Box [C + vouch + metadata](C'->S')
;   This is the text sent encrypted in the box
vouch = vouch-nonce vouch-box
vouch-nonce = 16OCTET       ; Long nonce, prefixed by "VOUCH---"
vouch-box = 80OCTET         ; Box [C',S](C->S')
metadata = *property
property = name value
name = OCTET *name-char
name-char = ALPHA | DIGIT | "-" | "_" | "." | "+"
value = value-size value-data
value-size = 4OCTET         ; Size in network order
value-data = *OCTET         ; 0 or more octets

;   READY command, 30+ octets
ready = %d5 "READY" ready-nonce ready-box
ready-nonce = 8OCTET        ; Short nonce, prefixed by "CurveZMQREADY---"
ready-box = 16*OCTET        ; Box [metadata](S'->C')

;   ERROR command, 7+ octets
error = %d5 "ERROR" error-reason
error-reason = OCTET 0*255VCHAR

;   MESSAGE command, 33+ octets
message = %d7 "MESSAGE" message_nonce message-box
message-nonce = 8OCTET      ; Short nonce, prefixed by "CurveZMQMESSAGE-"
message-box = 17*OCTET      ; Box [payload](S'->C') or (C'->S')
;   This is the text sent encrypted in the box
payload = payload-flags payload-data
payload-flags = OCTET       ; Explained below
payload-data = *octet       ; 0 or more octets
```

The command size is not part of the command encoding but is assumed to be a property of the command (in other words, known to the implementation when it encodes or decodes a command).

### CurveZMQ Commands

We explain how each command is encrypted, the contents of each command, and the semantics of each command. Note that we work with four pairs of keys:

* The client and server each have a permanent key pair (public and secret keys), called C and S.
* The client and server each generate a transient key pair for the connection, called C' and S'.

We use the CurveCP notation "Box [X](C->S)" to mean a cryptographic box that encrypts X "from C to S", meaning only C can create the box and only S can open it. A box is a one-way transfer of information from C to S where both sender and recipient can be certain the transfer is secret, secure, and authentic (if it arrives, which is not guaranteed). When S opens the box, it knows that C created it. The actual steps for creating and opening a box are:

* Create the box using S's public key, and C's secret key and a well-chosen nonce.
* Open the box using C's public key, and S's secret key, and the same nonce.

CurveZMQ uses the same cryptography as CurveCP, so keys are 32 octets (256 bits) and nonces are 24 octets. An encrypted box is always 16 octets larger than the clear-text data it holds. These sizes are not configurable; they are enforced by the underlying cryptography library and act as universal constants for CurveZMQ implementations. This may change in future versions.

#### The HELLO Command

The first command on a CurveZMQ connection is the HELLO command. The client SHALL send a HELLO command after opening the stream connection. This command SHALL be 200 octets long and have the following fields:

* The command ID, which is [5]"HELLO".

* The CurveZMQ version number, which SHALL be the two octets 1 and 0.

* An anti-amplification padding field. This SHALL be 70 octets, all zero. This filler field ensures the HELLO command is larger than the WELCOME command, so an attacker who spoofs a sender IP address cannot use the server to overwhelm that innocent 3rd party with response data.

* The client's public transient key C' (32 octets). The client SHALL generate a unique key pair for each connection it creates to a server. It SHALL discard this key pair when it closes the connection, and it MUST NOT store its secret key in permanent storage, nor share it in any way.

* A client short nonce (8 octets). The nonce SHALL be implicitly prefixed with the 16 characters @@"CurveZMQHELLO---"@@ to form the 24-octet nonce used to encrypt and decrypt the signature box.

* The signature box (80 octets). This SHALL contain 64 zero octets, encrypted from the client's transient key C' to the server's permanent key S.

The server SHALL validate all fields and SHALL reject and disconnect clients who send malformed HELLO commands.

When the server gets a valid HELLO command, it SHALL generate a new transient key pair, and encode both the public and secret key in a WELCOME command, as explained below. The server SHALL not keep this transient key pair and SHOULD keep minimal state for the client until the client responds with a valid INITIATE command. This protects against denial-of-service attacks where unauthenticated clients send many HELLO commands to consume server resources.

Note that the client uses an 8 octet "short nonce" in the HELLO, INITIATE, and MESSAGE commands. This nonce SHALL be an incrementing integer, and unique to each command within a connection. The client SHALL NOT send more than 2^64-1 commands in one connection. The server SHALL verify that a client connection does use correctly incrementing short nonces, and SHALL disconnect clients that reuse a short nonce.

#### The WELCOME Command

The server SHALL respond to a valid HELLO command with a WELCOME command. This command SHALL be 168 octets long and have the following fields:

* The command ID, which is [7]"WELCOME".

* A server long nonce (16 octets). This nonce SHALL be implicitly prefixed by the 8 characters "WELCOME-" to form a 24-octet nonce used to encrypt and decrypt the welcome box.

* A welcome box (144 octets) that encrypts the server public transient key S' (32 octets) and the server cookie (96 octets), from the server permanent key S to the client's transient key C'.

Note that the server uses a 16-octet "long nonce" in the WELCOME command and when creating a cookie. This nonce SHALL be unique for this server permanent key. The recommended simplest strategy is to use 16 random octets from a sufficiently good entropy source.

The cookie consists of two fields:

* A server long nonce (16 octets). This nonce SHALL be implicitly prefixed by the 8 characters @@"COOKIE--"@@ to form a 24-octet nonce used to encrypt and decrypt the cookie box.

* A cookie box (80 octets) that holds the client public transient key C' (32 octets) and the server secret transient key s' (32 octets), encrypted to and from a secret short-term "cookie key". The server MUST discard from its memory the cookie key after a short interval, for example 60 seconds, or as soon as the client sends a valid INITIATE command.

The server SHALL generate a new cookie for each WELCOME command it sends.

The client SHALL validate all fields and SHALL disconnect from servers that send malformed WELCOME commands.

#### The INITIATE Command

When the client receives a WELCOME command it can decrypt this to receive the server's transient key S', and the cookie, which it must send back to the server. The cookie is the only memory of the server's secret transient key s'.

The client SHALL respond to a valid WELCOME with an INITIATE command. This command SHALL be at least 257 octets long and have the following fields:

* The command ID, which is [8]"INITIATE".

* The cookie provided by the server in the WELCOME command (96 octets).

* A client short nonce (8 octets). The nonce SHALL be implicitly prefixed with the 16 characters "CurveZMQINITIATE" to form the 24-octet nonce used to encrypt and decrypt the vouch box.

* The initiate box (144 or more octets), by which the client securely sends its permanent public key C to the server. The initiate box holds the client permanent public key C (32 octets), the vouch (96 octets), and the metadata (0 or more octets), encrypted from the client's transient key C' to the server's transient key S'.

The vouch itself consists of two fields:

* A client long nonce (16 octets). This nonce SHALL be implicitly prefixed with the 8 characters @@"VOUCH---"@@ to give a 24-octet nonce, used to encrypt and decrypt the vouch box. This nonce SHALL be unique for all INITIATE commands from this client permanent key. A valid strategy is to use 16 random octets from a sufficiently good entropy source.

* The vouch box (80 octets), that encrypts the client's transient key C' (32 octets) and the server permanent key S (32 octets) from the client permanent key C to the server transient key S'.

The metadata consists of a list of properties consisting of name and value as size-specified strings. The name SHALL be 1 to 255 characters. Zero-sized names are not valid. The case (upper or lower) of names SHALL NOT be significant. The value SHALL be 0 to 2^31-1 octets of opaque binary data. Zero-sized values are allowed. The semantics of the value depend on the property. The value size field SHALL be four octets, in network order. Note that this size field will mostly not be aligned in memory.

The server SHALL validate all fields and SHALL reject and disconnect clients who send malformed INITIATE commands.

After decrypting the INITIATE command, the server MAY authenticate the client based on its permanent public key C. If the client does not pass authentication, the server SHALL not respond except by closing the connection. If the client passes authentication the server SHALL send a READY command and MAY then immediately send MESSAGE commands.

#### The READY Command

The server SHALL respond to a valid INITIATE command with a READY command. This command SHALL be at least 30 octets long and have the following fields:

* The command ID, which is [5]"READY".

* A server short nonce (8 octets). The nonce SHALL be implicitly prefixed with the 16 characters @@"CurveZMQREADY---"@@ to form the 24-octet nonce used to encrypt and decrypt the ready box.

* The ready box (16 or more octets). This shall contain metadata of the same format as sent in the INITIATE command, encrypted from the server's transient key S' to the client's transient key C'.

The client SHALL validate all fields and SHALL disconnect from servers who send malformed READY commands.

The client MAY validate the meta-data. If the client accepts the meta-data, it SHALL then expect MESSAGE commands from the server.

Note that the server uses an 8 octet "short nonce" in the HELLO, INITIATE, and MESSAGE commands. This nonce SHALL be an incrementing integer, and unique to each command within a connection. The server SHALL NOT send more than 2^64-1 commands in one connection. The client SHALL verify that a server connection does use correctly incrementing short nonces, and SHALL disconnect from servers that reuse a short nonce.

#### The MESSAGE Command

The client MAY send MESSAGE commands when it has received a valid READY command. The server MAY send MESSAGE commands when it has sent a READY command. This command SHALL be at least 33 octets long and have the following fields:

* The command ID, which is [7]"MESSAGE".

* A short nonce (8 octets). The nonce SHALL be implicitly prefixed with the 16 characters "CurveZMQMESSAGES" (from server) or "CurveZMQMESSAGEC" (from client) to form the 24-octet nonce used to encrypt and decrypt the message box.

* The message box (17 or more octets). This shall contain the message data, encrypted from the sender's transient key the recipient's transient key.

The message payload consists of a flags field (1 octet) followed by zero or more octets of payload data. The flags field consists of a single octet containing various control flags. Bit 0 is the least significant bit (rightmost bit):

* Bits 7-1: *Reserved*. These bits are reserved for future use and MUST be zero.

* Bit 0 (MORE): *More messages to follow*. A value of 1 indicates that more messages will follow. A value of 0 indicates that there are no more messages to follow.

The recipient SHALL validate all fields and SHALL reject and disconnect from peers who send malformed MESSAGE commands. A server SHALL NOT send an ERROR command in response to an invalid MESSAGE command.

#### The ERROR Command

A server SHALL indicate an authentication failure by sending an ERROR command back to the client. The server MAY respond with ERROR in other failure cases, or may disconnect the client silently depending on the case. This command SHALL be 7 or more octets long and have the following fields:

* The command ID, which is [5]"ERROR".

* The error reason, which is a length-specified string of 0 to 255 ASCII characters.

### Differences from CurveCP

While CurveCP defines a security handshake and traffic control for messages sent over UDP, CurveZMQ defines just a security handshake for messages sent over a connected stream protocol such as TCP, IPC, SCTP, or similar. We have made these significant changes from CurveCP:

* The INITIATE command vouch box is Box[C',S](C->S') instead of Box[C'](C->S), as recommended by [Codes in Chaos](https://codesinchaos.wordpress.com/2012/09/09/curvecp-1/) to reduce the risk of client impersonation.

* We use the term "command" instead of "packet" to represent an atomic block of data sent over the network.

* The HELLO command has a protocol version number (CurveCP is not versioned).

* We rename Cookie to WELCOME to be consistent with other mechanisms used in ZMTP 3.0.

* In the INITIATE command, we do not send message data nor a hostname, but we do send connection metadata which may include the hostname, and other properties.

* We add a READY command to send connection metadata from server to client, after security has been established.

* In INITIATE and MESSAGE commands from the client, we do not send the client public transient key since it is redundant in a connected protocol.

* In MESSAGE commands, the payload can be of any size from zero to 2^63-1 octets, whereas CurveCP limits packets to fit in UDP frames.

* In MESSAGE commands, the payload has a frame format that supports some requirements of ZMTP 3.0, particularly the continuation flag ("more").

* In MESSAGE commands, we do not use the CurveCP traffic control fields since traffic control is handled by the underlying stream protocol.

* CurveCP has no explicit ERROR response and does not inform clients of authentication failures.

* While CurveCP delivers a stream (reconstituted from packets), CurveZMQ delivers atomic messages.

## CurveCP

CurveCP is a replacement for TCP that creates a secure connection between a server and client using UDP messages. The security in CurveCP is designed to withstand a specific set of attacks, which we will explain. The cryptographic core in CurveCP is the Networking and Cryptography library (NaCl), more widely used as [libsodium](https://github.com/jedisct1/libsodium). NaCl is significant for its focus on speed, strength, and simplicity.

CurveCP was announced at the 27th Chaos Communication Congress on 28 December 2010 and CurveCP implementations are still considered experimental.

### General Design

CurveCP has two main functions: one, to deliver a reliable stream of octets from sender to receiver, and two, to ensure that transmitted data is confidential, cannot be tampered with, or forged. We will look only at the second part, which consists of a security handshake followed by an encryption and authentication mechanism.

The protocol defines a distinct "client" and "server" role that reflects the asymmetric nature of most architectures. For example, most attacks are focused on servers, not clients, and thus CurveCP's defenses are more focused on protecting servers than clients. CurveZMQ has inherited this perspective but we may shift to more symmetric defenses in the future.

CurveCP uses elliptic-curve cryptography to encrypt and authenticate each packet. It puts data into a *cryptographic box* that is encrypted and authenticated from a sender's secret key to a receiver's public key. To create a box and successfully open it, NaCl uses five elements:

* The sender's and receiver's *public keys*, which we assume are exchanged beforehand in some secure fashion. Public keys are not confidential, but any exchange mechanism itself must be secure against fraud, else an attacker can substitute their own keys and perform a man-in-the-middle attack.

* The two corresponding *secret keys*, which are never exchanged in any form.

* A public *nonce*, or "number used once", chosen by the sender. Nonces stop replay attacks and CurveCP pays a lot of attention on to how to choose nonces. Overall we either use random octets, or incremental integers, depending on the case.

NaCl (and this document) uses the syntax "Box [X](S->R)" to define a cryptographic box encrypting and authenticating data X from the sender S to the receiver R. To create a box we need the nonce, secret key for S, and public key for R. To open a box we need the nonce, secret key for R, and public key for S. When we successfully open a box, we know it was sent by S and that the data was not tampered with. If we cannot open a box, it was either tampered with, or created fraudulently.

Keys in CurveCP are 32 octets (256 bits) and nonces are 24 octets. These are the default sizes proposed by NaCl. These sizes are hard-coded into the CurveCP packet formats, and CurveZMQ uses the same sizes. There is no option for users to choose weaker security. However we would expect future versions of CurveZMQ to use longer key sizes.

Each peer (client or server) has a *permanent* key which is known to the other party beforehand. CurveCP uses the permanent keys at the start of a connection to provide authentication and to securely exchange *transient* transient keys (which CurveCP calls "short-term"). The permanent keys are never used to encrypt data. The transient keys are unique to the connection, and both peers discard these when the connection is finished. Thus, even if an attacker records encrypted boxes, and later seizes the permanent keys, he cannot open those boxes (so-called "perfect forward secrecy").

Each "key" is in fact a pair of keys, one public and one secret. Data is encrypted with the public key (and other information) and decrypted with the secret key. The public key may be known to other peers (shared across some secure medium), while the secret keys are never known to other peers.

To establish a connection, the client must know the server's permanent public key, while the server may know a set of client permanent public keys. During the handshake, the client sends its permanent public key to the server, protected by the transient keys. The server may thus authenticate the client, while the client's permanent public key cannot be extracted from the traffic by an attacker.

The packet flow in CurveCP is as follows:

* The client generates a transient key pair and sends the public key to the server as a **Hello** packet. This packet has a signature (Box [zeros](C'->S)) that prevents tampering.

* The server generates a new, random key pair and send this to the client as a **Cookie** packet, with the public key in clear, and the secret key in a box (the "cookie") that only the server can open, and at most for two minutes.

* The client sends the cookie back to the server, along with its permanent public key to the server as an **Initiate** packet. At this point the server may authenticate the client's identity and establish the connection, opening the cookie to get the transient keys for the connection.

* Client and server can now send **Message** packets that hold data, in boxes locked with the transient keys. CurveCP uses a fixed maximum message size designed to fit into UDP packets.

### Specific Defenses

We list the specific attacks that CurveCP's security design aims to prevent:

* *Eavesdropping,* in which an attacker sitting between the client and server monitors the data. We send all in boxes that only the authentic recipient, knowing the necessary secret key, can open.

* *Fraudulent data,* in which an attacker creates packets that claim to come from the server or client. Only the authentic sender, knowing the necessary secret key, can create a valid box, and thus a valid packet.

* *Altering data,* in which an attacker sitting between client and server manipulates packets in specific ways. If a packet is modified in any way, the box it contains will not open, so the recipient knows there has been an attack and discards the packet.

* *Replaying data,* in which an attacker captures packets and re-sends them later to imitate a valid peer. We encrypt every box with a unique nonce. An attacker cannot replay any packet except a Hello packet, which has no impact on the server or client. Any other replayed packet will be discarded by its recipient.

* *Amplification attacks,* in which an attacker makes many small unauthenticated requests but uses a fake origin address so large responses are sent to that innocent party, overwhelming it. Only a Hello packets are unauthenticated, and they are padded to be larger than Cookie packets.

* *Man-in-the-middle attacks,* in which an attacker sitting between client and server substitutes its own keys so it can play "server" to the client and play "client" to the server, and thus be able to open all boxes. Since permanent keys are known beforehand, an attacker cannot successfully imitate a server, nor a client if the server does client authentication.

* *Key theft attacks,* in which an attacker records encrypted data, then seizes the secret keys at some later stage, perhaps by physically attacking the client or server computer. Client and server use transient keys that they throw away when they close the connection.

* *Identifying the client,* in which an attacker traces the identity of a client by extracting its permanent public key from packets. The client sends this only in an Initiate packet, protected by the transient keys.

* *Denial-of-Service attacks,* in which an attacker forces the server to perform expensive operations before authentication, thus exhausting the server. CurveCP uses high-speed high-security elliptic-curve cryptography so that a typical CPU can perform public-key operations faster than a typical Internet connection can ask for those operations. The server does not allocate memory until a client sends the Initiate packet.

### Authentication

A CurveCP client must know the server permanent public key in order to start a connection. A server may differentiate the clients through their permanent public keys. Servers may thus accept unauthenticated clients, but clients will always authenticate the server. CurveCP uses the permanent keys to securely vouch for transient keys so that every box can be authenticated, but cannot be opened after the connection has closed (and each peer discards its transient keys).

CurveCP does not use client IP addresses as part of its security model. Firstly, because IP addresses are not secure, and secondly this means a CurveCP connection is independent of the IP endpoints (so can migrate across network changes).

### Simplicity-Oriented Design

CurveCP (like NaCl) expresses the view that security should not come with options, as these lead to bad choices and vulnerable systems. Thus it has no options to disable encryption or authentication. As the curvecp.org site says, "*CurveCP's server authentication is always active and cannot be disabled. CurveCP's client authentication is always active and cannot be disabled. CurveCP's encryption is always active and cannot be disabled. CurveCP's forward secrecy is always active and cannot be disabled. CurveCP has nothing analogous to IPsec's separation between AH and ESP, and nothing analogous to HTTPS renegotiation. The lack of options in CurveCP simplifies the protocol and prevents a wide range of design and implementation mistakes.*"

### Known Issues with Using CurveCP

To implement or use CurveCP you need to be aware of some issues and limitations with the design:

**CurveCP makes no attempt to defend against traffic analysis attacks.** When an application sends packets that correspond to real data (for example a password being typed, a voice-over-IP call, or a compressed video stream), an attacker can extract useful data from the timing, addressing, and size of encrypted packets. To protect against traffic analysis attacks, we could pad packets with random data, send bogus data randomly, and blur the difference between heartbeat xmessages and data messages.

**CurveCP does not explain how keys are exchanged.** To some extent, CurveCP is so simple and robust because it completely ignores this problem. However any realistic implementation must solve this problem. At the very least, clients can be configured with the server public key, and generate their own keys randomly. This lets clients be sure they are talking securely to the correct server. The server however will not be able to authenticate clients without knowing their permanent public keys in advance.

**CurveCP enforces a 1-to-many relationship from servers to clients.** One change we might propose to CurveCP would be the addition of the server public transient key in Message packets from the server, so that clients could work with many servers (a common pattern in ZeroMQ applications).

**CurveCP depends on NaCl.** The protocol is bound to the NaCl crypto library that is today the primary implementation of the necessary algorithms. NaCl is a young library and not as widely available, stable, or fast as it might be. This affects CurveZMQ equally.

**CurveCP is a full protocol stack.** This is perhaps the most serious challenge in using CurveCP: it does traffic control over UDP as well as security. The consequence is that there is only one implementation (in the NaCl library), which is difficult to integrate into existing systems. It is also uncertain how well CurveCP's traffic control works. TCP on the other hand, is available everywhere and is well understood (for good, and bad).

### CurveZMQ as an Interim Solution

CurveCP is remarkable for several reasons, and stands a good chance of becoming a real protocol since it addresses in one package a set of problems that many people have struggled with for some time. The world wants a secure, reliable UDP protocol, but it will take years to see this implemented at the kernel level, where it belongs.

As an interim solution, we have taken CurveCP's security handshake alone, and implemented that design on top of TCP as CurveZMQ. The results are good. We believe it is easier to validate this security model than the whole CurveCP design, we believe this will solve an immediate need (secure messaging at Internet scales), and we believe it will help CurveCP gain further mind-share.

## CurveZMQ Reference Implementation

The reference implementation for CurveZMQ, which tracks this specification and always aims to implement the latest stable version, is the [Curve project](https://github.com/zeromq/libcurve).
