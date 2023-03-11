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
implementations using 2 stop bits (for a total of 11 bits per byte),
10169 bytes is the practical limit that can be transmitted within the
allotted time.

The maximum length of an NHACP message is defined to be 8256 bytes.  This
length was chosen for the following reasons:

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
| options | u16     | Protocol options               |

N.B. This request is sent while the network adapter is in legacy mode and
thus DOES NOT use NHACP message framing.

If the network adapter supports the new protocol, it will response with
a NHACP-STARTED response (see below) and will then wait for futher messages
in the new framing format.  If the client is using a protocol version greater
than what the network adapter supports, the network adapter SHOULD ignore
the message and not enter NHACP mode.

In the original draft of NHACP, the NABU application switched to the new
protocol by sending a single byte 0xaf to the network adapter.  Unfortunately,
there were two issues with this approach:

* The message byte 0xaf later collided with a different NABU network adapter
  protocol extension and there was no simple way to distinguish between
  the NHACP start-up message and message from the other protocol extension.
* The message had no way to convey any protocol versioning information to
  the server.

Due to its use of a multi-byte sequence, The new START-NHACP message is
much less likely to collide with another message and includes versioning
information.

Servers MAY wish to support the original 0xaf message type for NHACP; such
a server MUST maintain compatibility with the original draft specification
if this message is used to start NHACP.

## Protocol versioning

The NHACP-STARTED response contains a protocol version field.
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
    * Defined the new STORAGE-GET-BLOCK and STORAGE-PUT-BLOCK requests.
    * Added section about complex aggregate types, and added definitions
      for the DATE-TIME structure and the FILE-ATTRS structure.
    * Redefined the DATE-TIME response message to use the DATE-TIME structure.
      Layout of the message is backwards-compatible.
    * Defined the new UINT8-VALUE, UINT16-VALUE, and UINT32-VALUE responses.
    * Defined the new FILE-READ, FILE-WRITE, and FILE-SEEK requests.
    * Defined the new LIST-DIR and GET-DIR-ENTRY requests and DIR-ENTRY
      response.
    * Renamed STORAGE-CLOSE to FILE-CLOSE.  The semantics of the operation
      are unchanged.
* Version 0.0 - Initial version

The protocol version field in the START-NHACP and NHACP-STARTED messages
is a single 16-bit unsigned integer that encodes the protocol version in
a way that is directly arithmetically comparable.  This is intended to
make protocol version checking simple for the client.  As a convention,
the "major" and "minor" versions of the protocol are kept in the most-
significant and last-significant 8 bits of the version field, respectively.
However, this is merely a convention and the version values defined here
are authoritative:

| Value  | Protocol version    |
|--------|---------------------|
| 0x0000 | initial NHACP draft |
| 0x0001 | NHACP version 0.1   |

## Protocol options

At this time, there are no protocol options defined.  All bits in the
START-NHACP options field are reserved for future use.  Network adapters
MUST ignore a START-NHACP request that specifies a protocol option not
supported by the network adapter.

## Complex aggregate types

In addition to the simple scalar types (u8, u16, u32) and simple aggregate
types (char/byte arrays), there are some complex aggregate types used in
NHACP that are shared by multiple request/response messages.

### DATE-TIME structure

| Name | Type    | Notes                           |
|------|---------|---------------------------------|
| date | char[8] | Current date in YYYYMMDD format |
| time | char[6] | Current time in HHMMSS format   |

### FILE-ATTRS structure

| Name      | Type      | Notes                       |
|-----------|-----------|-----------------------------|
| mtime     | DATE-TIME | Last file modification time |
| flags     | u16       | File attribute flags        |
| file-size | u32       | File size                   |

The following file attribute flags are defined.  All other flags are
reserved.

| Name | Value  | Notes                    |
|------|--------|--------------------------|
| RD   | 0x0001 | File is readable         |
| WR   | 0x0002 | File is writable         |
| DIR  | 0x0004 | File is a directory      |
| SPEC | 0x0008 | File is a "special" file |

A "special" file is a file that is not a regular file nor a directory.

