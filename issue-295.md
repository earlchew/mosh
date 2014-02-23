Mosh Session Multiplexing
-------------------------

Reference: https://github.com/keithw/mosh/issues/295

### Design Goals

* The existing mosh functionality shall be preserved.
* An unprivileged user running multiple mosh sessions on a single unshared unprivileged external UDP port shall be supported.
* Multiple mosh sessions for a single user on a common single system-wide privileged external UDP port shall be supported.
* Multiple mosh sessions for multiple users on a common single system-wide privileged external UDP port shall be supported.

### Candidate Design Requirements

#### Connection Negotiation

* When negotiating with the mosh-server, a port multiplexing capable mosh-client shall send `MOSH_PORT_MULTIPLEX=1`.
* When the mosh-server starts, the mosh-server shall connect to moshd using a SOCK_STREAM UNIX socket.
* If mosh-server successfully connects to moshd and the mosh-client has sent `MOSH_PORT_MULTIPLEX`, the mosh-server shall operate in multiplex mode, otherwise the mosh-server shall operate in non-multiplex mode.
* When the mosh-server operates in non-multiplex mode, the mosh-server shall operate in the existing manner.
* When the mosh-server operates in multiplex mode, the mosh-server shall create a multiplexed port connection with moshd.
* When the mosh-server receives the session connection parameters from moshd, the mosh-server will respond to the mosh-client with `MOSH MULTIPLEX-CONNECT Port Session-Token Key` instead of `MOSH CONNECT Port Key`.
* If the mosh-client receives `MOSH MULTIPLEX-CONNECT`, the mosh-client shall start a connection in multiplex-mode, otherwise if the mosh-client receives `MOSH CONNECT`, the mosh-client shall start a connection in non-multiplex mode.

#### Multiplex Port Session

* When the SOCK_STREAM UNIX socket connection is established, moshd shall create a moshd session.
* If the SOCK_STREAM UNIX socket connection is torn down, moshd shall destroy the associated moshd session.
* When a new moshd session is created, moshd shall create a Session Token that uniquely identifies the moshd session.
* moshd shall send the Session Token to the mosh-server.

#### Session Token

* Session Tokens shall uniquely identify a moshd session across any set of processes running on a host.
* Session Tokens shall uniquely identify a moshd session across any set of hosts.
* Session Tokens shall uniquely identify a moshd session across any set of sessions over time.

#### Packet Forwarding

* moshd shall listen for UDP packets on a single UDP port.
* When moshd receives a packet on the UDP port, moshd shall extract the Session Token and find the associated moshd session.
* If moshd fails to find an associated moshd session for a packet received on the UDP port, moshd shall drop the packet.
* When moshd finds the associated moshd session for a packet received on the UDP port, moshd shall remove the Session Token, prepend a Server Multiplex Header and send the remainder of the packet via the associated SOCK_STREAM UNIX socket.
* When moshd receives a packet from the SOCK_STREAM UNIX socket, moshd shall extract the Client Multiplex Header, prepend the Session Token and send the resuting packet to the indicated mosh-client.

#### Multiplex Headers

* The Server Multiplex Header shall convey the following information to the mosh-server: mosh-client IP address and UDP port.
* The Client Multiplex Header shall convey the following information to moshd: mosh-client IP address and UDP port.

### Discussion

#### UNIX Sockets

UNIX sockets have been used because the channel between mosh-server and moshd only needs to be local, and so there is no need to require any more resources from the TCP/IP stack. UNIX sockets can be bound to names in the local filesystem, and so easily allows an arbitrary number of moshd instance to be run.

Using a SOCK_STREAM allows each of the mosh-server and moshd to detect that the other has shut down the channel. In particular, this allows moshd to destroy the Session Token associated with that mosh-server instance and recover resources allocated to that channel.

There is also opportunity to try to communicate the destruction of the mosh-server to the mosh-client, but there is no guarantee that the mosh-client has TCP/IP connectivity at that time so there is no way to be sure that the mosh-client will receive the message. With this complication, it seems reasonable not to try to optimise this case, especially since this scenario already exists with a simple mosh-client and mosh-server configuration.

#### Security

No mention has been made of any additional encryption over above the currently supported functionality. The above design does not attempt to encrypt the channel between the mosh-client and moshd, nor to encrypt the channel between moshd and the mosh-server, relying only on the end-to-end secure channel between the mosh-client and mosh-server.

The moshd to mosh-server channel is relatively secure since it uses an internal SOCK_STREAM UNIX socket channel relying on existing mechanisms that secure processes running on the same host. The use of a stream allows either end to determine if the its peer has unexpectedly terminated, and avoids the possibility of the another process masquerading as the peer.

The mosh-client to moshd connection is no less secure than the current mosh-client to mosh-server connection, with the exception of the Session Token. Since the Session Token is temporally and globally unique, it is not possible to mistake one instance for another.

It is possible for Session Tokens to be extracted from packets in flight, and for a malicious agent to either masquerade as the real mosh-client and disrupt, or to perhaps attempt a man-in-the-middle maneuver. The exposure to either of these attacks is no greater than for the current mosh-client to mosh-server design because moshd simply forwards packets on to the designated mosh-server.

It is possible to provide some additional mitigatation by encrypting the Session Token that is prepended to the in-flight packets between mosh-client and moshd. The main payload is already encrypted so there is nothing to be gained by doubly encrypting that data. Because the entire session establishment process overs over a secure ssh channel, a symmetric key encryption of the Session Token is sufficient.
