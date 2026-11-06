# noble-examples

Two programs built on the packages of the noble Bluetooth Low Energy stack, with the commands that build, flash and run them.

| Program | Runs on | What it is |
|---|---|---|
| `hci_uart` | nRF54LM20 DK (PCA10184) | the noble Controller: a Link Layer that answers the HCI commands a Host sends over the DK's first virtual COM port |
| `main_pc` | a Linux PC | a Host that brings that Controller up over the serial port and advertises through it |

Together they are a complete Bluetooth Low Energy device split the way the specification splits it: the Controller on the chip, the Host on a PC, the
HCI UART Transport Layer between them.  A phone scanning for devices sees the one that `main_pc` advertises.

The programs use the packages `noble` (the Host), `noble-controller` (the Link Layer and the HCI command handler), `noble-hci` (the HCI
packets) and `noble-radio` (the traits between the Link Layer and a chip).  The radio, the cipher, the key-agreement engine and the random
generator of the DK are blocks of the board package that the toolchain carries; the program names them and holds no register.

## The Controller on the DK: `hci_uart`

**On the device.**  The image starts the DK's radio, cipher, P-256 engine and random number generator, reads the factory device address from
`FICR.DEVICEADDR`, and enters one loop.  Each turn of the loop answers the HCI command that is complete, finishes a command that waits for the
hardware, moves a bounded number of octets each way on UARTE30, and runs the next radio event of the Link Layer.  The Controller holds one
connection as Peripheral and takes the connection requests that `DefaultLinkPolicy` accepts.  The octet a turn reads is taken only when the
Controller accepts it, so a command that cannot be answered yet leaves the stream in the UART and the RTS/CTS flow control holds the Host.

**Device effects.**  `main` is `[hw, mutate]`: it reads and writes the registers of RADIO, DPPIC, CCM, the public key engine, the random
generator, UARTE30 (1 Mbaud, 8N1, RTS/CTS), the GRTC timer and the four LED pins.  The image writes no flash.  Its log goes to RTT (a start
line, and a status line every five seconds with the counts of the Controller, the UART and the P-256 engine); the UART carries only HCI.

**Board.**  nRF54LM20 DK, on USB.  The J-Link on the board makes the first virtual COM port the Host connects to.

**Build and flash.**  From the directory of this package:

```sh
novo build --target=nrf54lm20-dk --hci-uart=physical -o hci_uart.elf controller/hci_uart_main.nv
probe-rs download --verify --chip nRF54LM20A hci_uart.elf
probe-rs reset --chip nRF54LM20A
```

`--hci-uart=physical` puts HCI on the UART and the log on RTT.  `probe-rs attach --chip nRF54LM20A hci_uart.elf` shows the log.

A Host for a phone that streams LE Audio needs the Controller's isochronous channels, which the default build leaves out.  The profile
`configs/controller_iso.toml` turns them on (four Connected Isochronous Streams in one group, two Broadcast Isochronous Groups of four streams):

```sh
novo build --target=nrf54lm20-dk --hci-uart=physical --config configs/controller_iso.toml -o hci_uart_iso.elf controller/hci_uart_main.nv
```

**What to see.**  The four LEDs show what the Controller does, read from its own state once in each turn of the loop (`controller/chip/status_leds.nv`):

| LED | Meaning |
|---|---|
| LED1 | blinks once a second while the Controller advertises, four times a second while a connection is up and not encrypted, and stays on while it is encrypted |
| LED2 | on while a Connected Isochronous Stream is established or a Broadcast Isochronous Group is up |
| LED3 | toggles with every tenth packet a stream received or a broadcast sent |
| LED4 | the brightness the Host reports with the vendor command `0xFC10` (Volume, Mute); the Controller answers the command with Unknown HCI Command, which a Host ignores |

**What it costs.**  Measured with `novo build ... --panic-codes` on novo 0.23.1, from the memory report of the build:

