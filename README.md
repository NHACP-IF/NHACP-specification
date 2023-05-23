# The NHACP specification

This repository is the home of the NABU HCCA Application Communication
Protocol specification.  This specification is the result of a collaborative
effort between NHACP client and server implementers and other
retro-computing enthusiasts.  The purpose of this specification is to provide a
common framework for connected applications running on resource-limited
computer systems, such as the NABU Personal Computer (which features a
Z80 CPU and 64KB of RAM).

The NHACP specification is in active development and currently has provisions
for:
* Block-oriented I/O to physical and virtual block devices and regular files.
* Sequential and positional I/O to regular files.
* File metadata operations (get-info, set-size).
* Directory enumeration.
* File namespace operations (remove, rename, make-directory).
* Date/time queries.

![CC BY-SA 4.0](https://i.creativecommons.org/l/by-sa/4.0/88x31.png)

The NHACP specification is licensed under a
[Creative Commons Attribution-ShareAlike 4.0 International License](https://creativecommons.org/licenses/by-sa/4.0/).

The NHACP specification is available [here](nhacp-specification.md).  The
tip of the main branch will always describe the latest released version of
the protocol, which is currently **NHACP-0.1**.