## Request messages

### STORAGE-OPEN

Open a URL and assign a storage index to it.  The URL can be relative
or a file URL to open a local file.  Network adapters may implement
additional, nonstandard URL schemes.  Relative URLs are interpreted 
as files.  The flags field can be used to pass additional information
to the storage handler.  The meaning of the flags is not specified in
the protocol itself.

The network adapter will attempt to use the file descriptor specified by
the caller unless the caller specifies 0xff, in which case the network
adapter will attempt to allocate a file descriptor.  If the requested
file descriptor is already in-use by another file object, the STORAGE-OPEN
request MUST fail.

| Name       | Type  | Notes                                                                 |
|------------|-------|-----------------------------------------------------------------------|
| type       | u8    | 0x01                                                                  |
| index      | u8    | File descriptor to use for response (0xff => Network Adapter selects) |
| flags      | u16   | Flags to pass to storage handler                                      |
| url-length | u8    | Length of resource in bytes                                           |
| url        | char* | URL String                                                            |

The following flags are defined:

| Name      | Value  | Notes                                      |
|-----------|--------|--------------------------------------------|
| O_RDRW    | 0x0000 | Psuedo-flag; open the file read-write      |
| O_RDONLY  | 0x0001 | Open the file read-only                    |
| O_CREAT   | 0x0002 | Create the file if it does not exist       |
| O_EXCL    | 0x0004 | Return an error if the file already exists |

All other flag values are reserved.

Note that O_EXCL has no effect if O_CREAT is not specified in the
request.

While NHACP-0.0 had a slot allocated in the STORAGE-OPEN request for flags,
it did not define any flags.  Network adapters that support NHACP-0.0
SHOULD ignore the flags field and assume O_RDRW+O_CREAT for connections
in NHACP-0.0 mode.

Possible responses: STORAGE-LOADED, ERROR

### STORAGE-GET

Get data from network adapter storage.

The maximum payload langth for a STORAGE-GET is 8192 bytes.  Network
adapters MUST return an error for STORAGE-GET requests whose length
field exceeds this value.

| Name   | Type | Notes                            |
|--------|------|----------------------------------|
| type   | u8   | 0x02                             |
| index  | u8   | File descriptor to access        |
| offset | u32  | Offset into the storage in bytes |
| length | u16  | Number of bytes to return        |

Possible responses: DATA-BUFFER, ERROR

The length returned in the DATA-BUFFER response reflects
the amount of data actuallty read from the underlying
file object.  If the offset is beyond the object's
end-of-file, then the returned length MUST be 0.
If the read operation would cross the object's end-of-file,
then the length MUST be the number of bytes read before
the end-of-file was encountered.

If the underlying file object cannot be accessed at arbitrary offsets,
then STORAGE-GET MUST fail with an ESEEK error.

### STORAGE-PUT

Update data stored in the network adapter.  If possible, the
underlying storage (file/URL) should be updated as well.

The maximum payload langth for a STORAGE-PUT is 8192 bytes.  Network
adapters MUST return an error for STORAGE-PUT requests whose length
field exceeds this value.

| Name   | Type | Notes                            |
|--------|------|----------------------------------|
| type   | u8   | 0x03                             |
| index  | u8   | File descriptor to access        |
| offset | u32  | Offset into the storage in bytes |
| length | u16  | Number of bytes to write         |
| data   | u8*  | Data to update the storage with  |

Possible responses: OK, ERROR

If a write originates at or beyond the underlying file
object's end-of-file or therwise crosses the end-of-file,
then the underlying file object SHOULD be implicitly
enlarged to accommodate the write.  For writes that originate
beyond end-of-file, the region between the old end-of-file and
the newly-written region MUST be implicitly zero-filled.  If
a server implementation does not support extending the underlying
file object, then the server MUST return an error without
performing the write operation.

If the underlying file object cannot be accessed at arbitrary offsets,
then STORAGE-PUT MUST fail with an ESEEK error.

### GET-DATE-TIME

Retrieve the current date and time from the network adapter.

