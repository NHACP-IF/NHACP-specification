# NABU HCCA application communication protocol

The original NABU network adapter provided a unidirectional connection
from the headend system to the NABUs.  Each NABU could access the
software that was periodically broadcast through the cable network,
but they could not send any data back to the headend.

With the new surge of interest in the NABU PC, the original network
adapter has been replaced by virtual network adapters running on a
local system (i.e. a PC running Windows or Linux, or an embedded
system).  With these network adapters, bidirectional communication
through the network adapter interface is possible and opens up the
possiblity of using the NABU as an internet terminal or providing
network based storage services to applications running on the NABU.

This document proposes a unified protocol to be implemented by network
adapters to support NABU applications.

## Serial communication

The NABU HCCA serial interface is an asynchronous RS422 port running
at ~111000 bits per second using 8 bits per character, no parity and
one stop bit.  As the bitrate of the interface does not match any of
the standard baud rates provided by serial interfaces on modern
computers, network adapters usually use the standard 115200 bps baud
rate with two stop bits.  While this is slightly too fast, it matches
close enough for both sides to be able to read the bytes coming from
the other end.

## Message framing

The original network adapter protocol did not have any message
framing.  Each message that was exchanged with between the NABU and
the adapter had its specific length and response requirements.  This
protocol is not easy to extend, and as such does not form a good basis
for functionality proposed in this document.  Thus, for the remainder
of this document, it is assumed that the NABU application and the
network adapter are switched to the new protocol.  Once they have
switched, the old protocol can not be used until explicitly
re-enabled.

All communication in the protocol is initiated by the NABU side.  The
NABU sends a request message which is responded to by the network
adapter with exactly one response message.  For real-time
communication applications in which messages are sent to the NABU
through e.g. a network service, the network adapter must buffer these
messages until the NABU requests to see them.

All messages exchanged use the big-endian byte order.  Thus, a 16 bit
number 0xaa00 is transmitted as the two bytes 0x00 and 0xaa.

Each message consists of a 16 bit length field followed by the number
of bytes indicated by the length field.  The maximum length of a
message is 32767 bytes to ensure that the most significant bit of the
message length is never set.  This helps in crash recovery (see
below).  No further framing is provided.  Messages must be completely
transmitted within one second.  The receiver should ignore partial
messages and signal an error to the operator if it detects the timeout
condition.

## Message type

The first field of each message is a 1 byte message type.  Request
message types (NABU -> Network Adapter) have the MSB set to zero,
response message types have MSB set to one.  The layout of the message
itself depends on the message type.

## Switching to the new protocol

The NABU application switches to the new protocol by sending the
single byte 0xaf to the network adapter.  If the network adapter
supports the new protocol, it will respond with an PROTOCOL-STARTED
response (see below) and then wait for further messages in the new
framing format.

## Request messages

### STORAGE-HTTP-GET

GET a URL and buffer the contents in the network adapter under the
given index.

| Name       | Type  | Notes                           |
|------------|-------|---------------------------------|
| type       | u8    | 0x01                            |
| index      | u8    | Storage slot to use for reponse |
| url-length | u8    | Length of url in bytes          |
| url        | char* | URL String                      |

Possible responses: STORAGE-LOADED, ERROR

### STORAGE-LOAD-FILE

Load a file stored in in the network adapter under the given index.

| Name            | Type  | Notes                                      |
|-----------------|-------|--------------------------------------------|
| type            | u8    | 0x02                                       |
| index           | u8    | Storage slot to use for the file's content |
| filename-length | u8    | Length of filename in bytes                |
| filename        | char* | Filename                                   |

Possible responses: STORAGE-LOADED, ERROR

### STORAGE-GET

Get data from network adapter storage

| Name   | Type | Notes                            |
|--------|------|----------------------------------|
| type   | u8   | 0x03                             |
| index  | u8   | Storage slot to access           |
| offset | u32  | Offset into the storage in bytes |
| length | u16  | Number of bytes to return        |

Possible responses: DATA-BUFFER, ERROR

### STORAGE-PUT

Update data stored in the network adapter.  If possible, the
underlying storage (file/URL) should be updated as well.

| Name   | Type | Notes                            |
|--------|------|----------------------------------|
| type   | u8   | 0x04                             |
| index  | u8   | Storage slot to access           |
| offset | u32  | Offset into the storage in bytes |
| length | u16  | Number of bytes to write         |
| data   | u8*  | Data to update the storage with  |

Possible responses: OK, ERROR

### GET-DATE-TIME

Retrieve the current date and time from the network adapter.

| Name | Type | Notes |
|------|------|-------|
| type | u8   | 0x05  |

Possible responses: DATE-TIME, ERROR

### END-PROTOCOL

Return to legacy protocol processing, i.e. before returning to the
legacy menu system.

| Name | Type | Notes |
|------|------|-------|
| type | u8   | 0xEF  |

No specific response message is returned by the network adapter.  The
NABU is free to send legacy messages and expect the adapter to respond
like normal.

## Response messages

### PROTOCOL-STARTED

| Name              | Type  | Notes                                   |
|-------------------|-------|-----------------------------------------|
| type              | u8    | 0x80                                    |
| version           | u16   | Version number of the protocol          |
| adapter-id-length | u8    | Length of adapter identification string |
| adapter-id        | char* | Adapter identification string           |

### OK

| Name | Type | Notes |
|------|------|-------|
| type | u8   | 0x81  |

### ERROR

| Name           | Type  | Notes                   |
|----------------|-------|-------------------------|
| type           | u8    | 0x82                    |
| message-length | u8    | Length of error message |
| message        | char* | Error message           |

### STORAGE-LOADED

| Name   | Type | Notes                                |
|--------|------|--------------------------------------|
| type   | u8   | 0x83                                 |
| length | u32  | Length of the data that was buffered |

### DATA-BUFFER

| Name   | Type | Notes              |
|--------|------|--------------------|
| type   | u8   | 0x84               |
| length | u16  | Length of response |
| data   | u8*  | Data from buffer   |

### DATE-TIME

| Name | Type    | Notes                           |
|------|---------|---------------------------------|
| type | u8      | 0x85                            |
| date | char[8] | Current date in YYYYMMDD format |
| time | char[6] | Current time in HHMMSS format   |

## Recovering from a crash

When a NABU application crashes while the modern protocol is selected,
the network adapter needs to detect that the NABU is restarting.  As
the NABU ROM sends at least 4 "SET STATUS (0x83)" messages during
startup, the network adapter can assume that if it receives a length
field of 0x8383, it needs to restart in legacy protocol mode in order
to boot the NABU.
