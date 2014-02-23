Mosh Session Multiplexing
-------------------------

Reference: https://github.com/keithw/mosh/issues/295

### Design Goals

* The existing mosh functionality shall be preserved.
* Multiple mosh sessions for a single user shall be supported on a single external UDP port.
* Multiple mosh sessions for multiple users shall be supported on a common single external UDP port.

### Candidate Design Requirements

#### Connection Negotiation

* When negotiating with the mosh-server, a port multiplexing capable mosh-client shall send `MOSH_PORT_MULTIPLEX=1`.
* When the mosh-server starts, the mosh-server shall connect to moshd using a SOCK_STREAM UNIX socket.
* If mosh-server successfully connects to moshd and the mosh-client has sent `MOSH_PORT_MULTIPLEX`, the mosh-server shall operate in multiplex mode, otherwise the mosh-server shall operate in non-multiplex mode.
* When the mosh-server operates in non-multiplex mode, the mosh-server shall operate in the existing manner.
* When the mosh-server operates in multiplex mode, the mosh-server shall create a multiplexed port connection with moshd.
* When the mosh-server receives the session connection parameters from moshd, the mosh-server will respond to the mosh-client with `MOSH MULTIPLEX-CONNECT port token key` instead of `MOSH CONNECT port key`.
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
* When moshd receives a packet on the public UDP port, moshd shall extract the Session Token and find the associated moshd session.
* If moshd fails to find an associated moshd session, moshd shall drop the packet.
* When moshd finds the associated moshd session, and when this is the first packet received from this source IP address and UDP port, moshd shall record this as the new mosh-client address.
* moshd shall remove the Session Token and send the remainder of the packet via the associated SOCK_STREAM UNIX socket.
* When moshd receives a packet from the SOCK_STREAM UNIX socket, moshd shall prepend the Session Token and send the resuting packet to the recorded mosh-client address, or drop the packet if no mosh-client address is available. 

### Discussion

No mention has been made of any additional encryption over above the currently supported functionality. The above design does not attempt to encrypt the channel between the mosh-client and moshd, nor to encrypt the channel between moshd and the mosh-server, relying only on the end-to-end secure channel between the mosh-client and mosh-server.

The moshd to mosh-server channel is relatively secure since it uses an internal SOCK_STREAM UNIX socket channel relying on existing mechanisms that secure processes running on the same host. The use of a stream allows either end to determine if the its peer has unexpectedly terminated, and avoids the possibility of the another process masquerading as the peer.

The mosh-client to moshd connection is no less secure than the current mosh-client to mosh-server connection, with the exception of the Session Token. Since the Session Token is temporally and globally unique, it is not possible to mistake one instance for another.

It is possible for Session Tokens to be extracted from packets in flight, and for a malicious agent to either masquerade as the real mosh-client and disrupt, or to perhaps attempt a man-in-the-middle maneuver. The exposure to either of these attacks is no greater than for the current mosh-client to mosh-server design because moshd simply forwards packets on to the designated mosh-server.

It is possible to provide some additional mitigatation by encrypting the Session Token that is prepended to the in-flight packets between mosh-client and moshd. The main payload is already encrypted so there is nothing to be gained by doubly encrypting that data. Because the entire session establishment process overs over a secure ssh channel, a symmetric key encryption of the Session Token is sufficient.
