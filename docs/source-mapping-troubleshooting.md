# Source Mapping Troubleshooting

Use this guide to match a source-mapping symptom to its failure class and the
next step to take. Every class below exists in `internal/sourcemap` or in the
`debug` command's flag validation. For flags and discovery rules, see
[source-mapping.md](source-mapping.md).

Run `glassbox debug --dry-run <tx-hash>` first. It checks `--contract-source`
and `--source-alias` before any simulation runs.

## Symptom table

| Symptom (message excerpt) | Failure class | Source | Next step |
|---------------------------|---------------|--------|-----------|
| `source discovery: local build directory not found` / preflight `wasm_target_dir` warning | **Missing artifacts:** there is no build output directory | `discovery.go`, `preflight.go` | Build the contract so that `target/wasm32-unknown-unknown/release/` exists, or pass `--contract-source` |
| preflight `wasm_artifacts`: `no .wasm files found in …` | **Missing artifacts:** the directory exists but is empty | `preflight.go` | Rebuild. Discovery only indexes `*.wasm` files directly in that directory |
| `does not have a valid WASM magic number (\0asm) — skipped` | **Invalid artifact:** the file is misnamed or corrupt | `discovery.go` | Remove or rebuild the file. Files that do not start with `\0asm` are never indexed |
| `could not read "<path>": … (skipped)` | **Unreadable artifact** | `discovery.go` | Fix the file permissions |
| `build mismatch: local WASM hash … does not match on-chain hash …` | **Mismatched artifact:** the local build differs from the deployed bytecode | `discovery.go` (`HashMismatchError`) | Rebuild from the deployed commit with the same flags. Matching is by SHA-256 only |
| Result quality `partial` (`.debug_info` stripped) | **Partial debug info** | `fallback.go` stage 2 | Keep DWARF in the release profile (`debug = true`) |
| Result quality `heuristic` | **No DWARF:** paths are inferred from symbols or `Cargo.toml` | `fallback.go` stages 3–4 | Keep DWARF, then treat the reported paths as approximate |
| Result quality `unknown`; WAT shown instead of source | **No mapping possible** | `fallback.go` stage 5 | Same as above |
| `WASM data is nil or too small (N bytes)` | **Truncated or empty bytecode** | `fallback.go` input guard | Make sure the full binary was uploaded or fetched |
| `Location: <path>:L:C` printed with no source excerpt | **Unresolved path:** the embedded path does not exist on this machine | `debug.go` `displaySourceLocation` | Remap the path with `--source-alias` or point `--contract-source` at the checkout. See [path remapping](source-mapping.md#path-remapping-example) |
| `--contract-source: directory not found` / `is a file, not a directory` / `value must not be empty` | **Bad override path** | `debug.go` flag validation | Pass an existing directory |
| `--source-alias: file not found` / `failed to parse … as JSON` | **Bad alias file** | `debug.go` flag validation | Fix the path, or make the file a flat JSON object |
| `alias: target for "<name>" must point to an existing directory` | **Stale alias target** | `alias.go` `LoadAliasConfig` | Update or remove that entry. A relative target is resolved against the directory that holds the alias file |
| `path contains null bytes and cannot be used` | **Unsafe path** | `debug.go`, `discovery.go`, `preflight.go` | Remove the null bytes |
| preflight `skip_source_mapping_env` | **Mapping disabled** | `preflight.go` | Unset `GLASSBOX_SKIP_SOURCE_MAPPING` |
| preflight `source_map_cache_dir` error | **Bad cache directory** | `preflight.go` | Unset `GLASSBOX_SOURCE_MAP_CACHE` or point it at a writable directory |

## Notes

- These checks do not depend on a toolchain. Discovery hashes `.wasm`
  bytes, and the fallback pipeline reads DWARF or symbol names whatever
  produced them. The `cargo build` hints in messages are only the
  default suggestion. Any build that writes a matching `.wasm` to the scanned
  directory, or any source tree passed with `--contract-source`, works.
- A hash mismatch never blocks debugging. It warns you that line numbers may
  not match the deployed contract.
