# The Controller image and the Host on a PC

`noble-examples` holds two programs.  This page lists the modules each one is made of, the packages they read, how the two meet, what the
configuration changes, and what the package leaves out.

## How the two programs meet

```
 PC                                          nRF54LM20 DK
 main_pc  ── noble: advertiser, hci_transport  hci_uart ── noble-controller: ctl_hci, ll_air
    │        noble-hci: hci_frame, hci_codec     │         noble-radio: Radio, Cipher, Ecdh
    │                                            │         board blocks: radio, CCM, P-256, RNG
 hci_socket ── UNIX socket ── socat ── USB ── UARTE30 (1 Mbaud, RTS/CTS)
```

The Host sends HCI command packets and the Controller answers with HCI event packets, each framed by its HCI packet indicator ([Vol 4]
Part A, Section 2).  The Host's `hci_socket` module carries those octets over a stream socket; `socat` relays the socket to the serial port;
the Controller's `HostPort` hands them to the UART.  Neither program knows the other's code: the seam between them is the HCI packet.

## The modules

| File | Package it belongs to | Holds |
|---|---|---|
| `controller/hci_uart_main.nv` | this package | the entry of the Controller image: the start-up, the loop, the Host's port, the status line |
| `controller/chip/status_leds.nv` | this package | the four LEDs, written from the Controller's state |
| `controller/chip/led_snoop.nv` | this package | the follower of the HCI stream that reads the vendor command `0xFC10` (Volume, Mute) and its tests |
| `controller/chip/rng_ctl.nv` | this package | the random octets of the chip's generator handed to the Controller's LE Rand answer |
| `pc/main_pc.nv` | this package | the entry of the Host: the arguments, the bring-up, the loop that prints packets |
| `pc/hci_socket.nv` | this package | `HciSocket`: the HCI UART Transport Layer over a UNIX-domain stream socket |
| `pc/hci_console.nv` | this package | the console lines of an HCI exchange |

The Controller image reads `ctl_hci` and `ll_air` of `noble-controller` (the commands it answers and the radio program it runs),
`ll_pdu`, `ll_cipher` and `ll_ecdh` of `noble-radio` (the buffer for one PDU and the cipher and key-agreement traits the board implements),
`ll_policy` (the policy the Link Layer asks about connection requests) and, from the board package of the toolchain, `radio_nrf54`,
`ccm_nrf54`, `ecdh_nrf54`, `rng_nrf54` and `pdu_mem`.  The Host reads `advertiser`, `hci_transport` and `hci_deframer` of `noble`, and `hci_frame`,
`hci_codec`, `hci_commands` and `hci_h4` of `noble-hci`.  Every item of those modules that the programs name is `pub`.

## The effects

`hci_uart_main.nv` declares `main` as `[hw, mutate]`, and so do the functions that touch a register.  A function that only computes (`header_octets` of
`led_snoop`, for one) declares no effect and carries `@no_alloc`.  `pc/main_pc.nv` declares `[io, net, mutate]`:
`net` for the socket, `io` for the console, `mutate` for the connection it updates.  The Host is built for the host; the package's
embedded tier applies to the Controller image, which is why the programs under `pc/` are in a directory of their own: a module under `src/`
would join every device build, and the device runtime has no `std.net`.

## The configuration

The Controller's tables are the keys of the `[noble-controller]` schema, with the defaults of the `device` profile.  A consumer overrides them in
a config file (`docs/config.md` of the toolchain).  `configs/controller_iso.toml` extends the profile `ctl_iso` of `noble-controller`, which
sets `ctl_cis = 4`, `ctl_cigs = 1`, `ctl_bigs = 2`, `ctl_bis = 4` and the features an isochronous Host asks for (power control, subrating,
feature pages, path loss, peer SCA, connection rate, frame space).  The isochronous tables cost 209,736 B of flash (259,776 B to 469,512 B)
and no RAM in the figures of the memory report, because they are fields of the `Controller` value on the stack of `main`.  Print the
storage a profile adds up to with

```sh
novo config explain --config configs/controller_iso.toml
```

## Check without a board

```sh
novo test controller/chip/led_snoop.nv
novo pkg build --bin main_pc --target=host -o main_pc && ./main_pc --help
```

The first command runs five tests of the vendor command.  The second builds the Host and prints its usage.  The Host's status codes can be
read without a Controller: an address that is not six colon-separated octets exits with 2, and a socket nothing listens on exits with 1.

## Dependencies

`novo.toml` names the packages by registry range (`noble = "^0.1.0"`), as a published package must: `novo pkg publish` refuses a `path`
dependency, and a clone of this package would resolve a path to a directory it does not have.  In the novo repository the packages are
developed side by side, and a copy of this package with the ranges replaced by the sibling directories is what the repository's gate builds.

## What is not here

The noble stack also has an LE Audio earbud (a Unicast Server, a Volume Renderer, the Device Information and Battery Services) and an Auracast
Broadcast Source, each a Host and a Controller in one image on the nRF54LM20 DK, and each verified with a phone.  Their Host is the
session of the Bluetooth Test Protocol that the stack's conformance tooling uses: a Host that a test tool configures with packets, which the
image gives it at start-up.  That session is not a published package, and a program that links it cannot be built from the registry
versions of the packages above.  The two images join this package when the session is published, or when they are rewritten over the
public Host API of `noble`, `noble-audio` and `noble-services`.
