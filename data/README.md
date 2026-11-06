# Result data

[Main report](../README.md) · [Method and settings](../appendix-a-tests.md)

| File | Rows | Meaning |
| --- | ---: | --- |
| [native-intervals.csv](native-intervals.csv) | 106 | Measured interval fps for 12 Windows runs, 12 Linux sweep runs and two Debian 90-second runs. The reported total validated frame count repeats per interval. Group by trial, BAR, codec and mode before summing totals. |
| [native-finite-sweep.csv](native-finite-sweep.csv) | 28 | Four finite attempts at 256 MiB and four passing tests per larger BAR. The 48-surface failures yielded three decodable frames each. Empty PSNR/hash cells have no complete passing result. |
| [rolling-pool-intervals.csv](rolling-pool-intervals.csv) | 56 | Four intervals per mode and BAR size. The full 0–60 second interval overlaps the three component intervals. The decoded frame count belongs to a separate saved sample and repeats per interval. |
| [reproducer.json](reproducer.json) | — | Case platform, synthetic fixture fingerprint and native reference parameters and software fingerprints. |

MiB and GiB use binary units. Actual interval endpoints are elapsed monotonic seconds from first completed output for the native clients and from first positive progress for the rolling-pool harness. Requested endpoints define the reporting bins. Actual endpoints may differ slightly because output is sampled at discrete times. The data retain six decimal places for interchange. The narrative tables round for readability.

T-series IDs refer to the main investigation. WR-series IDs refer to the separate boot-engineering investigation. The CSVs are selected measurements for the report, with case setup failures covered in the text. No blank cell represents a zero-throughput Windows large-BAR test. That test was never performed.
