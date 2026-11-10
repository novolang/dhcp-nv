# Changelog

All notable changes to dhcp-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0

First release.

### Added

- The DHCPv4 message of RFC 2131 in a 576-byte buffer: the header, the
  magic cookie, the options of RFC 2132, the option overload of section
  9.3 and the long options of RFC 3396.
- The client of RFC 2131 section 4.4 in the states INIT, SELECTING,
  REQUESTING, BOUND, RENEWING and REBINDING, with T1 and T2, the
  retransmission schedule, DHCPDECLINE and DHCPRELEASE.
- A server for a pool of up to 16 addresses: offers, acknowledgements,
  refusals, releases, declines and DHCPINFORM.
- `tests/embedded_probe.nv`, a client and a server run to a lease on a
  Cortex-M4 under QEMU. The image for `--target=nrf52-qemu` is 25,548 bytes
  of flash, of which the program is 12,812 bytes and the runtime and
  libgcc the rest; the probe uses no heap.

### Not included

- DHCPv6 (RFC 8415), the INIT-REBOOT and REBOOTING states, a relay agent,
  the parameter request list in the server, and any socket.

This release requires novo 0.24.0 or later.
