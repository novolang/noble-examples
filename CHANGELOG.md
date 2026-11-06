# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.1.0

First release.  Two programs over the noble packages: `hci_uart`, the
noble Controller on the nRF54LM20 DK answering HCI over the DK's first
virtual COM port (with a profile that turns on its isochronous channels),
and `main_pc`, a Host on a PC that brings the Controller up through a UNIX
socket and advertises.  The package depends on `noble`, `noble-hci`,
`noble-controller` and `noble-radio`, by registry range.
