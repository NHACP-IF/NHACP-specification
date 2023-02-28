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
at ~111860 bits per second using 8 bits per character, no parity and
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

Messages must be completely transmitted within one second, intended to
reduce error recovery time.

Each message consists of a 16 bit length field followed by the number of
bytes indicated by the length field.  Because messages must be completely
transmitted within one second, there is a hard ceiling on the maximum
transmission unit, dictacted by the NABU hardware.  With the native
tramission rate of 111860 bits per second and many network adapter
implementations using 2 stop bits, 11186 bytes is the practical limit
that can be transmitted within the allotted time.  For this reason, the
maximum length of a message is 8256 bytes.  This length was chosen for
the following reasons:

* It fits within the the one second time limit.
* It is sufficient for an 8KB data payload plus request/response message
  and framing overhead.
* It satisfies a requirement of the original NHACP draft that the most
  significant bit of the length field is never set, which aids in crash
  recovery (see below).

No further framing is provided.  The receiver should ignore partial
messages and signal an error to the operator if it detects the timeout
condition.

## Message type

The first field of each message is a 1 byte message type.  Request
message types (NABU -> Network Adapter) have the MSB set to zero,
response message types have MSB set to one, with two exceptions:

* The START-NHACP request message type has the value 0x8f.
* The END-PROTOCOL request message type has the value 0xef.

The layout of the message itself depends on the message type.

## Error handling

For requests that can return an error, there is a well-defined set of
error codes with standard meanings.  See the ERROR response below.

In the original NHACP draft, the ERROR response could have returned a
detailed error message in every response.  However, this requires that
applications always have a buffer large enough to receive this error
response (which was over 256 bytes in length).

To address this concern, the behavior (but not the format) of the ERROR
response has been changed to only include the fixed-sized portions of
the response in the default ERROR response.  This allows requests that
can return an ERROR response to have a buffer as small as 4 bytes to
receive the initial error indication.  A new GET-ERROR-DETAILS request
has been added for applications that wish to get detailed error information.
See the GET-ERROR-DETAILS request below.

## Switching to the new protocol

The NABU application switches to the new protocol by sending a
START-NHACP message with the following format:

| Name    | Type    | Notes                          |
|---------|---------|--------------------------------|
| type    | u8      | 0x8f                           |
| magic   | char[3] | "ACP"                          |
| version | u16     | Version number of the protocol |

If the network adapter supports the new protocol, it will response with
a PROTOCOL-STARTED response (see below) and will then wait for futher messages
in the new framing format.  If the client is using a protocol version greater
than what the network adapter supports, the network adapter SHOULD ignore
the message.

In the original draft of NHACP, the NABU application switched to the new
protocol by sending a single byte 0xaf to the network adapter.  Unfortunately,
there were two issues with this:

* The message byte 0xaf later collided with a different NABU network adapter
  protocol extension.
* The message had no way to convey any protocol versioning information to
  the server.

Due to its use of a multi-byte sequence, The new START-NHACP message is
much less likely to collide with another message, and includes versioning
information.

Servers MAY wish to support the original 0xaf message type for NHACP; such
a server MUST use the original framing mode if this message is used to start
NHACP.

## Protocol versioning

The PROTOCOL-STARTED response contains a protocol version field.
The initial version of that field was undefined and 0 by convention.
This document describes version **0.1** of the protocol.  This
section describes the changes between protocol revisions.  New
revisions are backward-compatible with previous clients, to the
extent possible; any possible compatibility issues are called out
explicitly here.

* Version 0.1
    * Defined the protocol versioning convention.
    * Defined the new START-NHACP message.
    * Defined the initial set of error codes.
    * Defined new ERROR response behavior and GET-ERROR-DETAILS request.
* Version 0.0 - Initial version

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

No response message is returned by the network adapter.  If the server
receives a slot that is not currently in use by the client, the request
is simply ignored.

### GET-ERROR-DETAILS

Request details for the most recent ERROR response from the network
adapter.

| Name            | Type | Notes                                           |
|-----------------|------|-------------------------------------------------|
| type            | u8   | 0x06                                            |
| code            | u16  | Last error code returned by the network adapter |
| max-message-len | u8   | Maximum message length to return                |

Possible responses: ERROR

A network adapter MAY choose to remember the last error returned to an
application, in which case if the error code in the request matches the
saved error code, a detailed description of the error will be returned
to the application.  If the network adapter does not remember the last
error returned to an application, or if the error code does not match
the last remembered error code, then the network adapter MUST return
a generic error description string for the error code in the request.
In either case, any saved error code is cleared.

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

The returned version is a single 16-bit unsigned integer that encodes
the protocol version in a way that is directly arithmetically comparable.
This is intended to make protocol version checking simple for the client.
As a convention, the "major" and "minor" versions of the protocol are
kept in the most-significant and last-significant 8 bits of the version
field, respectively.  However, this is merely a convention and the
version values defined here are authoritative:

| Value  | Protocol version    |
|--------|---------------------|
| 0x0000 | initial NHACP draft |
| 0x0001 | NHACP version 0.1   |

### OK

| Name | Type | Notes |
|------|------|-------|
| type | u8   | 0x81  |

### ERROR

| Name           | Type      | Notes                   |
|----------------|-----------|-------------------------|
| type           | u8        | 0x82                    |
| code           | u16       | Error code              |
| message-length | u8        | Length of error message |
| message        | char[255] | Error message           |

Note that for all requests other than GET-ERROR-DETAILS, the
message-length field will be 0.

The following error codes are defined:

| Name      | Value | Notes                         |
|-----------|-------|-------------------------------|
| undefined | 0     | undefined generic error       |
| ENOTSUP   | 1     | Operation is not supported    |
| EEOF      | 2     | End-of-file                   |
| EPERM     | 3     | Operation is not permitted    |
| ENOENT    | 4     | Requested file does not exist |
| EIO       | 5     | Input/output error            |
| EBADF     | 6     | Bad file descriptor           |
| ENOMEM    | 7     | Out of memory                 |
| EACCES    | 8     | Access denied                 |
| EBUSY     | 9     | File is busy                  |
| EEXIST    | 10    | File already exists           |
| EISDIR    | 11    | File is a directory           |
| EINVAL    | 12    | Invalid argument/request      |
| ENFILE    | 13    | Too many open files           |
| EFBIG     | 14    | File is too large             |
| ENOSPC    | 15    | Out of space                  |
| ESEEK     | 16    | Seek on non-seekable file     |

All other error codes are reserved.

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