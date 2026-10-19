# Common setup and launch

Sources: the pinned MemCache config/install/compatibility docs and MemFabric installation docs in [sources](sources.md).

## Inspect before installing

Use the intended runtime interpreter:

```bash
python3 -m pip show memcache-hybrid memfabric-hybrid vllm vllm-ascend
python3 -m pip check
npu-smi info
uname -r
getconf PAGESIZE
free -h
```

Record package paths, package versions, repository commits and local patches. MemCache's dependency metadata selects its MemFabric range; resolve that range before pinning both versions. Prefer a compatible existing image. If installing, use explicitly selected compatible wheel versions for the target architecture/Python, then verify imports in that interpreter. Do not upgrade a running environment just because the Wiki says latest. The historical SSD introduction uses 1.2.0; it does not establish that every later UBSIO setting exists in that wheel.

Current MemFabric installation docs require a HYBM AICPU operator for A5 (and A2 device_rdma via hcomm). For versions exposing `mfcli`, inspect `mfcli kernel info`, then use `mfcli kernel install` when installation is needed and within scope; the current standalone kernel build requires CANN >=9.1.0. Older packages have different delivery flows: use their documentation. Do not replace that procedure universally with an import side effect.

Containers must see their allocated NPU devices, required driver/CANN libraries, transport devices and shared memory; SSD owners additionally need the selected raw block devices. Reuse a site-approved container definition. No generic requirement to launch every container privileged or mount the entire host filesystem.

## Configuration

Locate installed templates instead of assuming a Python path:

```bash
python3 -c 'from pathlib import Path; import memcache_hybrid; print(Path(memcache_hybrid.__file__).resolve().parent / "config")'
```

Copy `mmc-meta.conf` and `mmc-local.conf` into the deployment directory. Preserve release-specific settings. These are substitutions, not literal runnable values:

```ini
# meta.conf: one reachable control-plane endpoint per deployment
ock.mmc.meta_service_url = tcp://<META_HOST>:5000
ock.mmc.meta_service.config_store_url = tcp://<CONFIG_HOST>:6000
ock.mmc.meta_service.metrics_url = http://<METRICS_BIND_HOST>:8000
```

```ini
# local.conf: all participants agree on Meta / ConfigStore
ock.mmc.meta_service_url = tcp://<META_HOST>:5000
ock.mmc.local_service.config_store_url = tcp://<CONFIG_HOST>:6000
ock.mmc.local_service.world_size = 256
ock.mmc.local_service.protocol = <SELECTED_PROTOCOL>
ock.mmc.local_service.dram.size = <PER_OWNER_CAPACITY>
ock.mmc.local_service.max.dram.size = <RELEASE_SUPPORTED_RESERVATION>
ock.mmc.local_service.storage.enabled = false
```

`world_size` is the maximum rank count in the inspected template, not the model TP/DP count; count standalone and inference participants and keep it stable after registration. Inspect the selected release's reservation constraints before choosing `max.dram.size`. For device protocols, current templates describe 1GB capacity alignment. Actual DRAM capacity is the sum of aligned per-owner allocations. Separate MemCache DRAM from UBSIO buffers, model/runtime host memory and OS reserves; `max.dram.size` is not proof of registered physical capacity.

A minimal `run_meta.sh` contains:

```bash
export MMC_META_CONFIG_PATH=/absolute/deployment/meta.conf
python3 -c 'from memcache_hybrid import MetaService; MetaService.main()'
```

## Inference integration

Set `MMC_LOCAL_CONFIG_PATH` in each inference launcher. Retain the working model/parallel arguments; for a colocated vLLM Ascend service, the official integration example adds:

```bash
--enable-prefix-caching --kv-transfer-config '{"kv_connector":"AscendStoreConnector","kv_role":"kv_both","kv_connector_extra_config":{"backend":"memcache"}}'
```

This fragment is not a complete model launch command. Confirm connector registration and accepted extra fields in the installed vllm-ascend. Do not blindly add `use_layerwise`, a module path or a connector from another release. Hybrid models and layerwise transfer require matching layout support in that release.

For PD, preserve the working P-to-D connector and role topology. Compose pooling through the framework's supported multi-connector mechanism; replacing the transfer connector with the store connector breaks PD. Do not assume that `kv_both` from the colocated example is the correct role for every PD child connector. Generate P/D/proxy scripts only when PD was requested.