| Build | Flash | `.data` + `.bss` | Arena |
|---|---|---|---|
| defaults of the schemas | 259,776 B (12 % of the 2,052,096 B of the RRAM) | 4,872 B | 65,536 B |
| `configs/controller_iso.toml` | 469,512 B (22 %) | 4,872 B | 65,536 B |

The Controller's tables are fields of a value on the stack of `main`, so they are in neither the data figure nor the arena.  Without
`--panic-codes` an image also carries the source path of each trap site, so the flash figure of a plain build depends on where the packages
are: a build inside the novo repository, with the packages beside it by relative path, adds 4,048 B to the first row and 5,184 B to the second,
and a build with the packages at an absolute path of 68 characters adds about twice that.

## The Host on a PC: `main_pc`

**What it does.**  `main_pc` opens a UNIX socket, sends the Controller the HCI commands of a legacy advertising bring-up (Reset, Read Local
Version Information, LE Set Random Address, LE Set Advertising Parameters, LE Set Advertising Data, LE Set Scan Response Data, LE Set
Advertising Enable), prints each command it sends (`>`) and each packet the Controller sends back (`<`) with a line that names it, and then
prints every packet the Controller sends until SIGINT or SIGTERM.  It disables advertising before it exits.  The HCI UART Transport Layer
runs over the socket: each packet follows its HCI packet indicator, and the bytes that arrive in pieces are cut into packets by
`hci_deframer`.

**Device effects.**  None on a device; the program is for the host.  Its effects are `[io, net, mutate]`: a UNIX socket and the console.

**Build.**

```sh
novo pkg build --bin main_pc --target=host -o main_pc
./main_pc --help
```

**Run.**  Two terminals.  The first holds the bridge between the DK's serial port and a UNIX socket (`socat` is the bridge; the Controller's
UART is 1 Mbaud with RTS/CTS):

```sh
socat -b 32 UNIX-LISTEN:/tmp/novo_hci.sock,unlink-early \
      /dev/serial/by-id/usb-SEGGER_J-Link_<serial>-if00,raw,b1000000,crtscts
```

One program talks to the socket at a time, and `socat` exits when that program does; start it again before the next run.  The second
terminal runs the Host:

```sh
./main_pc --socket /tmp/novo_hci.sock --name NovoPcHost --address C0:4E:4F:56:4F:01 --interval 100
```

| Flag | Meaning | Default |
|---|---|---|
| `-s`, `--socket` | the bridge's UNIX socket | `/tmp/novo_hci.sock` |
| `-n`, `--name` | the Complete Local Name to advertise | `NovoPcHost` |
| `-a`, `--address` | the static random address, most significant octet first | `C0:4E:4F:56:4F:01` |
| `-i`, `--interval` | the advertising interval in milliseconds, 20 to 10240 | `100` |

An address that is not six colon-separated octets exits with status 2, and a socket nothing listens on exits with status 1.

**What to see.**  The console prints `advertising as NovoPcHost from C0:4E:4F:56:4F:01 every 100 ms; Ctrl-C stops` after the Controller has
answered every command with Success.  LED1 of the DK then blinks once a second, and a scanner on a phone lists `NovoPcHost`.

**What it costs.**  The host binary is 1,076,040 bytes (`novo pkg build --bin main_pc --target=host`, default optimisation).

## Build and check

```sh
novo pkg build    # the Host on a PC, the entry of the package
novo test controller/chip/led_snoop.nv
```

`novo test controller/chip/led_snoop.nv` runs the five tests of the vendor command that sets the brightness of LED4.  The two programs have no
test on a PC that needs a board.  The package depends on the registry versions of the packages named in `novo.toml` (`noble`, `noble-hci`,
`noble-controller`, `noble-radio`); a device build also needs the board catalogue of the toolchain and `probe-rs` to flash.

## What is not here

The LE Audio earbud and the Auracast Broadcast Source of the noble stack run a Host on the same chip as the Controller.  Their Host is driven
by the BTP session that the stack's conformance tooling uses, which is not a published package yet, so they are not part of this package.
`docs/examples.md` states what each of them needs to join.
