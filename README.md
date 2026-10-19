# Cosmos Called It Spam After Triage. Then They Shipped the Patch.

Three security reports against Cosmos CosmWasm wasmd / IBC v2 covered one Critical and two High vulnerabilities. All three were reproduced, validated, and moved to Triaged by an official reviewer. 2 weeks after, the program updated its rules and closed the reports as Spam. Seven weeks later, the exact fixes shipped in v0.70.0 and v0.61.11.

## Meta
* **Reports:** #3547293 (Critical), #3547382 (High), #3551078 (High)
* **Target:** CosmWasm wasmd / IBC v2 (ibc-go v10)
* **Tested Versions:** wasmd v0.61.7 / v0.61.8
* **Patched Releases:** v0.61.11 and v0.70.0
* **HackerOne Triagers:** [povim_serdar](https://github.com/srdtrk), [cozartshmoopler](https://github.com/aljo242)
* **Commits/PRs:** [@pinosu](https://github.com/pinosu) (author/merger), [@DariuszDepta](https://github.com/DariuszDepta) (approver)

---

## Timeline and Administrative Sequence

Between February 10 and February 12, 2026, three reports were submitted against wasmd v0.61.7 and v0.61.8. Each included an ibctesting reproduction script.

On February 16 and 19, @povim_serdar reviewed the reports, confirmed the bugs, and marked all three as Triaged.

On March 2, the Cosmos program published revised eligibility rules. On the same day, @cozartshmoopler closed all three previously Triaged reports as Spam. 

A Spam status is an eligibility designation that applies a signal penalty. It is an administrative action, not a technical refutation.

An escalation was filed with multiple HackerOne Support. Support noted the conflict but left the Spam status untouched.

On March 18, commit `56ca5e5` by @pinosu landed on main with the commit message `fix`. It changed the async-ack logic.

On April 23 and 24, four PRs (#2478, #2479, #2480, #2481) were opened and merged without descriptions. @DariuszDepta approved them with `LGTM`. They were tagged `g:fix` and merged as a single release batch for main and v0.61.x.

The fixes shipped in v0.70.0 and v0.61.11. The changelogs explicitly state: "Fix IBC v2 async acknowledgements" (#2480) and "Fix IBC v2 packet send" (#2478).

On May 4, a Request to Disclose was submitted. The program locked all three reports that day.

The justification split. For the Critical report (#3547293), the triager claimed: "The changes made were related to audit fixes, not to your submissions." For the two High reports (#3547382 and #3551078) in the same release batch, the program claimed the disclosure request was "threatening in direct violation of our policies."

In September 2026, a final escalation through HackerOne Support confirmed the program refused to alter its stance, citing "signal criteria" and Support closed the ticket.

---

## Git Markers

| Report | Severity | Vulnerability Class | Git Marker |
|---|---|---|---|
| #3547293 | Critical | Async ack authorization | Commit `56ca5e5` (Mar 18), PR #2480 / #2481 (Apr 24) |
| #3547382 | High | Unvalidated SourcePort on send | PR #2478 / #2479 (Apr 24) |
| #3551078 | High | V1 port passing to V2 handlers | PR #2478 / #2479 (Apr 24) |

---

## Selective Enforcement and Platform Metrics

The program cited account signal and trial reports to maintain the Spam status. Platform data shows a different pattern.

Account `knafeh` joined in June 2026, months after the February reports. Cosmos accepted their report and paid reward bounty.

Account `wataha` joined in February 2026 with a negative signal rating (-3.60, 32nd percentile). Cosmos accepted their report and paid reward bounty.

Neither account age nor low signal blocked those payouts. The policy was applied selectively to reports that were already triaged.

The program had full access to the technical reports and proof-of-concept code before commit `56ca5e5`, before PRs #2478-#2481, and before the audit claims were introduced.

---

## Technical Root Cause

The issues stem from the migration between IBC v1 and IBC v2.

IBC v1 uses `wasm.` followed by the contract address. This port is set at instantiation only if the contract exports V1 IBC interface handlers. IBC v2 uses `wasm2` followed by the address for IBC2-capable contracts.

Three distinct bugs existed in this execution path.

### 1. #3547293: Async WriteAck Authorization (Critical)

* **Versions:** wasmd v0.61.7, ibc-go v10.5.0

IBC packets can return `PacketStatus_Async`. The ibc-go module holds the pending packet until `WriteAcknowledgement` is called. A successful write clears the packet entry.

In wasmd v0.61.7, `IBC2RawPacketHandler` accepted `WriteAcknowledgement` calls as long as the handler's port string was not empty. It did not verify if the caller's V2 port matched the destination port recorded on the pending packet.

In `keeper.go`, `contractInfo.IBCPortID` (the V1 port) was passed into the handler chain. A contract with stubbed V1 IBC exports produced a non-empty V1 port string. This passed the non-empty check.

An attacker contract could execute `WriteAcknowledgement` on behalf of another contract's pending async packet. The legitimate destination contract would later attempt to finalize the packet and hit a `packet-not-found` error. If counterparty logic relies on acknowledgement state, the execution flow breaks.

* **Fix:** Commit `56ca5e5`, followed by PRs #2480 and #2481. `GetAsyncPacket` now validates that the caller's V2 port matches the destination port.

### 2. #3547382: Unverified SourcePort on Send (High)

* **Versions:** wasmd v0.61.7

In IBC v1, the source port was derived directly from the executing contract address.

In IBC v2, `EncodeIBCv2Msg` copied `payload.SourcePort` straight from the contract payload into `MsgSendPacket`. The Cosmos SDK signer still matched the caller, but IBC v2 callbacks resolve the executing contract identity via `SourcePort`.

A contract could set `SourcePort` to another contract's V2 port. This allowed an attacker to invoke send, ack, and timeout callbacks under the target contract's identity. 

* **Fix:** PRs #2478 and #2479. `SourcePort` must equal `PortIDForContractV2(sender)`. Identity checks were added to `OnSendPacket`.

### 3. #3551078: V1 Port Passed to V2 Handlers (High)

* **Versions:** wasmd v0.61.8

`IBCPortID` is only populated when a binary exports V1 IBC. `IBC2PortID` is populated for V2.

The entry points for `instantiate`, `execute`, `migrate`, `Sudo`, and `reply` forwarded `IBCPortID` instead of `IBC2PortID`. An empty V1 port caused the V2 handler to return an unsupported error.

A pure IBC2 contract could receive an `OnRecvIBC2Packet` callback and return an async status. If it later called `WriteAcknowledgement` inside an `execute()` call, the write failed because the handler received an empty `IBCPortID`. The receive state committed on-chain, but the ack could not be written. The packet remained stuck, blocking source timeouts.

In addition, `OnSendIBC2Packet` forwarded `IBCPortID`, whereas other IBC2 callbacks used `IBC2PortID`.

* **Fix:** PRs #2478 and #2479. Updated dispatch logic to use `IBC2PortID` for all IBC2 callbacks, including `OnSendIBC2Packet`.

---

## Upgrade Path

* **0.61.x release line:** Upgrade to `v0.61.11` or higher.
* **0.70.x release line:** Upgrade to `v0.70.0` or higher.

Contract developers should continue validating the caller explicitly inside application-level callbacks.

---

## References

* HackerOne Reports #3547293, #3547382, #3551078
* [Commit 56ca5e5](https://github.com/CosmWasm/wasmd/commit/56ca5e529c66b3fa0e18d2aa0915812ae249ba01)
* [PR #2478](https://github.com/CosmWasm/wasmd/pull/2478)
* [PR #2479](https://github.com/CosmWasm/wasmd/pull/2479)
* [PR #2480](https://github.com/CosmWasm/wasmd/pull/2480)
* [PR #2481](https://github.com/CosmWasm/wasmd/pull/2481)
* [wasmd v0.61.11 CHANGELOG](https://github.com/CosmWasm/wasmd/blob/v0.61.11/CHANGELOG.md)
* [wasmd v0.70.0 CHANGELOG](https://github.com/CosmWasm/wasmd/blob/v0.70.0/CHANGELOG.md)
