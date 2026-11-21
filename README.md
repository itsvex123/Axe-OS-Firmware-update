# Axe-OS Firmware Update

Verified ESP-Miner **v2.15.4-vex-mod-dual-audit4** for the Bitaxe Gamma 601 (BM1370, ESP32-S3).

| File | Purpose |
| --- | --- |
| [esp-miner.bin](esp-miner.bin) | Exact AxeOS OTA application image |
| [ESP-MINER-SHA256.txt](ESP-MINER-SHA256.txt) | SHA-256 for the binary |
| [VEX-Gamma601-Audit4.bundle](VEX-Gamma601-Audit4.bundle) | Complete Git history, exact source, and test evidence |
| [Stratum V1/V2 audit](AUDIT-STRATUM-V1-V2-2026-11-21.md) | Findings on share replies, fresh work, and reconnects |

Firmware source: `fd26427173401e9d0e84b2a02738e98616dd215a`. The archive contains the previously tested image, with 97 passing QEMU tests and both host sanitizer configurations passing.

To restore the source, download the Git bundle and run:

```sh
git clone --branch main VEX-Gamma601-Audit4.bundle vex-gamma601
```

Based on [bitaxeorg/ESP-Miner](https://github.com/bitaxeorg/ESP-Miner). See [LICENSE](LICENSE).
