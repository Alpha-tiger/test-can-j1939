# CAN-J1939 on linux

The [Kickstart guide is here](can-j1939-kickstart.md)

## CAN on linux

See [Wikipedia:socketcan](http://en.wikipedia.org/wiki/Socketcan)

## J1939 networking in short

* Add addressing on top of CAN (destination address & broadcast)

* Any (max 1780) length packets.
  Packets of 9 or more use **Transport Protocol** (fragmentation)
  Such packets use different CANid for the same PGN.

* only **29**bit, non-**RTR** CAN frames

* CAN id is composed of
  * 0..8: SA (source address)
  * 9..26:
    * PDU1: PGN+DA (destionation address)
    * PDU2: PGN
  * 27..29: PRIO

* SA / DA may be dynamically assigned via j1939-81  
  Fixed rules of precedence in Specification, no master necessary

## J1939 on SocketCAN

J1939 is *just another protocol* that fits
in the Berkely sockets.

	socket(AF_CAN, SOCK_DGRAM, CAN_J1939)

## differences from CAN_RAW
### addressing

SA, DA & PGN are used, not CAN id.

Berkeley socket API is used to communicate these to userspace:

  * SA+PGN is put in sockname ([getsockname](http://man7.org/linux/man-pages/man2/getsockname.2.html))
  * DA+PGN is put in peername ([getpeername](http://man7.org/linux/man-pages/man2/getpeername.2.html))  
    PGN is put in both structs

PRIO is a datalink property, and irrelevant for interpretation
Therefore, PRIO is not in *sockname* or *peername*.

The *data* that is [recv][recvfrom] or [send][sendto] is the real payload.
Unlike CAN_RAW, where addressing info is data.

### Packet size

J1939 handles packets of 8+ bytes with **Transport Protocol** fragmentation transparently.
No fixed data size is necessary.

	send(sock, data, 8, 0);

will emit a single CAN frame.

	send(sock, data, 9, 0);

will use fragementation, emitting 1+ CAN frames.

## Enable j1939 (obsolete!)

CAN has no protocol id field.
The can-j1939 stack only activates when a socket opens
for a network device.

The methods described here existed in earlier implementations.

### netlink

	ip link set can0 j1939 on

This method is obsoleted in favor of _on socket connect_.

### procfs for legacy kernel (2.6.25)

This API is dropped for kernels with netlink support!

	echo can0 > /proc/net/can-j1939/net

# Using J1939

## BSD socket implementation
* socket
* bind / connect
* recvfrom / sendto
* getsockname / getpeername

## Modified *struct sockaddr_can*

	struct sockaddr_can {
		sa_family_t can_family;
		int         can_ifindex;
		union {
			struct {
				__u64 name;
				__u32 pgn;
				__u8 addr;
			} j1939;
		} can_addr;
	}

* *can_addr.j1939.pgn* is PGN

* *can_addr.j1939.addr* & *can_addr.j1939.name*  
  determine the ECU

  * receiving address information,  
    *addr* is always set,  
    *name* is set when available.

  * When providing address information,  
    *name* != 0 indicates dynamic addressing

## iproute2 (obsolete!)

Older versions of can-j1939 used a modified iproute2
for manipulating the kernel lists of current addresses.

### Static addressing

	ip addr add j1939 0x80 dev can0

### Dynamic addressing

	ip addr add j1939 name 0x012345678abcdef dev can0

