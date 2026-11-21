# ESP-Miner Audit 4: Stratum V1 and V2

Audited 2026-11-21 for Bitaxe Gamma 601 (BM1370, ESP32-S3). This is a review of the existing release, with no firmware or tuning changes.

- Firmware: `v2.15.4-vex-mod-dual-audit4`, **2,724,000 bytes**, built with ESP-IDF `v6.0.2`.
- Exact firmware source commit: `fd26427173401e9d0e84b2a02738e98616dd215a`; the archival commit adds the binary and evidence without changing firmware source.
- SHA-256: `b371e5e31f81126e65453d207ab492f03f3ddc7d9130dec0e538a01e4c9ef151`.
- The image is an AxeOS app OTA update. It is not a factory image or a disk/flash dump, and the backup does not contain device logs or configuration extracted from the miner.

## Why CK SV2 shows ~1.5 seconds

CKPool's [published share handler][ck-share] validates shares, then [queues successful acknowledgments][ck-batch]. It flushes at 10 accepted shares or when the batch is at least one second old, with additional flush paths on job changes and close. The [connector polls for aged batches roughly once per second][ck-tick]. An isolated share can wait one second to become eligible and nearly one additional second for the next pass. A one-share acknowledgment can therefore take roughly 1–2 seconds before transport; with a uniform arrival phase and no early flush, the median wait is about 1.5 seconds. This is the *same response format* when the batch contains one or many shares, and the [SV2 specification][sv2-spec] permits aggregation. The code path is shared by Standard and Extended in the published CK implementation.

This explains the observed pattern well. It does not establish which commit or settings the live CK service is running, or directly measure network RTT. No packet capture or CK server timestamps were available. The batch timer is after share validation in the published code and does not delay that validation or make the miner wait to continue hashing.

| CK SV2 Extended sample | Log 7: final Extended session | Log 8: Extended excerpt |
| --- | ---: | ---: |
| Coverage | 3 h 21 m | 6 h 26 m |
| Accepted shares | 1,408 | 2,679 |
| Logged stales / other rejects | 2 / 0 | 0 / 0 |
| Stratum reconnects | 0 | 1 |
| Single-share replies | 998 | 1,927 |
| Single-share p50 / p99 | 1,519.65 / 2,035.6 ms | 1,526.3 / 2,044.8 ms |
| Single-share replies in [1, 2) seconds | 928 (93.0%) | 1,745 (90.6%) |
| Multi-share reply p50 / p99 | 792.9 / 1,831.0 ms | 697.45 / 1,917.8 ms |
| Max logged nonce-to-socket | 9.847 ms | 8.397 ms |
| Max logged fresh-work handoff | 5.849 ms | 4.855 ms |

Nearest-rank p99; multi-share timings are one observation per reply and use the latest accepted sequence in that reply. Both logged stales arrived within 40 ms of receiving a new block job on the miner. The one log-8 reconnect reopened the channel in 4.585 s; the next logged acceptance was 31.551 s after the receive error. The latter includes finding a share and receiving its reply, so it cannot be used as an ASIC downtime measurement. The earlier dashboard zero cannot be matched to this reconnect. The SV1 30–40 ms comparison is an operator observation on the same miner and pool, not a controlled packet capture.

## Firmware audit and follow-ups

| Area | Finding | Recommended action |
| --- | --- | --- |
| SV1 share replies | `components/stratum/stratum_api.c` records before the write; `main/tasks/stratum_v1_task.c` timestamps a full received line before JSON parsing and matches by request ID. Current fast CK SV1 replies give no evidence of a regression. | Keep this path; keep timing reply age separate from fresh work. |
| SV2 Extended and Standard share replies | `main/tasks/stratum_v2_task.c` records before encryption/write, then computes age using the reply's latest accepted sequence after frame processing and acquiring the state lock. No socket write holds that state lock. `components/stratum_v2/sv2_noise.c` combines each share's encrypted header and body into one contiguous write buffer. Both protocols enable TCP_NODELAY through `components/stratum/stratum_socket.c`. | UI: label “Share reply time” and explain aggregation even for count 1. Capture the last receive-byte timestamp before decryption/locking if diagnostic parity becomes necessary. Do not subtract a fixed second or use share difficulty to manipulate the number. |
| New work, share priority and fallback | SV1 invalidates old work on clean job events; SV2 advances work generation on a valid `SetNewPrevHash`. Existing tests cover stale generations, candidate priority over waiting ordinary shares, and adopting an already qualified connection on failback. An in-progress TCP frame cannot be preempted without breaking the stream. The exact physical automatic failback interval remains unmeasured. | Use clean job to ASIC and candidate to socket timing to judge latency. For any future changes, preserve stale checks and verify both SV1 and SV2. |
| Instantaneous hashrate at SV2 tip changes | The SV2 tip handler calls `SYSTEM_clean_jobs_queue()` in `main/system.c`. It also resets the monitor. `main/tasks/hashrate_monitor_task.c` needs a fresh counter pair for the long counter's rate, so a dashboard zero can occur while the ASIC's actual counter continues. This is a plausible explanation of the screenshot, not a trace proving it. | Decouple job invalidation from telemetry reset, with regression checks for continuous counters, true ASIC resets, and pauses. Preserve legitimate zero samples. |
| SV1 reader under backpressure | `mining.ping` and `client.get_version` replies in `main/tasks/stratum_v1_task.c` can write synchronously from the SV1 reader via the serialized writer in `components/stratum/stratum_write.c`. A following job notice can wait behind a blocked control write. No occurrence identified in the live logs. | Reproduce control message followed by clean job under a blocked writer before introducing a separate control queue. Preserve ordered frames, bounded queue behavior and connection ownership. |

**Disposition:** Keep Audit 4 as the release. The 1.5 s reading alone is no reason to downgrade SV2 or force a pool switch. Neither comparing reply times nor a short share sample proves one protocol produces more work on this board. Live Standard timing has not been independently measured.

## Verification repeated against this checkpoint

- Packed source history checked with `git fsck`. Production build's source commit was matched to the checkpoint. The archived app binary was byte-compared with that existing production build, with no rebuild substituted.
- ESP32-S3 image checksum and validation hash passed `esptool image-info`; image size fits each 4 MiB app slot with 1,470,304 bytes remaining.
- ESP32-S3 QEMU replay: **97 tests, 0 failures, 0 ignored**, clean return. QEMU exercises the target test application, not a physical miner or the live pools.
- Host suite passed at `-O0` and `-O2` with ASan/UBSan. Suites include V1 JSON/fallback, V2 Standard and Extended authenticated fallback transcripts, socket write errors, queue handoff and share-priority tests.
- Xtensa production object check passed: the mining job generation check uses an aligned load and memory barriers without calls to software atomic helpers.

[ck-share]: https://github.com/ckolivas/ckpool/blob/be3e1e0796fbea60b7792a9b62fc2614764e1fc0/src/sv2_strat.c#L1333-L1354
[ck-batch]: https://github.com/ckolivas/ckpool/blob/be3e1e0796fbea60b7792a9b62fc2614764e1fc0/src/sv2_strat.c#L588-L638
[ck-tick]: https://github.com/ckolivas/ckpool/blob/be3e1e0796fbea60b7792a9b62fc2614764e1fc0/src/connector.c#L1362-L1372
[sv2-spec]: https://stratumprotocol.org/specification/05-mining-protocol/#5313-submitsharessuccess-server---client
