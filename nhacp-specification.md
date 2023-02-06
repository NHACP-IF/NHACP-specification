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

All messages exchanged use the little-endian byte order.  Thus, a 16
bit number 0xaa00 is transmitted as the two bytes 0x00 and 0xaa.

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

### STORAGE-OPEN

Open a URL and assign a storage index to it.  The URL can be relative
or a file URL to open a local file.  Network adapters may implement
additional, nonstandard URL schemes.  Relative URLs are interpreted 
as files.  The flags field can be used to pass additional information
to the storage handler.  The meaning of the flags is not specified in
the protocol itself.

The network adapter will attempt to use the storage slot specified by
the caller unless the caller specifies 0xff, in which case the network
adapter will attempt to allocate a storage slot.  If the requested slot
is already in-use by another storage object, the STORAGE-OPEN request
MUST fail.

| Name       | Type  | Notes                                                              |
|------------|-------|--------------------------------------------------------------------|
| type       | u8    | 0x01                                                               |
| index      | u8    | Storage slot to use for response (0xff => Network Adapter selects) |
| flags      | u16   | Flags to pass to storage handler (TBD)                             |
| url-length | u8    | Length of resource in bytes                                        |
| url        | char* | URL String                                                         |

Possible responses: STORAGE-LOADED, ERROR

### STORAGE-GET

Get data from network adapter storage.

N.B. The maximum payload length that can be returned to the caller
is the maximum message length (32767) _minus_ the size of the
DATA-BUFFER reply message (3) (32767 - 3 -> 32764 bytes).  Servers
SHOULD return an error for STORAGE-GET requests whose length field
exceeds this value.

| Name   | Type | Notes                            |
|--------|------|----------------------------------|
| type   | u8   | 0x02                             |
| index  | u8   | Storage slot to access           |
| offset | u32  | Offset into the storage in bytes |
| length | u16  | Number of bytes to return        |

Possible responses: DATA-BUFFER, ERROR

The length returned in the DATA-BUFFER response reflects
the amount of data actuallty read from the underlying
storage object.  If the offset is beyond the object's
end-of-file, then the returned length MUST be 0.
If the read operation would cross the object's end-of-file,
then the length MUST be the number of bytes read before
the end-of-file was encountered.

### STORAGE-PUT

Update data stored in the network adapter.  If possible, the
underlying storage (file/URL) should be updated as well.

N.B. The maximum payload length that can be sent to the server
is the maximum message length (32767) _minus_ the size of the
STORAGE-PUT request message (8) (32767 - 8 -> 32759 bytes).  Servers
SHOULD return an error for STORAGE-PUT requests whose length field
exceeds this value.

| Name   | Type | Notes                            |
|--------|------|----------------------------------|
| type   | u8   | 0x03                             |
| index  | u8   | Storage slot to access           |
| offset | u32  | Offset into the storage in bytes |
| length | u16  | Number of bytes to write         |
| data   | u8*  | Data to update the storage with  |

Possible responses: OK, ERROR

If a write originates at or beyond the underlying storage
object's end-of-file or therwise crosses the end-of-file,
then the underlying storage object SHOULD be implicitly
enlarged to accommodate the write.  For writes that originate
beyond end-of-file, the region between the old end-of-file and
the newly-written region MUST be implicitly zero-filled.  If
a server implementation does not support extending the underlying
storage object, then the server MUST return an error without
performing the write operation.

### GET-DATE-TIME

Retrieve the current date and time from the network adapter.

| Name | Type | Notes |
|------|------|-------|
| type | u8   | 0x04  |

Possible responses: DATE-TIME, ERROR

### STORAGE-CLOSE

Close a previously opened storage slot.  Any resources associated with
it on the network adapter will be freed.

| Name  | Type | Notes                 |
|-------|------|-----------------------|
| type  | u8   | 0x05                  |
| index | u8   | Storage slot to close |

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

Type tags in response messages have their MSB set.  Their numbering
must be consecutive to support fast dispatching on the type byte.

### NHACP-STARTED

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

| Name           | Type      | Notes                   |
|----------------|-----------|-------------------------|
| type           | u8        | 0x82                    |
| code           | u16       | Error code (TBD)        |
| message-length | u8        | Length of error message |
| message        | char[255] | Error message           |

### STORAGE-LOADED

| Name   | Type | Notes                                       |
|--------|------|---------------------------------------------|
| type   | u8   | 0x83                                        |
| index  | u8   | Storage index that was provided or selected |
| length | u32  | Length of the data that was buffered        |

### DATA-BUFFER

| Name   | Type | Notes            |
|--------|------|------------------|
| type   | u8   | 0x84             |
| length | u16  | Length of data   |
| data   | u8*  | Data from buffer |

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
