# Helper Scripts

Collection of simple helpful tools for all network users and operators.

## State Sync Bootstrap

[statesync.sh]() configures a fresh republicd node to join the network via state sync instead of replaying the full block history.
It queries the official snapshot provider for the latest height, calculates a trust height aligned to the 1000-block snapshot interval, cross-verifies the block hash against a witness RPC node, and patches config.toml with the correct state sync parameters and persistent peers.

Prerequisites: `republicd init <moniker>` must have been run first. Requires curl, jq, and sed.

```sh
# Configure state sync with defaults (~/.republic)
./scripts/republicsync.sh

# Custom home directory
./scripts/republicsync.sh --home /opt/republic

# [OPTIONAL] Wipe existing state before configuring
./scripts/republicsync.sh --reset
```

The script connects to the following:
Node: Snapshot provider
Role: Serves snapshots + primary light client RPC
Endpoint: https://state-sync-service.republicai.io

---

Node: RPC node
Role: Witness for light client cross-verification
Endpoint: https://rpc.republicai.io
Both nodes are added as persistent peers automatically. After running the script, start the node with `republicd start`, or restart your systemd process..
