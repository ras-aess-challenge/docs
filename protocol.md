# Beacon and mission protocol

This describes the current implementation, not a proposed new wire format.
Writer and executor connect to the strong node over TCP and send one request per
connection. Responses are read until the server closes the socket. All protocol
peers must use the same UTF-8 SHARED_SECRET.

## Authenticated messages

```text
request = type_byte + payload + HMAC_SHA256(secret, type_byte + payload)
```

| Message | Type byte | Payload | HMAC | Total bytes |
| --- | --- | --- | --- | --- |
| Beacon | 0x01 | 16 bytes | 32 bytes | 49 |
| Mission request | 0x02 | Empty | 32 bytes | 33 |

SHA-256 HMAC is a binary digest, not a hexadecimal string. Server verification
uses constant-time digest comparison. TCP segmentation does not imply message
boundaries: the relay reads the exact expected byte count.

## Beacon payload layout

| Byte offsets | Size | Field | Encoding |
| --- | --- | --- | --- |
| 0–3 | 4 | timestamp | Unsigned big-endian Unix seconds, reduced modulo 2^32 by packing |
| 4–6 | 3 | x | Signed big-endian centimetres, rounded from metres |
| 7–9 | 3 | y | Signed big-endian centimetres, rounded from metres |
| 10–11 | 2 | event | Unsigned big-endian event code |
| 12–15 | 4 | pod | Big-endian IEEE-754 float32 |

Event codes: NONE=0, VICTIM=1, HAZARD=2, DEBRIS=3. The current encoder maps an
unrecognized event name to NONE; the server rejects unknown numeric event codes.
Coordinates must fit signed 24-bit centimetres. The server accepts PoD only in
[0, 1], timestamps no more than 300 seconds old, and future skew no more than
5 seconds. The replay key is timestamp seconds alone; accepted keys are held in
memory and reset on relay restart.

## Responses

Successful beacon storage returns `RECEIVED WN-<timestamp>`. Robot clients verify
this acknowledgement against the sent timestamp. Errors use UTF-8 strings prefixed
with `ERROR:`. Known rejection cases include authentication failure, replay,
stale/future timestamp, invalid PoD, unknown event/type, incomplete message and
unreachable command post. Robots fail their CLI job on an error response.

An authenticated mission request returns a UTF-8 JSON list of beacon records,
with EOF ending the response. An empty list is a valid retrieval response, but
the executor job fails because it cannot navigate without a target.

## Stored and returned record

Example values are illustrative:

```json
{
  "node_id": "WN-1790970865",
  "x": 10.5,
  "y": 0.0,
  "lat": 36.8065,
  "lon": 10.1816178,
  "timestamp": 1790970865,
  "event": "VICTIM",
  "pod": 0.9
}
```

The strong node assigns node_id using timestamp seconds and adds GPS coordinates
using the configured source-code anchor (36.8065, 10.1815). The current code uses
an equirectangular approximation for short distances. Float32 packing means the
returned PoD may differ slightly from its decimal input.

## Internal command-post protocol

The strong node sends either a UTF-8 JSON record or literal `GET_MISSION` to
`command-post:65433`. Requests are limited to 4096 bytes and accumulated across
TCP fragments. JSON writes require an object containing node_id; the endpoint
trusts its network peers for other fields. Successful atomic storage returns
`STORED`; write failure returns `ERROR: storage unavailable` without retaining the
new record in memory. Mission reads return the full JSON list until EOF.

This internal connection is unauthenticated and unpublished. The legacy relay in
Robots-Network speaks JSON directly and is not a compatible entry point for the
binary HMAC clients. HMAC protects authenticity, not confidentiality; see
[security.md](security.md). Changes to framing/IDs require a versioned protocol
rollout as described in [code-structure.md](code-structure.md).
