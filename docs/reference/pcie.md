# PCI Express bus

## Link Speeds

Per [wikipedia](https://en.wikipedia.org/wiki/PCI_Express):

| PCIe version | Transmission rate | Encoding  | Bandwidth (1x) | Bandwidth (16x) |
| :----------- | ----------------: | :-------  | -------------: | --------------: |
| 1.0          | 2.5 GT/s          | 8b/10b    | 0.25 GB/s      |        4.0 GB/s |
| 2.0          | 5.0 GT/s          | 8b/10b    | 0.50 GB/s      |        8.0 GB/s |
| 3.0          | 8.0 GT/s          | 128b/130b | 0.99 GB/s      |       15.7 GB/s |
| 4.0          | 16.0 GT/s         | 128b/130b | 1.97 GB/s      |       31.5 GB/s |
| 5.0          | 32.0 GT/s         | 128b/130b | 3.94 GB/s      |       63.0 GB/s |

These are the official specifications of the standard. "GB" is Gigabytes in base-10. As a rule of thumb, PCIe 4.0 can do 2 GB/s per lane, so 32 GB/s on a 16x slot. Some PCIe implementations may drive the links at a higher rate, for example the Extended Speed Mode (ESM) using 20.0 GT/s and 25.0 GT/s in CCIX on top of PCIe 4.0.

The transfer rates are relevant for e.g. GPUs connected to the CPU through PCIe (the common case for workstation/desktop graphics cards) and NICs.