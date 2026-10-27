# Challenge 3 — OPC Scout

**Protocol:** OPC-UA (likely) · **Port:** TBD · **Toolkit module:** none yet

---

> **Note:** unlike the other six, I don't have any confirmed details for this challenge from our
> session — no target port, node structure, or toolkit support has come up yet. The `sysrupt`
> toolkit's four modules (`modbus`, `iec104`, `tristation`, `goose`) don't cover OPC-UA, so this
> one likely needs an external client (e.g. a Python `opcua`/`asyncua` client, or `opcua-commander`
> for interactive browsing) rather than `sysrupt` itself. The structure below is a placeholder —
> tell me the actual port, endpoint URL, and what the badge/HMI says about this challenge and I'll
> fill in real steps and hints instead of a generic template.

## The Story

*(fill in once known)*

## What You Know

- *(target host:port for the OPC-UA endpoint)*
- *(any security policy / anonymous vs. authenticated access noted on the badge)*

## Step 1: Discover the endpoint

<details>
<summary>Hint</summary>

OPC-UA servers expose their own endpoint description — you don't need to guess the security
mode, you can ask the server what it supports.
</details>

<details>
<summary>Show command</summary>

```bash
# placeholder — needs an OPC-UA client, e.g.:
python3 -m asyncua.tools.uaclient opc.tcp://10.0.1.1:<PORT>
```
</details>

## Step 2: Browse the address space

<details>
<summary>Hint</summary>

Walk the node tree from the Objects folder down rather than guessing node IDs.
</details>

<details>
<summary>Show command</summary>

*(to fill in once the node structure is known)*
</details>

## What You Learned

*(fill in once solved)*
