# Validate the selected route

1. Record Meta registration and actual owner count/capacity. Confirm inference health and complete one request. In PD, verify P-to-D transfer and proxy routing as well.
2. Warm distinct shared prefixes, then query them again with new suffixes. Control the request route and DP coverage. If everything stays in HBM, enlarge the prefix working set or use a supported targeted cache-reset operation; restarting the whole pool invalidates this check.
3. Capture before/after deltas for all participating inference engines and Meta during the same window. Exclude warmup from formal performance. Verify external reuse independently from service readiness.
4. For SSD, require actual write/eviction/readback evidence and a successful consuming request. Native metrics, logs and a controlled workload are the first choice. `iostat -dxm -y <DEVICE> 2` shows interval device bandwidth but includes unrelated traffic and cannot prove useful KV hits. Kernel-bypass I/O may not be represented. Do not add custom statistics code unless requested.
5. Keep the user's performance metric and baseline. For a no-history-reuse baseline, disable HBM and external history reuse while retaining PD KV transfer. A two-tier versus three-tier comparison is a different experiment. Do not promise 4x, 90% hits or an SSD share from an unrelated historical model/run.

For matching token counters, combined reuse = (HBM hit + external hit) / HBM query. Equivalently H + (1-H)*E only when external query equals HBM query minus HBM hit. Never add conditional percentages or mix periods. SSD key-hit share is an object-count metric, not automatically a token/byte share for hybrid/layerwise layouts. Label the user's chosen SSD denominator and method; omit an unsupported exact attribution claim.

When a benchmark is requested, use an existing supported tool such as AISBench and preserve its native output and success/failure counts. Deliver commands, resolved configurations, source/package versions, data parameters/hash, warmup boundaries, logs and metrics. This skill includes no fixed benchmark wrapper or SSD tracing library.