| Name | Type | Notes |
|------|------|-------|
| type | u8   | 0x04  |

Possible responses: DATE-TIME, ERROR

### FILE-CLOSE

Close a previously opened file descriptor.  Any resources associated with
it on the network adapter will be freed.

| Name  | Type | Notes                    |
|-------|------|--------------------------|
| type  | u8   | 0x05                     |
| index | u8   | File descriptor to close |

No response message is returned by the network adapter.  If the server
receives a file descriptor that is not currently in use by the client,
the request is simply ignored.

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

### STORAGE-GET-BLOCK

Get a block of data from network adapter storage.  This is an optimization
of STORAGE-GET designed to support block-device storage.  Rather than a
byte offset and length, a block number and block length is specified.  The
byte offset into network adapter storage is computed by the network
adapter as:

	offset = block-number * block-length

STORAGE-GET-BLOCK operates on atomic units; partial reads are not allowed.
if the underlying file object is not large enough to satisfy the entire
request, the network adapter MUST return an error.

The maximum payload langth for a STORAGE-GET-BLOCK is 8192 bytes.  Network
adapters MUST return an error for STORAGE-GET-BLOCK requests whose block
length field exceeds this value.

| Name         | Type | Notes                            |
|--------------|------|----------------------------------|
| type         | u8   | 0x07                             |
| index        | u8   | File descriptor to access        |
| block-number | u32  | 0-based index of block to access |
| block-length | u16  | Length of the block              |

Possible responses: DATA-BUFFER, ERROR

The length returned in the DATA-BUFFER response MUST equal
the block size in the request.

If the underlying file object cannot be accessed at arbitrary offsets,
then STORAGE-GET-BLOCK MUST fail with an ESEEK error.

### STORAGE-PUT-BLOCK

Put a block of data to network adapter storage.  This is an optimization
of STORAGE-PUT designed to support block-device storage.  Rather than a
byte offset and length, a block number and block length is specified.  The
byte offset into network adapter storage is computed by the network
adapter as:

	offset = block-number * block-length

STORAGE-PUT-BLOCK operates on atomic units; partial writes are not allowed,
nor is extending the underlying file object.  If the underlying file
object is not large enough to satisfy the entire request, the network adapter
MUST return an error.

The maximum payload langth for a STORAGE-PUT-BLOCK is 8192 bytes.  Network
adapters MUST return an error for STORAGE-PUT-BLOCK requests whose block
length field exceeds this value.

| Name         | Type | Notes                            |
|--------------|------|----------------------------------|
| type         | u8   | 0x08                             |
| index        | u8   | File descriptor to access        |
| block-number | u32  | 0-based index of block to access |
| block-length | u16  | Length of the block              |

Possible responses: OK, ERROR

If the underlying file object cannot be accessed at arbitrary offsets,
then STORAGE-PUT-BLOCK MUST fail with an ESEEK error.

### FILE-READ

Read data sequentially from network adapter storage.

The maximum payload langth for a FILE-READ is 8192 bytes.  Network
adapters MUST return an error for FILE-READ requests whose length
field exceeds this value.

| Name   | Type | Notes                            |
|--------|------|----------------------------------|
| type   | u8   | 0x09                             |
| index  | u8   | File descriptor to access        |
| flags  | u16  | Flags                            |
| length | u16  | Number of bytes to return        |

Possible responses: DATA-BUFFER, ERROR

There are no flags currently defined for FILE-READ; all values are
reserved.

The length returned in the DATA-BUFFER response reflects
the amount of data actuallty read from the underlying
file object.  If the file cursor is beyond the object's
end-of-file, then the returned length MUST be 0.
If the read operation would cross the object's end-of-file,
then the length MUST be the number of bytes read before
the end-of-file was encountered.

### FILE-WRITE

Perform a sequential write to update data stored in the network adapter.
If possible, the underlying storage (file/URL) should be updated as well.

The maximum payload langth for a FILE-WRITE is 8192 bytes.  Network
adapters MUST return an error for FILE-WRITE requests whose length
field exceeds this value.

