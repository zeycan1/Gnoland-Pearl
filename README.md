# Sapphire → Pearl (pearl-1) Validator Geçişi 
<img width="1200" height="630" alt="image" src="https://github.com/user-attachments/assets/8a7c7873-d38d-4428-9383-bd74bb176e5d" />


Sapphire (sapphire-1) → Pearl (pearl-1) validator geçişi için eksiksiz, sahada test edilmiş rehber. Buradaki her komut gerçek bir sunucuda, gerçek geçiş sırasında çalıştırıldı.


## 🔗 Bağlantılar

- Gnoweb: https://pearl.testnets.gno.land
- RPC (Resmi): https://rpc.pearl.testnets.gno.land
- Faucet: https://faucet.gno.land/ (artık zincire özel alt domain değil, tek faucet hub — ağı seçmeyi unutma)
- Gnockpit: https://gnockpit.pearl.testnets.gno.land
- Status: https://status.pearl.testnets.gno.land
- Tx-indexer: https://indexer.pearl.testnets.gno.land/graphql
- Release/karşılaştırma: sapphire...pearl diff'i (release candidate PR #6091)


## ⚡ Özet — Bilmen gerekenler

| | |
|---|---|
| Pearl | YENİ BİR ZİNCİR, hardfork değil — Sapphire'dan hiçbir state taşınmıyor |
| Chain ID | `pearl-1` |
| Branch | `chain/pearl` |
| Genesis SHA256 | `c45fe60c8c8a1f859d9e4d5aad7ce4d100ff0eb78302e71318ba0de481a8dc91` |
| Genesis kaynağı | repo'nun `examples/` ağacından üretilmiş, 85 curated paket genesis'te var |
| Namespace enforcement | `r/sys/names` artık **blok 1'den itibaren** aktif (Sapphire'da sonradan geliyordu) |
| Validator seti | 3 founding validator, her biri 60 power — **1 validator offline olursa 1/3 halt sınırına tam oturuyor**. Kırılgan bir başlangıç, dikkatli ol |
| Faucet | Artık tek merkezi hub (`faucet.gno.land`), ağa özel alt domain değil |

## 0️⃣ DOKUNMADAN ÖNCE — Sapphire'a zarar vermemek için zorunlu kontroller

Aşağıdaki komutları çalıştır, çıktıları not al. Bunları bilmeden ilerlemek çalışan nodu riske atar.

**a) Sapphire şu an nasıl çalışıyor?**
```bash
systemctl status gnoland-sapphire --no-pager
cat /etc/systemd/system/gnoland-sapphire.service | grep -E "ExecStart|WorkingDirectory|data-dir"
```

**b) Sapphire zaten ayrı binary kullanıyor mu?**

Önceki geçişte `ExecStart=/root/go/bin/gnoland start ...` şeklinde bırakmıştık (paylaşılan binary). Eğer hâlâ öyleyse, Pearl'i derlemeden önce Sapphire'ı da Topaz gibi kendi binary'sine almalısın — yoksa Pearl derlemesi Sapphire'ın altındaki binary'yi ezer ve bir sonraki restart'ta Sapphire çöker.

**c) Hangi portları kullanıyor?**
```bash
ss -tulpn | grep gnoland
```
Sapphire `56xxx` bandını kullanıyordu (56656/56657/56658). Pearl için boş bir ön ek seç — örn. `57xxx`:
```bash
ss -tulpn | grep -E '57656|57657|57658' || echo "✅ 57xxx boş, kullanılabilir"
```

**d) Anahtar tipi Ed25519 mi?**
```bash
jq -r '.pub_key["@type"]' /root/sapphire-data/secrets/priv_validator_key.json
```
Zaten biliyoruz: `/tm.PubKeyEd25519` (yukarıdaki tablo) → taşınabilir. Yine de komutu çalıştırıp dosyanın gerçekten bu anahtarı içerdiğini teyit et.Beklenmedik bir farklılık varsa ilerlemeden dur.

## ⚠️ Önce bunları oku — seni yakacak tuzaklar

