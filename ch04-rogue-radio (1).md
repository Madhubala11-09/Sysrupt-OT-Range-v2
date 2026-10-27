# Challenge 4 — Rogue Radio

**Protocol:** Custom binary protocol over TCP  
**Target:** `10.0.1.1:7700`  
**Toolkit module:** None  
**Focus:** Binary frame analysis, checksum validation, anti-replay sequence numbers, feeder identification, and state readback

---

## The Story

The flow-control pump bus still uses an old radio gateway. The gateway exposes a custom binary protocol over TCP and does not require authentication.

A saved capture of normal traffic reveals the frame format, checksum rule, and feeder identifier for the real flow valve rather than a decoy. However, replaying a captured command byte-for-byte is rejected because the gateway uses sequence-based anti-replay protection.

The objective is to understand the protocol, identify the correct feeder, perform the required state transitions, and verify the resulting status through readback.

> **Safety:** Only run these steps against the isolated Sysrupt training range. Never send control frames to a production gateway or real pump bus.

## What You Know

The challenge email, **“Pump-bus radio gateway on :7700 — saved capture,”** states:

- Target: `10.0.1.1:7700`
- Protocol: custom binary TCP, not HTTP
- The supplied `pump_capture.txt` capture contains normal traffic.
- The checksum rule and feeder mapping can be inferred from the capture.
- Byte-for-byte replay is rejected; a fresh, higher sequence number is required.
- The status/flag block is read on the same socket using action byte `0x05`.
- The status target field represents a word index.
- The status remains zero until the target valve has been selected, opened, and held open.

## Step 1: Check the Gateway

<details>
<summary>Hint</summary>

First confirm that the TCP service is reachable. This does not prove that the protocol is understood or that commands will be accepted.

</details>

<details>
<summary>Show command</summary>

```bash
nc -vz 10.0.1.1 7700
```

</details>

## Step 2: Inspect the Saved Capture

<details>
<summary>Hint</summary>

Use Wireshark to inspect both directions of the TCP conversation. Follow the TCP stream, then inspect the payload bytes in hexadecimal. Focus on frames that receive different responses.

</details>

<details>
<summary>Show command</summary>

```text
tcp.port == 7700
```

In Wireshark:

1. Apply the display filter above.
2. Select a packet belonging to the gateway conversation.
3. Right-click and choose **Follow → TCP Stream**.
4. Switch to hexadecimal/raw display if the payload is not readable text.
5. Record the request bytes and the corresponding response bytes.

</details>

## Step 3: Identify the Frame Layout

<details>
<summary>Hint</summary>

The supplied solving script proposes an eight-byte command frame. Compare several captured frames to determine which fields remain fixed and which change between commands.

</details>

<details>
<summary>Show command</summary>

```python
def checksum(data):
    return sum(data) & 0xff

def cmd_frame(seq, feeder, cmd):
    body = bytes([
        0x50, 0x55, 0x01,
        seq, feeder, cmd, 0x00
    ])
    return body + bytes([checksum(body)])
```

Proposed frame layout:

| Offset | Field |
|---:|---|
| 0 | `0x50` frame marker |
| 1 | `0x55` frame marker |
| 2 | `0x01` command-frame type |
| 3 | sequence number |
| 4 | feeder identifier |
| 5 | command/action |
| 6 | reserved/fixed byte (`0x00` in the supplied script) |
| 7 | checksum: sum of the preceding bytes modulo 256 |

Confirm this layout and checksum against the supplied capture rather than treating the script as proof of every protocol detail.

</details>

## Step 4: Identify the Real Feeder

<details>
<summary>Hint</summary>

The capture distinguishes the real flow valve from decoy feeders. An acknowledgement alone does not prove that the correct target was selected.

</details>

<details>
<summary>Show command</summary>

Review the supplied capture and compare the feeder byte at offset 4 in frames associated with known normal traffic. Correlate it with the challenge's feeder descriptions or inventory before issuing state-changing commands.

The example script uses feeder `0x07`, but verify that this is the intended target in your instance.

</details>

## Step 5: Understand Sequence-Based Anti-Replay

<details>
<summary>Hint</summary>

A previously captured frame may be structurally valid but rejected because its sequence number is stale. Observe how the sequence changes in normal traffic and compare accepted versus rejected responses.

</details>

<details>
<summary>Show command</summary>

The supplied script's sequence-search logic is:

```python
seq = 0xec

while not send_and_check(seq, 0x07, 0x01):
    seq += 1
    if seq > 0xff:
        seq = 0x00
    time.sleep(0.05)
```

Here, `send_and_check()` constructs a frame, sends it, prints the request and response in hexadecimal, and treats a response beginning with `50 55 00` as an acknowledgement.

This is the script's observed acknowledgement test, not a universal rule for other gateways. Use the captured responses to understand the response format in this challenge.

</details>

## Step 6: Follow the Required State Sequence

<details>
<summary>Hint</summary>

The email says the target valve must be selected, opened, and held open. The supplied script shows an acknowledged SELECT action (`0x01`) and then increments the sequence for the next command, but the excerpt does not show the complete OPEN/hold implementation.

Do not guess the remaining command bytes. Confirm them from the capture and the challenge's protocol evidence.

</details>

<details>
<summary>Show command</summary>

The known selection-frame construction is:

```python
seq = 0xec
frame = cmd_frame(seq, 0x07, 0x01)
print(frame.hex())
```

The values above reproduce the example in the supplied script. Verify the correct feeder and sequence for your own target. The exact OPEN and hold commands must be established from the provided capture or challenge-specific evidence; they are not included in the supplied script excerpt.

</details>

## Step 7: Read Back the Status/Flag Block

<details>
<summary>Hint</summary>

The second email says the status/flag block is read on the same socket using action byte `0x05`. Its target field is a word index. The status will remain zero until the required valve state has actually been reached and held.

</details>

<details>
<summary>Show command</summary>

Use the frame structure and response behavior established from the capture to construct the status request with action byte `0x05` and the correct word index. Keep the same TCP connection open as required by the protocol.

The supplied snippet does not include the exact status-request helper or word index, so those values must be confirmed from the capture rather than invented.

</details>

## Troubleshooting

- **Connection refused or timeout:** confirm the lab is running and the target/port are correct.
- **No response:** confirm you are sending binary frames to the TCP service, not HTTP requests.
- **Checksum rejected:** verify the checksum against known captured frames, including whether the checksum covers the first seven bytes.
- **Command rejected:** check the sequence number, feeder identifier, action byte, and response status separately.
- **Acknowledged but no status change:** an acknowledgement is not proof that the target reached the desired state.
- **Status remains zero:** verify that the correct feeder was selected, the valve was opened, and the required hold interval was met.
- **Old frame rejected:** this is consistent with the anti-replay behavior described in the challenge email.

## What You Learned

- Custom binary protocols can be reverse-engineered by comparing normal requests and responses.
- Checksums detect accidental or malformed data but do not authenticate the sender.
- Sequence-based anti-replay can stop simple replay while still being weaker than cryptographic authentication.
- Decoy identifiers make it important to verify the intended target.
- Readback is essential: accepted commands and confirmed process state are different things.

## Defensive Takeaways

- Restrict gateway access to authorized control systems.
- Authenticate commands and protect message integrity.
- Use robust anti-replay protection instead of relying only on predictable sequence numbers.
- Validate feeder identifiers and allowed state transitions at the gateway.
- Enforce safety interlocks at the device that performs the action.
- Log rejected requests, accepted commands, and verified state changes.
