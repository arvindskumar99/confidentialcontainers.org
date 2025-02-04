---
title: SEV-SNP Host Setup 
description: Host configurations for AMD SEV-SNP machines 
weight: 10
categories:
- prerequisites
tags:
- SEV
---

## Platform Setup

The host BIOS and kernel must be capable of supporting AMD SEV-SNP and the host must be configured accordingly.

The latest SEV Firmware version is available on AMD's [SEV Developer Webpage](https://www.amd.com/en/developer/sev.html). It can also be updated via a platform OEM BIOS update.

The host kernel must be equal to or later than upstream version [6.11](https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.11.tar.xz).

To build just the upstream compatible host kernel, use the Confidential Containers fork of [AMDESE AMDSEV](https://github.com/confidential-containers/amdese-amdsev/tree/amd-snp-202501150000). Individual components can be built by running the following command:

```
./build.sh kernel host --install
```

Additionally, [sev-utils](https://github.com/amd/sev-utils/blob/coco-202501150000/docs/snp.md) can be used to install the required host kernel, but it will unnecessarily build AMD compatible guest kernel, OVMF, and QEMU components as these packages are already packaged with Kata. The additional components can be used with the script utility to test launch and attest a base QEMU SNP guest.