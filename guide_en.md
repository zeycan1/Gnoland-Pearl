# Sapphire → Pearl (pearl-1) Validator Migration Guide
<img width="1200" height="630" alt="image" src="https://github.com/user-attachments/assets/661348b2-a66b-4d5c-8f91-abb5d5daf64b" />


Complete, field-tested guide for the Sapphire (sapphire-1) to Pearl (pearl-1) validator migration. Every command here was executed on a live server during an actual migration.

## 🔗 Links

* **Gnoweb:** [https://pearl.testnets.gno.land](https://pearl.testnets.gno.land)
* **RPC (Official):** [https://rpc.pearl.testnets.gno.land](https://rpc.pearl.testnets.gno.land)
* **Faucet:** [https://faucet.gno.land/](https://faucet.gno.land/) *(now a single hub, not a chain-specific subdomain — remember to select the network)*
* **Gnockpit:** [https://gnockpit.pearl.testnets.gno.land](https://gnockpit.pearl.testnets.gno.land)
* **Status:** [https://status.pearl.testnets.gno.land](https://status.pearl.testnets.gno.land)
* **Tx-indexer:** [https://indexer.pearl.testnets.gno.land/graphql](https://indexer.pearl.testnets.gno.land/graphql)
* **Release/diff:** [sapphire...pearl diff](https://github.com/gnolang/gno/pull/6091) (release candidate PR #6091)

## ⚡ Summary — What you need to know

* **Pearl is a NEW CHAIN, not a hardfork** — No state is migrated from Sapphire.
* **Chain ID:** `pearl-1`
* **Branch:** `chain/pearl`
* **Genesis SHA256:** `c45fe60c8c8a1f859d9e4d5aad7ce4d100ff0eb78302e71318ba0de481a8dc91`
* **Genesis source:** Generated from the repo's `examples/` tree, 85 curated packages are in genesis.
* **Namespace enforcement:** `r/sys/names` is now active from block 1 (it came later in Sapphire).
* **Validator set:** 3 founding validators, each with 60 power — if 1 validator goes offline, it perfectly hits the 1/3 halt threshold. A fragile start, be careful.
* **Faucet:** Now a single central hub (faucet.gno.land), no longer a network-specific subdomain.

---

## 0️⃣ BEFORE TOUCHING ANYTHING — Mandatory checks to avoid breaking Sapphire

Run the following commands and note the outputs. Proceeding without knowing these risks your running node.

**a) How is Sapphire currently running?**
```bash
systemctl status gnoland-sapphire --no-pager
cat /etc/systemd/system/gnoland-sapphire.service | grep -E "ExecStart|WorkingDirectory|data-dir"
```

**b) Is Sapphire already using a separate binary?**
If it was left as `ExecStart=/root/go/bin/gnoland start ...` (the shared binary), you must isolate Sapphire to its own binary (like Topaz) *before* compiling Pearl. Otherwise, the Pearl compilation will overwrite Sapphire's binary and crash Sapphire on its next restart.

**c) Which ports is it using?**
```bash
ss -tulpn | grep gnoland
```
Sapphire used the 56xxx range (56656/56657/56658). Pick an empty prefix for Pearl — e.g., 57xxx:
```bash
ss -tulpn | grep -E '57656|57657|57658' || echo "✅ 57xxx is empty, available to use"
```

**d) Is the key type Ed25519?**
```bash
jq -r '.pub_key["@type"]' /root/sapphire-data/secrets/priv_validator_key.json
```
It should return: `/tm.PubKeyEd25519`. Verify this before proceeding. Stop if there is an unexpected difference.

### ⚠️ Read these first — Pitfalls that will burn you
* **Do not compile Pearl without isolating Sapphire to a separate binary first.** `make install` overwrites `$GOPATH/bin/gnoland` every time. Do not skip Step 1.
* **Fill both `p2p.persistent_peers` and `p2p.seeds`** with the same addresses (official docs usually only mention one, but both were needed previously).
* In Pearl, `AddPackage` now only type-checks production files; test files are parsed for syntax (#6025) — keep this in mind when deploying your own realm/package.
* **1 validator = 1/3 halt risk.** There are 3 founding validators, equally weighted. Until the valset expands, the network is fragile. Keep your node stable and avoid restarts/downtime.
* **NEVER copy the `priv_validator_state.json` file from Sapphire.** Pearl starts from block 1.
* Gas requirements vary between chains. 100M gas was used previously; allocate the same, but increase it if you hit out-of-gas errors.
* **Registering does not automatically put you in the valset.** (Just like Sapphire, GovDAO approval is likely required).

---

## 1️⃣ Isolate Sapphire to a separate binary (if still using shared)

```bash
cd $HOME/gno-sapphire
git checkout chain/sapphire
go build -o $HOME/go/bin/gnoland-sapphire ./gno.land/cmd/gnoland

sudo sed -i 's|ExecStart=.*/gnoland start|ExecStart='"$HOME"'/go/bin/gnoland-sapphire start|' \
  /etc/systemd/system/gnoland-sapphire.service

sudo systemctl daemon-reload    # DOES NOT touch the running process
systemctl status gnoland-sapphire --no-pager   # Verify it is still running with the same PID
```

## 2️⃣ Compile Binary (Pearl)

```bash
cd $HOME && rm -rf gno-pearl
git clone https://github.com/gnolang/gno.git gno-pearl
cd gno-pearl && git checkout chain/pearl
make -C gno.land install.gnoland install.gnokey

gnoland version && gnokey version    # Both should say chain/pearl
```
*Note: The announcement includes prebuilt binaries + CHECKSUMS.txt for darwin/linux × amd64/arm64. If you download them instead of compiling, verify the checksums.*

## 3️⃣ Directories, keys, and state reset

```bash
mkdir -p /root/pearl-data/config /root/pearl-data/secrets

gnoland config init  -config-path /root/pearl-data/config/config.toml
gnoland secrets init -data-dir    /root/pearl-data/secrets/

# Safety net: backup the newly generated key
cp /root/pearl-data/secrets/priv_validator_key.json /root/pearl-newkey-backup.json

# Migrate Sapphire consensus + node identity
cp /root/sapphire-data/secrets/priv_validator_key.json /root/pearl-data/secrets/
cp /root/sapphire-data/secrets/node_key.json           /root/pearl-data/secrets/

# 🔴 CRITICAL: state MUST BE ZERO
cat > /root/pearl-data/secrets/priv_validator_state.json << 'EOF'
{"height": "0", "round": "0", "step": 0}
EOF

# Verify
gnoland secrets get -data-dir /root/pearl-data/secrets/ validator_key
cat /root/pearl-data/secrets/priv_validator_state.json
```

## 4️⃣ Node configuration

```bash
CFG=/root/pearl-data/config/config.toml

# Verified public seed nodes from official Pearl RPC /net_info
PEERS="g1m37xukfq6yl555k93fcyzns83qnmgyax9zm875@98.95.153.199:26656,g1ngukqd3khekaqjf90k45cglzm0l25wwzl2fkn2@52.19.119.34:26656"

gnoland config set -config-path $CFG rpc.laddr        tcp://0.0.0.0:57657
gnoland config set -config-path $CFG p2p.laddr        tcp://0.0.0.0:57656
gnoland config set -config-path $CFG proxy_app        tcp://127.0.0.1:57658
gnoland config set -config-path $CFG moniker          "ZeycaNode"
gnoland config set -config-path $CFG application.prune_strategy         syncable
gnoland config set -config-path $CFG consensus.timeout_commit           3s
gnoland config set -config-path $CFG consensus.peer_gossip_sleep_duration 10ms
gnoland config set -config-path $CFG p2p.flush_throttle_timeout         10ms
gnoland config set -config-path $CFG mempool.size                       10000
gnoland config set -config-path $CFG p2p.max_num_outbound_peers         40
gnoland config set -config-path $CFG p2p.pex                            true
gnoland config set -config-path $CFG p2p.external_address "$(wget -qO- eth0.me):57656"

# 🔑 SET BOTH
gnoland config set -config-path $CFG p2p.seeds            "$PEERS"
gnoland config set -config-path $CFG p2p.persistent_peers "$PEERS"

# Firewall
sudo ufw allow 57656/tcp comment "pearl p2p"
```

## 5️⃣ Genesis

```bash
cd /root/pearl-data/config
wget -O genesis.json https://github.com/gnolang/gno/releases/download/chain/pearl/genesis.json
shasum -a 256 genesis.json
```
**Must match exactly:**
`c45fe60c8c8a1f859d9e4d5aad7ce4d100ff0eb78302e71318ba0de481a8dc91`

## 6️⃣ systemd service

```bash
sudo tee /etc/systemd/system/gnoland-pearl.service > /dev/null <<EOF
[Unit]
Description=Gnoland Pearl node (ZeycaNode)
After=network-online.target

[Service]
User=root
WorkingDirectory=/root
Environment=HOME=/root
Environment=GNOROOT=/root/gno-pearl
ExecStart=/root/go/bin/gnoland start \\
  --chainid pearl-1 \\
  --genesis /root/pearl-data/config/genesis.json \\
  --data-dir /root/pearl-data/ \\
  --log-level info \\
  --skip-genesis-sig-verification
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now gnoland-pearl
journalctl -u gnoland-pearl -n 30 --no-pager -o cat
```

## 7️⃣ Sync

```bash
curl -s http://localhost:57657/status | jq -r '.result.sync_info | "height: \(.latest_block_height) | catching_up: \(.catching_up)"'
curl -s http://localhost:57657/net_info | jq -r '.result.n_peers'

# Auto-monitor
while true; do
  R=$(curl -s http://localhost:57657/status | jq -r '"\(.result.sync_info.latest_block_height) \(.result.sync_info.catching_up)"')
  echo "$(date +%H:%M:%S) $R"
  [ "${R#* }" = "false" ] && echo "🎉 SYNC COMPLETE" && break
  sleep 15
done
```

## 8️⃣ Faucet + valoper registration

```bash
gnokey query -remote "https://rpc.pearl.testnets.gno.land" auth/accounts/g1eep2z6k5u0t5964a5fa6qtqplkt9pnk926kh34
```
If balance is zero/insufficient: Select the Pearl network via [https://faucet.gno.land/](https://faucet.gno.land/) and send funds to your wallet.

```bash
gnoland secrets get -data-dir /root/pearl-data/secrets/ validator_key
```
Verify that the `signing address` and `gpub` match the output you copied from Sapphire in Step 3.

**Registration (Candidacy):** (Replace `your_wallet_name` with your gnokey name for `g1eep2z...`)
```bash
gnokey maketx call \
  --pkgpath gno.land/r/gnops/valopers \
  --func Register \
  --args "ZeycaNode" \
  --args "Your_Description" \
  --args "Your_Data_Center" \
  --args "your_operator_address" \
  --args "your_pubkey_address" \
  --gas-fee 1000000ugnot \
  --gas-wanted 100000000 \
  --chainid pearl-1 \
  --remote https://rpc.pearl.testnets.gno.land \
  --broadcast \
  your_wallet_name
```

**Verify:**
```bash
gnokey query vm/qrender -data "gno.land/r/gnops/valopers:your_operator_address" -remote https://rpc.pearl.testnets.gno.land
```

## 9️⃣ Discord announcement

> Registered on Pearl ✅
> [https://pearl.testnets.gno.land/r/gnops/valopers:your_operator_address](https://pearl.testnets.gno.land/r/gnops/valopers:your_operator_address)
> Txhash: TX_HASH
>
> Node fully synced ✅ ready for valset inclusion.

---

## 🛠 Troubleshooting

* **Stuck at 0 peers / height 0:** Check if both `p2p.seeds` and `p2p.persistent_peers` are set with the correct addresses. If it still doesn't progress, delete `db/` and `wal/` and restart.
* **error on replay: wrong Block.Header.AppHash (crash loop):**
  ```bash
  sudo systemctl stop gnoland-pearl
  rm -rf /root/pearl-data/db /root/pearl-data/wal
  sudo systemctl start gnoland-pearl
  ```
  *(Keys inside `secrets/` will not be affected).*
* **Out of gas on Register:** Increase the `--gas-wanted` value (try `150000000`).
* **Sapphire crashes on restart with a new chain ID:** You skipped Step 1. Recompile Sapphire to a separate binary from its source branch and update the systemd service.
* **Network halts if 1 of 3 validators is offline:** This is a known fragility at launch (hitting the 1/3 halt threshold). If your node is one of the 3, plan restarts for off-peak hours.

## ✅ Post-migration checklist

* `gnoland version` → `chain/pearl`
* Genesis SHA256 matches (`c45fe60c...`)
* Both `p2p.seeds` and `p2p.persistent_peers` are set with real Pearl addresses
* Validator key migrated (Ed25519), state reset to 0
* `catching_up: false`, sufficient peers
* Sapphire is running on its own isolated binary (`gnoland-sapphire`)
* Valoper registration realm pkgpath verified from official source
* Valoper registration completed (with the same operator address)
* Discord announcement made, valset inclusion is being monitored
* Service is enabled (survives reboot)
* Firewall: new p2p port (57xxx) is open
