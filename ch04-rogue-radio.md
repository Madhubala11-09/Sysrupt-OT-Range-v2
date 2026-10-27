# Challenge 4 — Rogue Radio

**Protocol:** Custom binary TCP protocol  
**Gateway:** `10.0.1.1:7700`  
**Toolkit module:** None  
**Difficulty focus:** Protocol reverse engineering, sequence numbers, anti-replay, command sequencing, and state verification

## The Story

The flow-control pump bus still communicates through an older radio gateway. Although the gateway uses a custom binary protocol, it does not require authentication.

A saved packet capture from normal traffic reveals the frame structure, checksum rule, and which feeder corresponds to the real flow valve rather than a decoy.

However, replaying an old command byte-for-byte does not work. The gateway enforces sequence-based anti-replay protection, so a valid command must use an acceptable, fresh sequence number.

The challenge is to identify the correct feeder, understand the command sequence, and verify the valve's actual state through the gateway's status readback.

> **Safety:** Perform all experiments only against the isolated Sysrupt training environment. Do not send control frames to production equipment or a real pump bus.

## What You Know

An email from the Pump & Water instrumentation team provides the following clues:

- The gateway is reachable at `10.0.1.1:7700`.
- The protocol is custom binary TCP, not ordinary HTTP.
- The protocol has a checksum and a sequence-based anti-replay mechanism.
- A saved capture contains normal traffic and enough information to reconstruct the frame format.
- The feeder identifier distinguishes the real flow valve from decoys.
- The status/flag block is read on the same TCP socket as a command.
- The status request uses action byte `0x05`, with the target represented as a word index.
- The status remains zero until the target valve has actually been selected, opened, and held open.

## Step 1: Identify the Gateway

Check whether the gateway is reachable:

```bash
nc -vz 10.0.1.1 7700
```

If the connection succeeds, the TCP port is reachable. This does not, by itself, prove that the application protocol is understood or that a command will be accepted.

## Step 2: Inspect the Saved Capture

Start with the supplied `pump_capture.txt` attachment or the corresponding capture provided by the challenge.

For a packet capture file, open it in Wireshark and use this display filter:

```text
tcp.port == 7700
```

Inspect both directions of the TCP conversation. Record:

- The bytes sent by the client.
- The gateway's response to each request.
- Which byte changes between successive commands.
- The feeder identifier used in successful traffic.
- The checksum calculation.
- The response format for accepted and rejected commands.
- Whether the gateway maintains state between requests.

Use **Follow → TCP Stream** to view the conversation in sequence. For binary traffic, select hexadecimal display rather than interpreting the payload as text.

## Step 3: Understand the Frame Structure

The supplied solving script proposes the following eight-byte frame:

| Byte offset | Field | Meaning |
|---:|---|---|
| `0` | `0x50` | Frame marker |
| `1` | `0x55` | Frame marker |
| `2` | `0x01` | Command frame type |
| `3` | Sequence | Command sequence number |
| `4` | Feeder | Target feeder identifier |
| `5` | Command | Requested action |
| `6` | `0x00` | Reserved or fixed field |
| `7` | Checksum | Sum of the preceding seven bytes, modulo 256 |

The proposed checksum is:

```python
def checksum(data):
    return sum(data) & 0xFF
```

The command-frame builder is:

```python
def cmd_frame(seq, feeder, cmd):
    body = bytes([
        0x50, 0x55, 0x01,
        seq, feeder, cmd, 0x00
    ])
    return body + bytes([checksum(body)])
```

**Important:** Treat this structure as a hypothesis until it has been confirmed against the supplied capture. In particular, verify the checksum rule, reserved byte, and response format rather than assuming every frame follows the same layout.

## Step 4: Determine the Correct Feeder

The email explicitly warns that one feeder controls the real flow valve while others are decoys.

Do not select a feeder merely because the gateway acknowledges its command. An acknowledgement establishes that a request was accepted; it does not necessarily establish that the intended physical or simulated target was selected.

Correlate the feeder identifier with the normal traffic in the capture and the challenge's inventory or documentation.

Record the evidence supporting the selected feeder before proceeding.

## Step 5: Understand Sequence-Based Anti-Replay

The gateway rejects byte-for-byte replay of an old command. The sequence number therefore matters independently of the feeder and command fields.

The supplied script begins with sequence `0xEC` and tests successive byte values, wrapping from `0xFF` to `0x00`. It pauses briefly between attempts.

This is an observation about the supplied script, not proof that every sequence value is valid or that the gateway always accepts the first acknowledged request.

When investigating sequence behavior, distinguish these cases:

1. A malformed frame or invalid checksum.
2. A valid frame with an unacceptable sequence number.
3. A valid frame with a fresh sequence number but an incorrect feeder or action.
4. A command that is accepted but does not produce the expected state change.

Use gateway responses and the captured traffic to determine which case applies.

## Step 6: Understand the Command Sequence

The solving script identifies an acknowledged command with action byte `0x01` as the selection step. It then advances the sequence number for the next command.

The challenge email indicates that the target must be selected, opened, and held open before the status/flag block reports a nonzero value.

Do not assume that the next action byte or its required timing can be inferred from the sequence increment alone. Confirm the command meanings and expected transitions from the capture and challenge evidence.

A useful state checklist is:

- [ ] Correct feeder identified.
- [ ] Selection acknowledged.
- [ ] Next command uses the expected sequence.
- [ ] Open state confirmed by readback.
- [ ] Valve remains open for the required interval.
- [ ] Status/flag block read using action byte `0x05` and the appropriate word index.
- [ ] Final state verified from the gateway response.

## Step 7: Verify State, Not Just Acknowledgements

The status/flag block is returned on the same TCP connection and uses the same frame shape as a command.

The documented action byte is `0x05`; its target field represents a word index.

A zero status does not necessarily mean the protocol is broken. According to the challenge email, the flag remains zero until the target valve has been selected, opened, and held open.

Use the readback to distinguish a successfully accepted command from a successfully completed state transition. Preserve the request and response bytes so that each conclusion can be checked against the capture.

## What You Learned

This challenge illustrates several security lessons relevant to legacy OT gateways:

- **No authentication:** Network reachability may be enough to interact with the protocol.
- **Custom protocol exposure:** Binary protocols require careful analysis of frame boundaries, field meanings, and response behavior.
- **Anti-replay limitations:** Sequence numbers can prevent simple replay without providing strong authentication.
- **Target ambiguity:** Decoy devices or feeder identifiers can make an acknowledged command misleading.
- **Stateful control:** A command acknowledgement is not the same as a verified process state.
- **Readback importance:** Status responses help establish whether the requested transition actually occurred.

## Defensive Takeaways

For a real OT deployment:

- Restrict gateway access to explicitly authorized control systems.
- Authenticate commands and protect their integrity.
- Use robust anti-replay mechanisms rather than relying solely on predictable sequence numbers.
- Validate target identifiers and command transitions on the gateway.
- Enforce safety interlocks at the device that performs the action.
- Log rejected requests, accepted commands, and verified state transitions.
- Keep training gateways isolated from production control networks.

## References

- Challenge email: **Pump-bus radio gateway on :7700 — saved capture**
- Attachment: `pump_capture.txt`
- Gateway: `10.0.1.1:7700`