| Name   | Type | Notes                            |
|--------|------|----------------------------------|
| type   | u8   | 0x0a                             |
| index  | u8   | File descriptor to access        |
| flags  | u16  | Flags                            |
| length | u16  | Number of bytes to write         |
| data   | u8*  | Data to update the storage with  |

Possible responses: OK, ERROR

There are no flags currently defined for FILE-WRITE; all values are
reserved.

If the position of the file cursor causes the write to originate at or
beyond the underlying file object's end-of-file or therwise crosses
the end-of-file, then the underlying file object SHOULD be implicitly
enlarged to accommodate the write.  For writes that originate beyond
end-of-file, the region between the old end-of-file and the newly-written
region MUST be implicitly zero-filled.  If a server implementation does
not support extending the underlying storage object, then the server MUST
return an error without performing the write operation.

### FILE-SEEK

Re-position the file cursor by a signed offset relative to the specified
seek origin.

| Name   | Type | Notes                            |
|--------|------|----------------------------------|
| type   | u8   | 0x0b                             |
| index  | u8   | File descriptor to access        |
| offset | s32  | Offset relative to seek origin   |
| whence | u8   | The origin of the seek           |

The following whence values are defined:

| Name      | Value | Notes                                                    |
|-----------|-------|----------------------------------------------------------|
| SEEK_SET  | 0x00  | Set the cursor to offset bytes.                          |
| SEEK_CUR  | 0x01  | Set the cursor to the current location plus offset bytes |
| SEEK_END  | 0x02  | Set the cursor to the size of the file plus offset bytes |

Once the cursor has been repositioned, FILE-SEEK returns the resulting offset
measured in bytes from the beginning of the file.

If the underlying file object does not have a cursor that can be respositioned,
then FILE-SEEK MUST fail with an ESEEK error.

Possible responses: UINT32-VALUE, ERROR

### LIST-DIR

This request causes the network adapter to scan a directory for file names
matching the specified pattern, retrieve the attributes of those files,
and cache a list of those files to be retrieved one file at a time by future
requests.  The directory must have already been opened.  Any cached listing
is released upon a subsequent LIST-DIR or FILE-CLOSE request for that
file descriptor.

| Name           | Type  | Notes                                   |
|----------------|-------|-----------------------------------------|
| type           | u8    | 0x0c                                    |
| index          | u8    | File descriptor of directory to list    |
| pattern-length | u8    | Legth of the file name matching pattern |
| pattern        | char* | File name matching pattern              |

Possible responses: OK, ERROR

The network adapter SHOULD implement file name matching compatible with
the IEEE Std 1003.2 for the glob() function.

### GET-DIR-ENTRY

Returns the next directory entry cached by a LIST-DIR request and
advances the directory cursor.

| Name            | Type  | Notes                                             |
|-----------------|-------|---------------------------------------------------|
| type            | u8    | 0x0d                                              |
| index           | u8    | File descriptor of directory that has been listed |
| max-name-length | u8    | Maximum length of the returned file name          |

Possible responses: OK, DIR-ENTRY, ERROR

If there are no more entries to return, or if the listing has not been
generated, the server MUST repond with OK to indicate the end-of-directory.

The name of the file in the returned directory entry MUST be truncated to
the max-name-length specifed by the application.

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
| EPERM     | 2     | Operation is not permitted    |
| ENOENT    | 3     | Requested file does not exist |
| EIO       | 4     | Input/output error            |
| EBADF     | 5     | Bad file descriptor           |
| ENOMEM    | 6     | Out of memory                 |
| EACCES    | 7     | Access denied                 |
| EBUSY     | 8     | File is busy                  |
| EEXIST    | 9     | File already exists           |
| EISDIR    | 10    | File is a directory           |
| EINVAL    | 11    | Invalid argument/request      |
| ENFILE    | 12    | Too many open files           |
| EFBIG     | 13    | File is too large             |
| ENOSPC    | 14    | Out of space                  |
| ESEEK     | 15    | Seek on non-seekable file     |
| ENOTDIR   | 16    | File is not a directory       |

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

