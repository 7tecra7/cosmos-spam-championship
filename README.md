# Cosmos called it Spam after Triage. Then they shipped the patch.

Cosmos wasmd IBC v2: 1 Critical and 2 High triaged, closed as Spam, then shipped in v0.70.0 / v0.61.11.

**Reports:** #3547293 (Critical), #3547382 (High), #3551078 (High)  
**Asset:** CosmWasm wasmd / IBC v2 (ibc-go v10)  
**Versions tested:** wasmd v0.61.7 / v0.61.8  
**Patched releases:** v0.61.11 and v0.70.0  
**HackerOne Triagers:** [@srdtrk](https://github.com/srdtrk) (`povim_serdar`), [@aljo242](https://github.com/aljo242) (`cozartshmoopler`)  
**Public GitHub:** [@pinosu](https://github.com/pinosu) authored and merged the commit and PRs below; [@DariuszDepta](https://github.com/DariuszDepta) approved them.

Three reports against Cosmos CosmWasm wasmd / IBC v2 were filed on 10–12 February 2026. A program reviewer reproduced all three and moved them to **Triaged**. On 2 March the program published new eligibility rules and, the same day, closed those already-Triaged reports as **Spam**.

On 18 March commit `56ca5e5` changed the async-ack path. On 23–24 April PRs #2478, #2479, #2480, and #2481 merged as one release batch (send path and async ack, main and v0.61.x). They shipped in v0.70.0 and v0.61.11.

On 4 May, after Request to Disclose, all three reports were locked. That batch got two lock stories. The Critical ticket was called an audit: “The changes made were related to audit fixes, not to your submissions.” The two High tickets that shipped in the same batch were not called an audit. Disclosure was cancelled as “threatening in direct violation of our policies.”

This article is the chronology and the architecture. It is not a proof of concept. It is not the August 2026 Cosmos EVM incident.

## Public markers

| Report | Severity | Class | Public markers |
|---|---|---|---|
| #3547293 | Critical | Async ack authorization | `56ca5e5` (18 Mar); #2480 main, #2481 v0.61.x (24 Apr) |
| #3547382 | High | Unvalidated SourcePort on send | #2478 main, #2479 v0.61.x (24 Apr) |
| #3551078 | High | V1 port into V2 handlers | #2478 / #2479 (`IBCPortID` → `IBC2PortID` on `OnSendIBC2Packet`) |

## Chronology

- **10–12 Feb 2026.** #3547293, #3547382, #3551078 filed against v0.61.7 and v0.61.8, each with ibctesting reproduction.
- **16 and 19 Feb.** [srdtrk](https://github.com/srdtrk) (`povim_serdar` on HackerOne) moved all three to **Triaged**.
- **2 Mar.** New eligibility rules published. [aljo242](https://github.com/aljo242) (`cozartshmoopler` on HackerOne) closed the already-Triaged reports as Spam. Spam is an eligibility closure and a Signal penalty. It is not a technical clearance of the defects.
- **Mar–Apr.** Escalated to HackerOne Support (Ticket #642104). Support recorded a conflict and left Spam in place.
- **18 Mar.** [`56ca5e5`](https://github.com/CosmWasm/wasmd/commit/56ca5e529c66b3fa0e18d2aa0915812ae249ba01) by @pinosu, message `fix`, async-ack path.
- **23–24 Apr.** @pinosu opened #2478, #2479, #2480, #2481 without PR bodies. @DariuszDepta labeled `g:fix`, approved `LGTM`, @pinosu merged:
  - #2478 Fix ibc2 packet send (main)
  - #2479 Fix ibc2 packet send, backport v0.61.x
  - #2480 Fix ibc2 async ack (main)
  - #2481 Fix ibc2 async ack, v0.61.x  
  v0.70.0 changelog: “Fix IBC v2 async acknowledgements” (#2480), “Fix IBC v2 packet send” (#2478). v0.61.11: #2481 and #2479.
- **Late Apr.** Those links were posted on the HackerOne threads. #3547293 was pointed at `56ca5e5`.
- **4 May.** Request to Disclose. [Cozartshmoopler](https://github.com/aljo242) locked all three the same day.  
  - **Critical (#3547293):** “The changes made were related to audit fixes, not to your submissions. We are locking this down.”  
  - **High (#3547382, #3551078):** “cancelling as you are threatening in direct violation of our policies.”
- **Sept 2026.** Final escalation attempt via HackerOne Support. Support stated that the CSM team reached out, but the program refused to change its stance due to "criteria related to signal" and "trial report attempts." Support declared no further action could be taken on their side.

## Signal Pretext and Disparate Treatment

HackerOne Support cited "signal criteria" and "trial report attempts" as the program's grounds for maintaining the Spam status. Public platform activity contradicts this pretext:

- **[knafeh](https://hackerone.com/knafeh):** Account joined in **June 2026** (months after the February submissions). The Cosmos program accepted their report as resolved and awarded a **$12,500** bounty.
- **[wataha](https://hackerone.com/wataha):** Account joined in **February 2026** with a negative signal rating (**-3.60**, 32nd percentile). The Cosmos program accepted their report as valid and awarded a **$1,000** bounty.

Both accounts demonstrate that account age or negative signal metrics did not block valid submissions from receiving rewards, highlighting selective enforcement against already-triaged findings.

The April PRs are one batch. On 4 May only the Critical ticket in that batch was called an audit. The two High tickets were not.

The program had the three analyses, and Support had the escalation, before `56ca5e5`, before the April PRs, and before the lock notes.

## One migration, three breaks

IBC v1 port is `wasm.` plus address, set at instantiate only if the contract exports V1 IBC. IBC v2 adds `wasm2` plus address for IBC2-capable contracts.

1. Dispatch passed the V1 port into chains that authorized V2 actions.  
2. The V2 send encoder copied contract-supplied `SourcePort` instead of deriving it from the sender.  
3. ibc-go v10 stored async packets by client and sequence, with no port key. wasmd did not require the caller to own the destination port before `WriteAcknowledgement`.

## #3547293 — async WriteAck authorization (Critical)

**Tested:** wasmd v0.61.7, ibc-go v10.5.0

A destination can return `PacketStatus_Async`. ibc-go keeps the packet until `WriteAcknowledgement`. A successful write deletes that entry.

In v0.61.7, `IBC2RawPacketHandler` accepted `WriteAcknowledgement` when the port string in the handler was non-empty. It did not load the pending packet. It did not require the caller’s V2 port to match the destination. `keeper.go` passed `contractInfo.IBCPortID` (V1) into that chain. A contract with stubbed V1 IBC exports gets a non-empty V1 port and passes the empty-string check.

A third-party contract could finish another contract’s pending async packet. The legitimate destination then failed with packet-not-found. If the counterparty treats acknowledgement bytes as authority, that is a protocol break.

**Shipped as:** `56ca5e5`, then #2480 / #2481 (`GetAsyncPacket`; caller V2 port must match destination).

## #3547382 — unverified SourcePort on send (High)

**Tested:** wasmd v0.61.7

V1 derived the port from the executing sender. V2 (`EncodeIBCv2Msg`) copied `payload.SourcePort` from the contract message into `MsgSendPacket`. The SDK signer still matched the executing contract. IBC v2 callbacks resolve the contract from `SourcePort`. A send that named another contract’s V2 port invoked send, ack, and timeout on the victim.

Destination apps that treat `SourcePort` as the sender then accept the attacker’s packet as the victim.

**Shipped as:** #2478 / #2479 (`SourcePort` must equal `PortIDForContractV2(sender)`; identity check on `OnSendPacket`).

## #3551078 — V1 port into V2 handlers (High)

**Tested:** wasmd v0.61.8

`IBCPortID` is set only if the binary exports V1 IBC. `IBC2PortID` is set for IBC2. `instantiate`, `execute`, `migrate`, `Sudo`, and `reply` forwarded `IBCPortID`. An empty V1 port made the V2 handler return unsupported.

A pure IBC2 contract could receive on `OnRecvIBC2Packet` (V2 port) and return async, then fail `WriteAcknowledgement` from `execute()`. Recv stayed committed; the ack could not be written. That blocks source timeout and blocks finalization. Whether funds stay locked depends on the application. `OnSendIBC2Packet` forwarded `IBCPortID` while the other IBC2 callbacks used `IBC2PortID`.

**Shipped as:** #2478 / #2479, including `IBC2PortID` on `OnSendIBC2Packet` and on dispatch.

No public incident in this record attaches a production drain to these three bugs. That does not clear the keeper defects.

## Operators

- v0.61.x → **v0.61.11 or later** (#2479, #2481)  
- 0.70 line → **v0.70.0 or later** (#2478, #2480)

Contracts should still check the signer in application callbacks.

## Sources

- HackerOne #3547293, #3547382, #3551078  
- [56ca5e5](https://github.com/CosmWasm/wasmd/commit/56ca5e529c66b3fa0e18d2aa0915812ae249ba01)  
- [#2478](https://github.com/CosmWasm/wasmd/pull/2478) · [#2479](https://github.com/CosmWasm/wasmd/pull/2479) · [#2480](https://github.com/CosmWasm/wasmd/pull/2480) · [#2481](https://github.com/CosmWasm/wasmd/pull/2481)  
- [v0.61.11 CHANGELOG](https://github.com/CosmWasm/wasmd/blob/v0.61.11/CHANGELOG.md)  
- [v0.70.0 CHANGELOG](https://github.com/CosmWasm/wasmd/blob/v0.70.0/CHANGELOG.md)
