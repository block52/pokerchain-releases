# Pokerchain releases

Public downloads for running a node on the Block52 network (`pokerchain`): the network's official genesis file, and
(soon) the `pokerchaind` release binaries.

| Path | What it is |
|---|---|
| [`networks/pokerchain/genesis.json`](networks/pokerchain/genesis.json) | Official genesis of the Block52 mainnet (chain id `pokerchain`) |
| [`networks/pokerchain/genesis.json.sha256`](networks/pokerchain/genesis.json.sha256) | Its SHA-256, in `sha256sum` format |
| [Releases](https://github.com/block52/pokerchain-releases/releases) | `pokerchaind` binaries for each version (not published here yet) |

## Genesis

| | |
|---|---|
| Chain ID | `pokerchain` |
| Genesis time | `2026-09-01T23:54:39.326034141Z` |
| Initial height | `1` |
| SHA-256 | `ae808c4889f1c520c92f40d55dd984d41fec7a03b0373170dcafdeeaf1454731` |
| Address prefix | `b52` |
| Denom | `usdc` (6 decimals) |
| Genesis validator | `node1` |

### Download and verify

```bash
curl -fsSL https://raw.githubusercontent.com/block52/pokerchain-releases/main/networks/pokerchain/genesis.json -o genesis.json
echo "ae808c4889f1c520c92f40d55dd984d41fec7a03b0373170dcafdeeaf1454731  genesis.json" | sha256sum -c -
# macOS: replace `sha256sum -c -` with `shasum -a 256 -c -`
```

Windows (PowerShell):

```powershell
Invoke-WebRequest https://raw.githubusercontent.com/block52/pokerchain-releases/main/networks/pokerchain/genesis.json -OutFile genesis.json
(Get-FileHash genesis.json -Algorithm SHA256).Hash.ToLower()   # must print the SHA-256 above
```

Only use a file whose hash matches.

> [!WARNING]
> Don't use the genesis that a node's RPC returns (`/genesis`). It is the same chain, but in CometBFT's flat
> `consensus_params` layout rather than the nested `consensus.params` layout `pokerchaind` keeps on disk. Its hash
> won't match, and a node replaying blocks from genesis with it fails with a wrong `AppHash` around block 150.

## Run a node with state sync

State sync restores a recent snapshot from the network instead of replaying every block since genesis, so a new node
is in sync within minutes. These steps were checked on 2026-11-21 with `pokerchaind` v0.1.211.

**1. Install the `pokerchaind` version the network runs.** Check which one that is:

```bash
curl -s https://node1.block52.xyz/cosmos/base/tendermint/v1beta1/node_info | jq -r .application_version.version
```

**2. Create the node's home and install the genesis.**

```bash
pokerchaind init <your-moniker> --chain-id pokerchain --home ~/.pokerchain
curl -fsSL https://raw.githubusercontent.com/block52/pokerchain-releases/main/networks/pokerchain/genesis.json \
  -o ~/.pokerchain/config/genesis.json
(cd ~/.pokerchain/config && echo "ae808c4889f1c520c92f40d55dd984d41fec7a03b0373170dcafdeeaf1454731  genesis.json" | sha256sum -c -)
```

**3. Pick a trust point** (a recent block the light client starts from):

```bash
RPC=https://node1.block52.xyz/rpc
TRUST_HEIGHT=$(( $(curl -s $RPC/status | jq -r .result.sync_info.latest_block_height) - 1000 ))
TRUST_HASH=$(curl -s "$RPC/block?height=$TRUST_HEIGHT" | jq -r .result.block_id.hash)
echo "$TRUST_HEIGHT $TRUST_HASH"
```

**4. Edit `~/.pokerchain/config/config.toml`.** Change these keys in their existing sections:

```toml
[p2p]
persistent_peers = "2aed496d3ee5201b510b7575b1523ffde51bc51e@node1.block52.xyz:26656,8b927a7e936fe95477284b6378f40bf1c8d695e9@node.texashodl.net:26656"

[statesync]
enable = true
rpc_servers = "http://node1.block52.xyz:26657,http://node.texashodl.net:26657"
trust_height = <TRUST_HEIGHT>
trust_hash = "<TRUST_HASH>"
trust_period = "168h0m0s"
```

**5. Edit `~/.pokerchain/config/app.toml`.** Run the poker engine inside the node: set it in the **existing `[pvm]`
section**, not a new one.

```toml
minimum-gas-prices = "0usdc"

[pvm]
pvm_url = ""
embedded_engine = "embedded"
```

Optional settings:

- **Keep only recent blocks** (a few thousand is plenty to play):

  ```toml
  pruning = "custom"
  pruning-keep-recent = "3000"
  pruning-interval = "10"
  min-retain-blocks = 3000
  ```

- **Let the Block52 desktop app use this node** (its *Localhost* network). Leave these listeners on localhost:

  ```toml
  # app.toml
  [api]
  enable = true
  enabled-unsafe-cors = true
  ```

  ```toml
  # config.toml
  [rpc]
  cors_allowed_origins = ["*"]
  ```

**6. Start it and wait until it has caught up.**

```bash
pokerchaind start --home ~/.pokerchain
# in another shell; done when this prints false:
curl -s localhost:26657/status | jq .result.sync_info.catching_up
```

The node log should show `engine_mode=embedded`. The node also serves the poker WebSocket relay on
`ws://localhost:8585/ws` (set `WS_ENABLED=false` to turn it off).

A state-synced node has no blocks or transactions from before its snapshot. Use an archive node or the Block52
indexer for older history.

## Release binaries

Each release will publish, under the version's tag:

- `pokerchaind-<os>-<arch>` (darwin amd64/arm64, linux amd64/arm64, windows amd64)
- the matching `.sha256` files
- `SHA256SUMS` and its Sigstore bundle `SHA256SUMS.sigstore.json`

Check a download against `SHA256SUMS` before running it:

```bash
sha256sum -c SHA256SUMS --ignore-missing
```