| Name      | Type      | Notes                 |
|-----------|-----------|-----------------------|
| type      | u8        | 0x85                  |
| date_time | DATE-TIME | Current date and time |

### DIR-ENTRY

| Name        | Type       | Notes                        |
|-------------|------------|------------------------------|
| type        | u8         | 0x86                         |
| attrs       | FILE-ATTRS | File attributes              |
| name-length | u8         | Length of returned file name |
| name        | char*      | File name                    |

### UINT8-VALUE

| Name  | Type | Notes               |
|-------|------|---------------------|
| type  | u8   | 0x87                |
| value | u8   | Generic 8-bit value |

### UINT16-VALUE

| Name  | Type | Notes                |
|-------|------|----------------------|
| type  | u8   | 0x88                 |
| value | u16  | Generic 16-bit value |

### UINT32-VALUE

| Name  | Type | Notes                |
|-------|------|----------------------|
| type  | u8   | 0x89                 |
| value | u32  | Generic 32-bit value |

## Recovering from a crash

When a NABU application crashes while the modern protocol is selected,
the network adapter needs to detect that the NABU is restarting.  As
the NABU ROM sends at least 4 "SET STATUS (0x83)" messages during
startup, the network adapter can assume that if it receives a length
field of 0x8383, it needs to restart in legacy protocol mode in order
to boot the NABU.

## Example exchanges

The following are examples of NHACP request / response exchanges.
In each column, the message starts with the message name, then the
byte stream, including the frame length bytes, that comprises the
example message being sent.  Double-quotes denote ASCII strings,
which are used for readability.

### Establishing an NHACP connection

| Application | Network Adapter    |
|-------------|--------------------|
| START-NHACP |                    |
| 0x8f        |                    |
| "ACP"       |                    |
| 0x01 0x00   |                    |
|             | NHACP-STARTED      |
|             | 0x14 0x00          |
|             | 0x80               |
|             | 0x01 0x00          |
|             | 0x10               |
|             | "NABU-ADAPTOR-1.1" |

### Opening a 1KB file, reading 1KB of data, closing the file

| Application         | Network Adapter      |
|---------------------|----------------------|
| STORAGE-OPEN        |                      |
| 0x0f 0x00           |                      |
| 0x01                |                      |
| 0xff                |                      |
| 0x00 0x00           |                      |
| 0x0a                |                      |
| "LEVEL1.DAT"        |                      |
|                     | STORAGE-LOADED       |
|                     | 0x06 0x00            |
|                     | 0x83                 |
|                     | 0x00                 |
|                     | 0x00 0x04 0x00 0x00  |
| STORAGE-GET         |                      |
| 0x08 0x00           |                      |
| 0x02                |                      |
| 0x00                |                      |
| 0x00 0x00 0x00 0x00 |                      |
| 0x00 0x04           |                      |
|                     | DATA-BUFFER          |
|                     | 0x03 0x04            |
|                     | 0x84                 |
|                     | 0x00 0x04            |
|                     | <1024 bytes of data> |
| FILE-CLOSE          |                      |
| 0x02 0x00           |                      |
| 0x05                |                      |
| 0x00                |                      |

### Opening a file, error return, getting error details

| Application       | Network Adapter                    |
|-------------------|------------------------------------|
| STORAGE-OPEN      |                                    |
| 0x0a 0x00         |                                    |
| 0x01              |                                    |
| 0xff              |                                    |
| 0x00 0x00         |                                    |
| 0x05              |                                    |
| "C.DSK"           |                                    |
|                   | ERROR                              |
|                   | 0x04 0x00                          |
|                   | 0x82                               |
|                   | 0x04 0x00                          |
|                   | 0x00                               |
| GET-ERROR-DETAILS |                                    |
| 0x04 0x00         |                                    |
| 0x06              |                                    |
| 0x04 0x00         |                                    |
| 0x40              |                                    |
|                   | ERROR                              |
|                   | 0x24 0x00                          |
|                   | 0x82                               |
|                   | 0x04 0x00                          |
|                   | 0x20                               |
|                   | "C.DSK: no such file or directory" |