- **Sapphire'ı önce ayrı binary'ye almadan Pearl'i derleme.** `make install` her seferinde `$GOPATH/bin/gnoland`'ın üzerine yazar. Bu yüzden 1️⃣ adımı atlama.
- `p2p.persistent_peers` **ve** `p2p.seeds` ikisini de aynı adreslerle doldur (Sapphire'da işe yaramıştı, resmi doküman genelde sadece birinden bahsediyor).
- Pearl'de `AddPackage` artık **sadece production dosyalarını type-check ediyor**, test dosyaları syntax olarak parse ediliyor (#6025) — kendi realm/paket deploy ederken bunun farkında ol, ama validator geçişini etkilemez.
- **1 validator = 1/3 halt riski.** 3 founding validator var, her biri eşit ağırlıkta. Sen valset'e eklendiğinde ağın güvenliği açısından bu hâlâ kırılgan — node'unu stabil tut, ilk günlerde restart/downtime'a özellikle dikkat et.
- `priv_validator_state.json` dosyasını Sapphire'dan **ASLA** kopyalama. Pearl blok 1'den başlıyor.
- Gas ihtiyacı zincirden zincire değişebiliyor. Sapphire'da resmi 50M yetmemiş, biz 100M kullanmıştık. Pearl için de aynı payı ayır ama out-of-gas alırsan artır.
- Register'ın seni doğrudan valset'e sokmayacağını unutma (Sapphire'daki gibi, muhtemelen Pearl'de de GovDAO onayı gerekiyor — teyit et).

## 1️⃣ Sapphire'ı ayrı binary'ye geçir (hâlâ paylaşılan binary'yi kullanıyorsa)

```bash
cd $HOME/gno-sapphire
git checkout chain/sapphire
go build -o $HOME/go/bin/gnoland-sapphire ./gno.land/cmd/gnoland

sudo sed -i 's|ExecStart=.*/gnoland start|ExecStart='"$HOME"'/go/bin/gnoland-sapphire start|' \
  /etc/systemd/system/gnoland-sapphire.service

sudo systemctl daemon-reload    # çalışan process'e DOKUNMAZ
systemctl status gnoland-sapphire --no-pager   # hâlâ aynı PID ile çalıştığını doğrula
```

## 2️⃣ Binary derleme (Pearl)

```bash
cd $HOME && rm -rf gno-pearl
git clone https://github.com/gnolang/gno.git gno-pearl
cd gno-pearl && git checkout chain/pearl
make -C gno.land install.gnoland install.gnokey

gnoland version && gnokey version    # ikisi de chain/pearl yazmalı
```

> Not: Duyuruda darwin/linux × amd64/arm64 için prebuilt binary + CHECKSUMS.txt de var. Kaynaktan derlemek yerine hazır binary indirmek istersen release sayfasındaki checksum'ları doğrulamayı unutma.

## 3️⃣ Dizinler, anahtarlar ve state sıfırlama

```bash
mkdir -p /root/pearl-data/config /root/pearl-data/secrets

gnoland config init  -config-path /root/pearl-data/config/config.toml
gnoland secrets init -data-dir    /root/pearl-data/secrets/

# Güvenlik ağı: yeni üretilen anahtarı yedekle
cp /root/pearl-data/secrets/priv_validator_key.json /root/pearl-newkey-backup.json

# Sapphire consensus + node kimliğini taşı
cp /root/sapphire-data/secrets/priv_validator_key.json /root/pearl-data/secrets/
cp /root/sapphire-data/secrets/node_key.json           /root/pearl-data/secrets/

# 🔴 KRİTİK: state SIFIR olmalı
cat > /root/pearl-data/secrets/priv_validator_state.json << 'EOF'
{"height": "0", "round": "0", "step": 0}
EOF

# Doğrula
gnoland secrets get -data-dir /root/pearl-data/secrets/ validator_key
cat /root/pearl-data/secrets/priv_validator_state.json
```

## 4️⃣ Node yapılandırması
```
CFG=/root/pearl-data/config/config.toml
```
# Resmi Pearl RPC'sinin /net_info uç noktasından doğrulanmış genel seed node'ları (gno-core-pubseed-1 / -2)
```
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
```
# 🔑 İKİSİNİ BİRDEN yaz
```
gnoland config set -config-path $CFG p2p.seeds            "$PEERS"
gnoland config set -config-path $CFG p2p.persistent_peers "$PEERS"and config set -config-path $CFG p2p.persistent_peers "$PEERS"

```

Güvenlik duvarı:
```bash
sudo ufw allow 57656/tcp comment "pearl p2p"
```

## 5️⃣ Genesis

```bash
cd /root/pearl-data/config
wget -O genesis.json https://github.com/gnolang/gno/releases/download/chain/pearl/genesis.json
shasum -a 256 genesis.json
```

Birebir eşleşmeli:
```
c45fe60c8c8a1f859d9e4d5aad7ce4d100ff0eb78302e71318ba0de481a8dc91
```

## 6️⃣ systemd servisi

```bash
sudo tee /etc/systemd/system/gnoland-pearl.service > /dev/null <<EOF
[Unit]
Description=Gnoland Pearl node (moniker)
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
```

Otomatik izlemek için:
```bash
while true; do
  R=$(curl -s http://localhost:57657/status | jq -r '"\(.result.sync_info.latest_block_height) \(.result.sync_info.catching_up)"')
  echo "$(date +%H:%M:%S) $R"
  [ "${R#* }" = "false" ] && echo "🎉 SYNC TAMAM" && break
  sleep 15
done
```

Genesis birkaç saniyede boot ediyormuş (duyuruya göre "booting in seconds") — sync süresi muhtemelen Sapphire'dan (~6 dk) çok farklı olmayacak ama peer sayısına bağlı, sabırlı ol.

## 8️⃣ Faucet + valoper kaydı

```bash
gnokey query -remote "https://rpc.pearl.testnets.gno.land" auth/accounts/g1eep2z6k5u0t5964a5fa6qtqplkt9pnk926kh34
```
Bakiye yoksa/yetersizse: https://faucet.gno.land/ üzerinden **Pearl ağını seçerek**, operator adresine cüzdan adresini yaıp gönder (artık zincire özel alt domain yok, tek hub üzerinden ağ seçiliyor).

```bash
gnoland secrets get -data-dir /root/pearl-data/secrets/ validator_key
```
Bu çıktının signing address ve gpub'ının yukarıdaki tabloyla birebir eşleştiğini doğrula — 3️⃣ adımında Sapphire'dan doğru anahtarı kopyaladığının son kontrolü budur.

Kayıt (adaylık — pkgpath'i teyit etmeden çalıştırma, bkz. yukarıdaki not). `wallet` yerine operator adresine (`g1eep2z...`) karşılık gelen kendi gnokey key ismini kullan:
```bash
gnokey maketx call \
  --pkgpath gno.land/r/gnops/valopers \
  --func Register \
  --args "Moniker" \
  --args "açıklamaların" \
  --args "data-center" \
  --args "operatör adresi" \
  --args "pubkeyadresi" \
  --gas-fee 1000000ugnot \
  --gas-wanted 100000000 \
  --chainid pearl-1 \
  --remote https://rpc.pearl.testnets.gno.land \
  --broadcast \
  wallet
```

Doğrula:
```bash
gnokey query vm/qrender -data "gno.land/r/gnops/valopers:operatör adresi" -remote https://rpc.pearl.testnets.gno.land
```

⚠️ 3 founding validator zaten genesis'te tanımlı geldiği için, senin eklenmen muhtemelen yine ayrı bir GovDAO/valset işlemi gerektiriyor. Sapphire'daki gibi Register = sadece adaylık varsayımıyla ilerle, teyit edene kadar valset'e otomatik gireceğini düşünme. Şu an Sapphire'da voting power 5 ile aktifsin (%0.75 pay) — Pearl'de aynı power ile mi başlayacağın yoksa GovDAO'nun ayrı belirleyeceği bir değer mi olacağı duyuruda net değil; kayıt sonrası bunu explorer'dan takip et.

## 9️⃣ Discord duyurusu

```
Registered on Pearl ✅ — ZeycaNode
https://pearl.testnets.gno.land/r/gnops/valopers:operatöradresin
Txhash: TX_HASH

Node fully synced ✅  ready for valset inclusion.
```

## 🛠 Sorun giderme

**0 peer / height 0'da takılı** → Hem `p2p.seeds` hem `p2p.persistent_peers` doğru adreslerle ayarlı mı kontrol et. Hâlâ ilerlemiyorsa `db/` + `wal/` silip baştan başla.

**error on replay: wrong Block.Header.AppHash (çökme döngüsü)** →
```bash
sudo systemctl stop gnoland-pearl
rm -rf /root/pearl-data/db /root/pearl-data/wal
sudo systemctl start gnoland-pearl
```
`secrets/` içindeki anahtarlar etkilenmez.

**Register'da out of gas** → `--gas-wanted` değerini artır (150000000 dene).

**Sapphire restart'ta yeni chain ID ile patlıyor** → 1. adımı atladın demektir; Sapphire'ı kaynağından tekrar ayrı binary derleyip systemd'yi ona yönlendir.

**3 validator'dan biri offline, ağ duruyor** → Bu launch'ta bilinen bir kırılganlık (1/3 halt sınırı tam üstünde). Kendi node'un bu 3'ten biriyse restart'larını gece/düşük trafik saatlerine planla.

## ✅ Geçiş sonrası kontrol listesi

- [ ] `gnoland version` → `chain/pearl`
- [ ] Genesis SHA256 eşleşiyor (`c45fe60c...`)
- [ ] Hem `p2p.seeds` hem `p2p.persistent_peers` gerçek Pearl adresleriyle ayarlı
- [ ] Validator anahtarı taşındı (Ed25519), state 0'a sıfırlandı
- [ ] `catching_up: false`, peer sayısı yeterli
- [ ] Sapphire kendi ayrı binary'sinde (`gnoland-sapphire`), hâlâ çalışıyor
- [ ] Valoper kayıt realm pkgpath'i resmi kaynaktan teyit edildi
- [ ] Valoper kaydı yapıldı (aynı operator adresiyle)
- [ ] Discord'da duyuru yapıldı, valset dahiliyeti takip ediliyor
- [ ] Servis enabled (reboot'tan sağ çıkar)
- [ ] Güvenlik duvarı: yeni p2p portu (57xxx) açık
