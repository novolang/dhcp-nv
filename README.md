# dhcp-nv

The Dynamic Host Configuration Protocol, version 4, is how a device on an
IP network asks for an address, a subnet mask, a gateway and a DNS server,
and keeps the answer for a limited time called a lease. It is specified in
[RFC 2131](https://www.rfc-editor.org/rfc/rfc2131), and the options it
carries are in [RFC 2132](https://www.rfc-editor.org/rfc/rfc2132). This
package is the protocol without a network: a client that obtains and keeps
a lease, and a server for a small pool, both written for a device with no
heap. Every message is a fixed 576-byte buffer. The caller supplies the
clock, a random number, and the two functions that send and receive a UDP
datagram.

## A device that obtains a lease

A device with a network interface runs the client in a loop of two calls.
`client_tick` is called when the time `client_next_at` comes, and it
builds the next message to send. `client_receive` is called when a datagram
arrives on port 68, and it builds a reply when one is due. Both answer an
action that says what to do with the buffer.

The program below runs that loop against a server in the same program, so
that it can be run anywhere. On a board the server is the network: the
lines marked *send* write the buffer to the interface, and the datagram
that `client_receive` reads is the one the interface received.

```novo
use dhcpmsg
use dhcpclient
use dhcpserver

fn main() [io]
    // The device's hardware address, 02:00:00:00:00:01, and its client.
    var mac: Vec[u8; 16] = Vec.new()
    for b in [2, 0, 0, 0, 0, 1]
        let _p = mac.push(b as u8)
    var client = dhcpclient.client_new(var mac, 1, true)

    // Stand-in for the network: a server that leases 192.168.4.100 to .109.
    var server = dhcpserver.server_new(dhcpmsg.ip(192, 168, 4, 1), dhcpmsg.ip(255, 255, 255, 0),
                                       dhcpmsg.ip(192, 168, 4, 1), dhcpmsg.ip(192, 168, 4, 53),
                                       3600, dhcpmsg.ip(192, 168, 4, 100), 10)

    var datagram: Vec[u8; 576] = Vec.new()
    var answer: Vec[u8; 576] = Vec.new()
    let now = 0

    // The first tick starts the exchange with a DHCPDISCOVER.
    var action = dhcpclient.client_tick(var client, now, 0x5EED1234, var datagram)
    for _ in 0..4
        if action == dhcpclient.DHCP_ACT_BROADCAST or action == dhcpclient.DHCP_ACT_UNICAST
            // Send: write `datagram` to the interface, to the broadcast
            // address for a broadcast and to the server for a unicast.
            // Receive: here the server's answer comes back at once.
            let sent = dhcpserver.server_handle(var server, var datagram, var answer, now)
            if sent.action == dhcpserver.DHCP_SEND_REPLY
                action = dhcpclient.client_receive(var client, now, var answer, var datagram)

    if dhcpclient.client_has_lease(var client)
        let ip = dhcpclient.client_ip(var client)
        println("address ${dhcpmsg.ip_octet(ip, 0)}.${dhcpmsg.ip_octet(ip, 1)}.${dhcpmsg.ip_octet(ip, 2)}.${dhcpmsg.ip_octet(ip, 3)}")
        println("renew at ${dhcpclient.client_t1_at(var client)} seconds")
```

## What DHCP is

A client without an address broadcasts a **DHCPDISCOVER**. Each server
that can serve it answers with a **DHCPOFFER** that names an address. The
client picks one and broadcasts a **DHCPREQUEST** for it, naming the
server, and the server confirms with a **DHCPACK** that carries the lease
time and the configuration. The client then holds a **lease**. A server
that cannot grant what a client asks for answers with a **DHCPNAK**.

A lease has two timers, both measured from the moment it was granted.
At **T1**, by default half the lease, the client asks the server that
granted it to extend the lease. At **T2**, by default seven eighths of
the lease, it asks any server. When the lease ends, the client must stop
using the address. These are the six states of RFC 2131 figure 5.

| State | The client is | Leaves on |
| --- | --- | --- |
| INIT | without a lease and without an exchange | its next tick |
| SELECTING | waiting for offers after a DHCPDISCOVER | an offer |
| REQUESTING | waiting for an ACK after a DHCPREQUEST | an ACK or a NAK, or five transmissions |
| BOUND | holding a lease | T1 |
| RENEWING | asking its server to extend the lease | an ACK, a NAK, or T2 |
| REBINDING | asking any server to extend the lease | an ACK, a NAK, or the end of the lease |

| Quantity | Value |
| --- | --- |
| Fixed header, up to the magic cookie | 236 bytes |
| Options start at | byte 240 |
| Magic cookie | `0x63825363` |
| Message buffer | 576 bytes |
| Shortest message sent | 300 bytes |
| Server port, client port | 67, 68 |
| Retransmission wait | 4, 8, 16, 32, 64 seconds, then 64, each one second early or late |
| Wait between renewals | half the time left, and at least 60 seconds |
| Wait after a DHCPDECLINE | 10 seconds |
| Requests sent before the client starts again | 5 |
| Lease that never ends | `0xFFFFFFFF` seconds |
| Addresses in the server's pool | 1 to 16 |

## Install

```
novo pkg add dhcp-nv
```

## What the package contains

| Module | Contents |
| --- | --- |
| `dhcpmsg` | The message in a `Vec[u8; 576]`: the header fields, the options with the overload and long-option rules, the writers, the faults of a received message, and the numbers RFC 2131 and RFC 2132 assign. |
| `dhcpclient` | The client: the six states, the retransmission schedule, T1 and T2, the lease, and the messages DHCPDISCOVER, DHCPREQUEST, DHCPDECLINE and DHCPRELEASE. |
| `dhcpserver` | The answer builder of a server with a pool of up to 16 addresses: offers, acknowledgements, refusals, releases, declines and DHCPINFORM. |

## How to choose an entry point

**`dhcpclient` is for the device.** It holds the whole exchange in one
`DhcpClient` value of 168 bytes and builds every message the client sends.

**`dhcpserver` is for a gateway, a test fixture or a bench.** It answers
the questions a small network asks and keeps its table in one
`DhcpServer` value of 448 bytes.

**`dhcpmsg` is for a program that reads or writes single messages.** A
packet tool, a test that builds a request, and a relay that reads the
`giaddr` field use the readers and writers directly.

## The rules a user needs

1. **The clock is a number of seconds, and the caller reads it.** Every
   function takes `now`. The unit is seconds and the origin is the
   caller's, so the lease times and the timers in the client are in the
   same unit. A clock that goes backwards is not handled.
2. **The caller supplies a random word.** `client_tick` uses it as the
   transaction identifier of a new exchange and to spread the
   retransmissions by one second either way. A device with no random
   source can pass a hash of its hardware address and its uptime.
3. **An action says where the buffer goes.** `DHCP_ACT_BROADCAST` means
   255.255.255.255 port 67, from port 68, with source address 0.0.0.0
   until the client has a lease and the leased address when it is
   rebinding. `DHCP_ACT_UNICAST` means `client_server` port 67, from the
   leased address.
4. **Call `client_tick` when `client_next_at` comes.** It does nothing
   earlier. For a client that holds a lease that never ends,
   `client_next_at` is -1.
5. **The first offer is taken.** A client that wants to compare offers
   keeps the clock and waits before it calls `client_receive`.
6. **Do not use an address after `DHCP_ACT_NAK` or `DHCP_ACT_EXPIRED`.**
   The client is back in INIT with no address. It starts a new exchange
   on its next tick.
7. **Check the address before using it.** RFC 2131 section 3.1 asks the
   client to probe a new address with ARP. If another host answers, call
   `client_decline`, which builds a DHCPDECLINE and waits ten seconds.
8. **T1 and T2 must satisfy T1 < T2 < lease.** A server that sends
   options 58 and 59 that do not is ignored, and the defaults of half and
   seven eighths are used (RFC 2131 section 4.5).
9. **A NAK is believed only from the server the client is talking to.**
   A DHCPNAK that names another server is ignored, so that a forged one
   cannot end a lease.
10. **A message is 576 bytes at most.** The client advertises 576 in
    option 57. A datagram the caller received that is longer must be
    dropped or truncated by the caller; the buffer cannot hold more.
11. **A writer that answers false has left a message that is not
    usable.** `put_option_bytes` and the other writers answer false when
    the buffer is full. The message built so far is discarded.
12. **An option in pieces is one value.** RFC 3396 allows an option longer
    than 255 bytes to be sent as several options with the same code, and
    the option overload of RFC 2132 section 9.3 moves options into the
    `sname` and `file` fields. The readers join the pieces in the order
    the message gives them and read the three areas as one.
13. **A reader answers -1 for what is not there.** `option_u32`, `option_ip`
    and the others answer -1 when the option is absent or has the wrong
    length. A header field read from a message that is too short is 0.
14. **The server sends its replies to the place RFC 2131 section 4.1
    names.** To the relay agent if `giaddr` is set, to the broadcast
    address for a DHCPNAK, to `ciaddr` if the client has an address, to
    the broadcast address if the client set the broadcast flag, and
    otherwise to the address being assigned. The answer carries the
    destination and the port.
15. **The server keeps its table in memory.** A restart forgets every
    lease. It identifies a client by its client identifier option, or by
    its hardware type and address when there is none, and the key is a
    64-bit hash of those bytes.
16. **The server does not probe an address before it offers it.** An
    ARP probe is the caller's, as it is for the client.

## Running on a microcontroller

All three modules are written for a device with no heap allocator, and the
compiler checks that claim on every build. Every function is
`@tier(embedded)` and `@no_alloc`, so none of them reaches the allocator.
Every function performs no input or output and reads no clock, and none
declares an effect.

| What | Size |
| --- | --- |
| A message, `Vec[u8; 576]` | 584 bytes of the caller's stack or static storage |
| `DhcpClient` | 168 bytes |
| `DhcpServer` with 16 slots | 448 bytes |
| `DhcpScan`, the result of a walk over the options | 40 bytes |

The client and the server need a message buffer each to run an exchange,
and a device that is only a client needs two: one to receive into and one
to build in. The same buffer can serve for both when the caller is done
with the received message before it builds the reply.

`tests/embedded_probe.nv` is the claim as a program that either builds or
does not. It runs a client and a server in one image through discovery,
request, acknowledgement, renewal at T1 and release, and prints
`PASS: dhcp-embedded`:

```bash
novo build --target=nrf52-qemu tests/embedded_probe.nv
qemu-system-arm -machine mps2-an386 -nographic -semihosting -kernel probe.elf
```

The registry shows `embedded` among the tiers of this package because
every module checks clean at that tier.

## What is not included

- **DHCPv6 (RFC 8415)** is NOT IMPLEMENTED. Its messages, its options and
  its state machine are different from DHCPv4's.
- **The INIT-REBOOT and REBOOTING states** of RFC 2131 section 4.4.2 are
  NOT IMPLEMENTED. A client that has restarted starts from INIT.
- **Sockets and ARP.** The caller sends and receives the datagrams and
  probes the address.
- **Dynamic DNS and the other options.** The client asks for the mask, the
  router, the DNS servers and the lease times. Other options can be read
  with `option_len` and `option_byte`.
- **A parameter request list in the server.** It sends the mask, the
  router and one DNS server whatever the client asked for.
- **A lease database that survives a restart.**
- **A relay agent.** The server answers a request that came through one;
  forwarding is not included.

## Related packages

- [ntp-nv](https://novo-lang.org/packages/ntp-nv) and
  [dns-codec-nv](https://novo-lang.org/packages/dns-codec-nv) are
  protocol codecs of the same shape: bytes in, values out, no socket.
  A device that has a lease usually asks them next, for the time and for a
  name.
- [ipaddr-nv](https://novo-lang.org/packages/ipaddr-nv) parses and prints
  addresses as text. The addresses here are 32-bit integers.

## Tests

```bash
novo test tests/dhcp_msg_tests.nv        # the message, byte by byte
novo test tests/dhcp_client_tests.nv     # the six states, the timers, the faults
novo test tests/dhcp_server_tests.nv     # offers, refusals, releases and the pool
novo test tests/dhcp_exchange_tests.nv   # the client against the server, and damaged messages
bash tests/coverage.sh                   # line coverage over src/, merged across the suites
```

The message vectors are written out by hand from the diagram of RFC 2131
figure 1 and the option formats of RFC 2132, with the field names beside
the bytes. The overload and long-option tests place options in the `file`
and `sname` areas and split one option across two instances. The client
tests set the clock and write the server's messages themselves, so each
test says what the client was told. The exchange tests damage a message at
every byte and require that no reader traps.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
