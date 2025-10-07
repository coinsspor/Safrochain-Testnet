# 🌌 Safrochain Cosmovisor Node Kurulum Rehberi

![Cosmos SDK](https://img.shields.io/badge/Cosmos%20SDK-v0.50.13-blue)
![Go](https://img.shields.io/badge/Go-1.23.9-00ADD8)
![Version](https://img.shields.io/badge/Version-v0.1.0-green)
![Cosmovisor](https://img.shields.io/badge/Cosmovisor-Latest-purple)
![libwasmvm](https://img.shields.io/badge/libwasmvm-v2.0.1-orange)

**SON TEST:** 07.10.2025 - Başarılı ✅

## 📋 İçindekiler

- [Sistem Gereksinimleri](#-sistem-gereksinimleri)
- [Değişkenler](#-değişkenler)
- [Kurulum](#-kurulum)
- [Servis Yönetimi](#-servis-yönetimi)
- [Faydalı Komutlar](#-faydalı-komutlar)
- [Sorun Giderme](#-sorun-giderme)
- [Node'u Tamamen Kaldırma](#-nodeu-tamamen-kaldırma)

## 🖥️ Sistem Gereksinimleri

| Bileşen | Minimum | Önerilen |
|---------|---------|----------|
| CPU | 4 vCPU | 8 vCPU |
| RAM | 8 GB | 16 GB |
| Depolama | 200 GB SSD | 500 GB NVMe |
| İşletim Sistemi | Ubuntu 20.04 | Ubuntu 22.04 |
| Bant Genişliği | 100 Mbps | 1 Gbps |

## 🔧 Değişkenler

**ÖNEMLİ:** Kurulum öncesi bu değişkenleri kendi değerlerinizle değiştirin:

```bash
# Kendi değerlerinizi girin
export MONIKER="YOUR_NODE_NAME"        # Node adınız
export PORT_PREFIX="53"                 # İstediğiniz port prefix (10-65 arası)
export WALLET_NAME="wallet"            # Cüzdan adı
```

## 🚀 Kurulum

### 1️⃣ Sistem Güncellemesi ve Gerekli Paketler

```bash
# Ana dizine geçin
cd $HOME

# Sistem güncellemesi
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git wget htop tmux build-essential jq make lz4 gcc unzip
```

### 2️⃣ Go 1.23.9 Kurulumu

**ÖNEMLİ:** Safrochain Go 1.23.9 gerektirir!

```bash
cd $HOME
sudo rm -rf /usr/local/go
wget "https://golang.org/dl/go1.23.9.linux-amd64.tar.gz"
sudo tar -C /usr/local -xzf "go1.23.9.linux-amd64.tar.gz"
rm "go1.23.9.linux-amd64.tar.gz"

# Go PATH ayarları
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> ~/.bash_profile
source ~/.bash_profile

# Versiyon kontrolü
go version
# Beklenen çıktı: go version go1.23.9 linux/amd64
```

### 3️⃣ Değişkenleri Kaydetme

```bash
# Değişkenleri kalıcı yapma
echo "export MONIKER=\"$MONIKER\"" >> $HOME/.bash_profile
echo "export WALLET_NAME=\"$WALLET_NAME\"" >> $HOME/.bash_profile
echo "export PORT_PREFIX=\"$PORT_PREFIX\"" >> $HOME/.bash_profile
echo "export SAFROCHAIN_PORT=\"${PORT_PREFIX}\"" >> $HOME/.bash_profile
source $HOME/.bash_profile

# Kontrol
echo "Moniker: $MONIKER"
echo "Port Prefix: $PORT_PREFIX"
echo "Wallet: $WALLET_NAME"
```

### 4️⃣ Binary Kurulumu

```bash
# Ana dizinde olduğunuzdan emin olun
cd $HOME

# Mevcut klasörü temizle
rm -rf safrochain-node

# Repository'yi klonla
git clone https://github.com/Safrochain-Org/safrochain-node.git
cd safrochain-node
git checkout v0.1.0

# Derle (Go 1.23.9 ile direkt çalışacak)
make install

# NOT: "cp: cannot stat '/safrochaind'" hatası önemsizdir, göz ardı edin
# Bu Makefile'daki küçük bir hatadır ama binary başarıyla derlenir

# Binary kontrolü
ls -la ~/go/bin/safrochaind
# Binary görünmeli: -rwxr-xr-x 1 root root 162249280 ... /root/go/bin/safrochaind
```

### 5️⃣ libwasmvm v2.0.1 Kurulumu (KRİTİK!)

**ÖNEMLİ:** 
- Safrochain SADECE libwasmvm v2.0.1 ile test edilmiş ve çalışıyor
- v2.2.4, v1.x veya diğer versiyonlar KESINLIKLE ÇALIŞMAZ!
- libwasmvm'i safrochain-node/lib dizinine kurmalısınız (Cosmovisor bu konuma bakacak)

```bash
# Safrochain dizinine gidin
cd ~/safrochain-node

# lib dizini oluştur
mkdir -p lib
cd lib

# libwasmvm v2.0.1 indir (TEST EDİLMİŞ VE ÇALIŞAN TEK VERSİYON!)
wget https://github.com/CosmWasm/wasmvm/releases/download/v2.0.1/libwasmvm.x86_64.so -O libwasmvm.so
chmod +x libwasmvm.so

# Test et
cd ~/safrochain-node
LD_LIBRARY_PATH=./lib ~/go/bin/safrochaind version

# Beklenen çıktı: v0.1.0
# Bu çıktı MUTLAKA görünmeli, yoksa devam etmeyin!
```

### 6️⃣ Cosmovisor Kurulumu

```bash
# Cosmovisor kur
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@latest

# Dizin yapısını oluştur
mkdir -p $HOME/.safrochain/cosmovisor/genesis/bin
mkdir -p $HOME/.safrochain/cosmovisor/upgrades

# libwasmvm'i Cosmovisor dizinine kopyala (yedek olarak)
cp $HOME/safrochain-node/lib/libwasmvm.so $HOME/.safrochain/cosmovisor/genesis/bin/

# Symlink oluştur
sudo ln -s $HOME/.safrochain/cosmovisor/genesis $HOME/.safrochain/cosmovisor/current -f
```

### 7️⃣ Cosmovisor için Wrapper Script (ZORUNLU!)

**KRİTİK:** 
- Cosmovisor'un libwasmvm'i bulabilmesi için wrapper script ŞART!
- Wrapper script'te `exec` veya `export` KULLANMAYINIZ! (Bu kritik, yoksa çalışmaz)
- Sadece `cd` ve direkt `LD_LIBRARY_PATH` kullanın

```bash
# WRAPPER SCRIPT OLUŞTUR (safrochaind adıyla)
cat > $HOME/.safrochain/cosmovisor/genesis/bin/safrochaind << 'EOF'
#!/bin/bash
cd /root/safrochain-node
LD_LIBRARY_PATH=./lib /root/go/bin/safrochaind "$@"
EOF

# Çalıştırılabilir yap
chmod +x $HOME/.safrochain/cosmovisor/genesis/bin/safrochaind

# Test et
$HOME/.safrochain/cosmovisor/genesis/bin/safrochaind version
# Beklenen çıktı: v0.1.0

# Eğer v0.1.0 görünmüyorsa, wrapper script'te sorun var!
```

### 8️⃣ Node Başlatma ve Konfigürasyon

```bash
# Initialize (wrapper ile)
$HOME/.safrochain/cosmovisor/genesis/bin/safrochaind init "$MONIKER" --chain-id safro-testnet-1

# client.toml manuel oluştur (config komutları çalışmıyor)
mkdir -p $HOME/.safrochain/config

cat > $HOME/.safrochain/config/client.toml << EOF
chain-id = "safro-testnet-1"
keyring-backend = "file"
output = "text"
node = "tcp://localhost:${PORT_PREFIX}657"
broadcast-mode = "sync"
EOF
```

### 9️⃣ Port Konfigürasyonu

```bash
# config.toml port ayarları
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:${PORT_PREFIX}658\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:${PORT_PREFIX}657\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:${PORT_PREFIX}060\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:${PORT_PREFIX}656\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":${PORT_PREFIX}660\"%" $HOME/.safrochain/config/config.toml

# app.toml port ayarları
sed -i -e "s%^address = \"tcp://localhost:1317\"%address = \"tcp://localhost:${PORT_PREFIX}317\"%; s%^address = \":8080\"%address = \":${PORT_PREFIX}080\"%; s%^address = \"localhost:9090\"%address = \"localhost:${PORT_PREFIX}090\"%; s%^address = \"localhost:9091\"%address = \"localhost:${PORT_PREFIX}091\"%" $HOME/.safrochain/config/app.toml
```

### 🔟 Genesis ve Addrbook

```bash
# Genesis dosyası
wget -O $HOME/.safrochain/config/genesis.json https://vault2.astrostake.xyz/testnet/safrochain/genesis.json

# Genesis kontrolü
if [ -f "$HOME/.safrochain/config/genesis.json" ]; then
    echo "✅ Genesis dosyası indirildi"
    sha256sum $HOME/.safrochain/config/genesis.json
else
    echo "❌ Genesis dosyası indirilemedi!"
    exit 1
fi

# Addrbook
wget -O $HOME/.safrochain/config/addrbook.json https://vault2.astrostake.xyz/testnet/safrochain/addrbook.json
```

### 1️⃣1️⃣ Seeds ve Peers Ayarları

```bash
# Seeds
SEEDS="2242a526e7841e7e8a551aabc4614e6cd612e7fb@88.99.211.113:26656"
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/" $HOME/.safrochain/config/config.toml

# Minimum gas price
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.001usaf\"/" $HOME/.safrochain/config/app.toml

# Pruning ayarları (opsiyonel - disk tasarrufu için)
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.safrochain/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.safrochain/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"10\"/" $HOME/.safrochain/config/app.toml
```

## 🎮 Servis Yönetimi

### Systemd Servisi Oluşturma (Cosmovisor ile)

```bash
# Cosmovisor servisi oluştur
sudo tee /etc/systemd/system/safrochaind.service > /dev/null <<EOF
[Unit]
Description=Safrochain Cosmovisor Node
After=network-online.target

[Service]
User=root
ExecStart=/root/go/bin/cosmovisor run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=/root/.safrochain"
Environment="DAEMON_NAME=safrochaind"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"
Environment="UNSAFE_SKIP_BACKUP=true"

[Install]
WantedBy=multi-user.target
EOF
```

### Servisi Başlatma

```bash
# Servisi etkinleştir ve başlat
sudo systemctl daemon-reload
sudo systemctl enable safrochaind
sudo systemctl start safrochaind

# Durumu kontrol et
sudo systemctl status safrochaind

# Logları izle
journalctl -fu safrochaind -o cat

# NOT: İlk başta "port already in use" hatası alabilirsiniz
# Servis birkaç kez restart olduktan sonra çalışmaya başlar
```

## 🚀 Hızlı Sync (Snapshot)

Sync işlemini hızlandırmak için snapshot kullanabilirsiniz:

### Snapshot İndirme ve Kurulum

```bash
# 1. lz4 kurulumu
sudo apt update
sudo apt-get install snapd lz4 -y

# 2. Servisi durdur ve state'i yedekle
sudo systemctl stop safrochaind
cp $HOME/.safrochain/data/priv_validator_state.json $HOME/.safrochain/priv_validator_state.json.backup

# 3. Eski data'yı temizle
rm -rf $HOME/.safrochain/data
safrochaind tendermint unsafe-reset-all --home ~/.safrochain/ --keep-addr-book

# 4. Snapshot'ı indir (NodeStake snapshot - her 12 saatte güncellenir)
SNAP_NAME=$(curl -s https://ss-t.safrochain.nodestake.org/ | egrep -o ">20.*\.tar.lz4" | tr -d ">")
curl -o - -L https://ss-t.safrochain.nodestake.org/${SNAP_NAME} | lz4 -c -d - | tar -x -C $HOME/.safrochain

# 5. Validator state'i geri yükle
mv $HOME/.safrochain/priv_validator_state.json.backup $HOME/.safrochain/data/priv_validator_state.json

# 6. Servisi başlat
sudo systemctl restart safrochaind

# 7. Logları kontrol et
sudo journalctl -fu safrochaind -o cat
```

**NOT:** Snapshot her 12 saatte bir güncellenir. Eğer snapshot indirme başarısız olursa, normal sync ile devam edebilirsiniz.

## 👤 Validator İşlemleri

### Cüzdan Oluşturma

```bash
# Alias kullanarak (eğer kurduysanız)
safrochaind keys add $WALLET_NAME

# Veya manuel
cd ~/safrochain-node
LD_LIBRARY_PATH=./lib ~/go/bin/safrochaind keys add $WALLET_NAME

# SEED PHRASE'İ GÜVENLİ BİR YERDE SAKLAYIN!
```

### Cüzdan Recovery

```bash
# Var olan cüzdanı seed phrase ile kurtar
safrochaind keys add $WALLET_NAME --recover
```

### Bakiye Kontrolü

```bash
# Adresinizi öğrenin
safrochaind keys show $WALLET_NAME -a

# Bakiye kontrolü
safrochaind query bank balances $(safrochaind keys show $WALLET_NAME -a)
```

### Validator Oluşturma

**ÖNEMLİ:** Önce sync olduğunuzdan emin olun ve yeterli token'a sahip olun!

```bash
# 1. Sync durumunu kontrol et
safrochaind status 2>&1 | jq .sync_info.catching_up
# false olmalı

# 2. Validator public key'i al
VAL_PUBKEY=$(safrochaind tendermint show-validator)

# 3. Validator config dosyası oluştur
cat <<EOF > /root/.safrochain/validator.json
{
  "pubkey": $VAL_PUBKEY,
  "amount": "1000000usaf",
  "moniker": "$MONIKER",
  "identity": "",
  "website": "",
  "security": "",
  "details": "Safrochain Validator",
  "commission-rate": "0.10",
  "commission-max-rate": "0.20",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
EOF

# 4. Validator oluştur
safrochaind tx staking create-validator /root/.safrochain/validator.json \
  --from=$WALLET_NAME \
  --chain-id=safro-testnet-1 \
  --gas=auto \
  --gas-adjustment=1.4 \
  --fees=300usaf \
  --home ~/.safrochain \
  -y
```

### Validator Düzenleme

```bash
safrochaind tx staking edit-validator \
--moniker "$MONIKER" \
--identity "<keybase_id>" \
--website "<website>" \
--security-contact "<email>" \
--details "<açıklama>" \
--commission-rate "0.10" \
--chain-id safro-testnet-1 \
--gas-prices 0.0001usaf \
--gas-adjustment 1.5 \
--gas auto \
--from $WALLET_NAME \
-y
```

### Token Delegate Etme

```bash
# Kendi validatörünüze delegate
safrochaind tx staking delegate $(safrochaind keys show $WALLET_NAME --bech val -a) 1000000usaf \
--chain-id safro-testnet-1 \
--gas-prices 0.0001usaf \
--gas-adjustment 1.5 \
--gas auto \
--from $WALLET_NAME \
-y
```

### Ödülleri Çekme

```bash
# Delegator ödülleri
safrochaind tx distribution withdraw-rewards $(safrochaind keys show $WALLET_NAME --bech val -a) \
--chain-id safro-testnet-1 \
--gas-prices 0.0001usaf \
--gas-adjustment 1.5 \
--gas auto \
--from $WALLET_NAME \
-y

# Validator komisyonu ile birlikte
safrochaind tx distribution withdraw-rewards $(safrochaind keys show $WALLET_NAME --bech val -a) --commission \
--chain-id safro-testnet-1 \
--gas-prices 0.0001usaf \
--gas-adjustment 1.5 \
--gas auto \
--from $WALLET_NAME \
-y
```

### Token Gönderme

```bash
safrochaind tx bank send $(safrochaind keys show $WALLET_NAME -a) <alıcı_adresi> 1000000usaf \
--chain-id safro-testnet-1 \
--gas-prices 0.0001usaf \
--gas-adjustment 1.5 \
--gas auto \
-y
```

## 📊 Node Durumu ve Bilgileri

```bash
# Sync durumu
safrochaind status 2>&1 | jq .sync_info

# Node ID
safrochaind tendermint show-node-id

# Validator bilgisi
safrochaind tendermint show-validator

# Peer sayısı
curl -s localhost:${PORT_PREFIX}657/net_info | jq .result.n_peers

# Bağlı peer'lar
curl -s localhost:${PORT_PREFIX}657/net_info | jq -r '.result.peers[] | "\(.node_info.id)@\(.remote_ip):\(.node_info.listen_addr)"' | awk -F ':' '{print $1":"$(NF)}'
```

## 🗳️ Governance (Yönetişim)

```bash
# Tüm teklifleri listele
safrochaind query gov proposals

# Belirli bir teklifi görüntüle
safrochaind query gov proposal <proposal_id>

# Oy kullan
safrochaind tx gov vote <proposal_id> <yes|no|no_with_veto|abstain> \
--chain-id safro-testnet-1 \
--gas-prices 0.0001usaf \
--gas-adjustment 1.5 \
--gas auto \
--from $WALLET_NAME \
-y
```

### Alias Kurulumu (Kolay Kullanım)

```bash
# Alias oluştur
echo 'alias safrochaind="LD_LIBRARY_PATH=/root/safrochain-node/lib /root/go/bin/safrochaind"' >> ~/.bashrc
source ~/.bashrc

# Artık direkt kullanabilirsiniz:
safrochaind version
safrochaind status
```

### Node Bilgileri

```bash
# Sync durumu
safrochaind status 2>&1 | jq .sync_info

# Alternatif (alias yoksa)
cd ~/safrochain-node
LD_LIBRARY_PATH=./lib ~/go/bin/safrochaind status 2>&1 | jq .sync_info

# Node ID
safrochaind tendermint show-node-id

# Validator bilgisi
safrochaind tendermint show-validator

# Peer sayısı
curl -s localhost:${PORT_PREFIX}657/net_info | jq .result.n_peers
```

### Cüzdan İşlemleri

```bash
# Yeni cüzdan oluştur
safrochaind keys add $WALLET_NAME

# Var olan cüzdanı recover et
safrochaind keys add $WALLET_NAME --recover

# Cüzdan listesi
safrochaind keys list

# Cüzdan adresi
safrochaind keys show $WALLET_NAME -a

# Bakiye kontrolü
safrochaind query bank balances $(safrochaind keys show $WALLET_NAME -a)
```

### Log Yönetimi

```bash
# Son 100 satır
journalctl -u safrochaind -n 100 --no-pager

# Canlı log takibi
journalctl -fu safrochaind -o cat

# Hata logları
journalctl -u safrochaind --no-pager | grep -i error
```

### Servis Kontrolü

```bash
# Durdur
sudo systemctl stop safrochaind

# Başlat
sudo systemctl start safrochaind

# Yeniden başlat
sudo systemctl restart safrochaind

# Durum
sudo systemctl status safrochaind
```

## 🛠️ Sorun Giderme

### Sorun: Binary "libwasmvm" Hatası Veriyor

```bash
# libwasmvm kontrolü
ldd ~/go/bin/safrochaind | grep wasmvm

# Eğer farklı bir path görüyorsanız (örn: /root/.lumera/lib/)
# Binary başka bir node'un libwasmvm'ini kullanıyor demektir

# Çözüm: Manuel test
cd ~/safrochain-node
LD_LIBRARY_PATH=./lib ~/go/bin/safrochaind version
# v0.1.0 görünmeli
```

### Sorun: Wrapper Script Çalışmıyor

Eğer wrapper script segmentation fault veriyorsa:

```bash
# Wrapper'ı kontrol et
cat $HOME/.safrochain/cosmovisor/genesis/bin/safrochaind

# Önemli: exec KULLANMAYIN!
# YANLIŞ:
# exec /root/go/bin/safrochaind "$@"

# DOĞRU:
# LD_LIBRARY_PATH=./lib /root/go/bin/safrochaind "$@"

# Wrapper'ı düzelt
rm $HOME/.safrochain/cosmovisor/genesis/bin/safrochaind
cat > $HOME/.safrochain/cosmovisor/genesis/bin/safrochaind << 'EOF'
#!/bin/bash
cd /root/safrochain-node
LD_LIBRARY_PATH=./lib /root/go/bin/safrochaind "$@"
EOF

chmod +x $HOME/.safrochain/cosmovisor/genesis/bin/safrochaind

# Test et
$HOME/.safrochain/cosmovisor/genesis/bin/safrochaind version
# v0.1.0 görünmeli
```

### Sorun: Port Already in Use

```bash
# Hangi process port kullanıyor?
sudo lsof -i:${PORT_PREFIX}656

# Tüm safrochain process'lerini kapat
sudo pkill -f safrochaind

# Servisi yeniden başlat
sudo systemctl restart safrochaind
```

### Alternatif: Cosmovisor'suz Direkt Servis

Eğer Cosmovisor ile sorun yaşıyorsanız:

```bash
# Cosmovisor servisini durdur
sudo systemctl stop safrochaind
sudo systemctl disable safrochaind

# Direkt servis oluştur
sudo tee /etc/systemd/system/safrochaind.service > /dev/null <<EOF
[Unit]
Description=Safrochain Node (Direct)
After=network-online.target

[Service]
User=root
WorkingDirectory=/root/safrochain-node
Environment="LD_LIBRARY_PATH=/root/safrochain-node/lib"
ExecStart=/root/go/bin/safrochaind start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

# Başlat
sudo systemctl daemon-reload
sudo systemctl enable safrochaind
sudo systemctl start safrochaind
```

### Sync Sorunları

```bash
# Addrbook güncelle
wget -O $HOME/.safrochain/config/addrbook.json https://vault2.astrostake.xyz/testnet/safrochain/addrbook.json
sudo systemctl restart safrochaind

# Node'u sıfırla (son çare)
sudo systemctl stop safrochaind
safrochaind tendermint unsafe-reset-all --home $HOME/.safrochain --keep-addr-book
sudo systemctl start safrochaind
```

## 🗑️ Node'u Tamamen Kaldırma

```bash
# Servisi durdur ve kaldır
sudo systemctl stop safrochaind
sudo systemctl disable safrochaind
sudo rm -f /etc/systemd/system/safrochaind.service
sudo systemctl daemon-reload

# Safrochain dosyalarını kaldır
rm -rf ~/.safrochain
rm -rf ~/safrochain-node
rm -f ~/go/bin/safrochaind

# Alias'ı kaldır
sed -i '/safrochaind/d' ~/.bashrc

# Değişkenleri temizle
sed -i '/MONIKER/d' ~/.bash_profile
sed -i '/WALLET_NAME/d' ~/.bash_profile
sed -i '/PORT_PREFIX/d' ~/.bash_profile
sed -i '/SAFROCHAIN/d' ~/.bash_profile

# Bash'i yenile
source ~/.bashrc
source ~/.bash_profile

echo "✅ Safrochain tamamen kaldırıldı!"
```

## ⚠️ Kritik Notlar

1. **Go Versiyonu**: Kesinlikle Go 1.23.9 kullanın
2. **libwasmvm**: v2.0.1 kullanın (v2.2.4 veya v1.x ÇALIŞMAZ!)
3. **Wrapper Script**: Cosmovisor için ZORUNLU
4. **LD_LIBRARY_PATH**: Her zaman `/root/safrochain-node/lib` göstermeli
5. **Binary Konumu**: Asıl binary `/root/go/bin/safrochaind`
6. **Wrapper Konumu**: `/root/.safrochain/cosmovisor/genesis/bin/safrochaind`

## 📞 Destek

- **GitHub**: https://github.com/Safrochain-Org/safrochain-node
- **Discord**: [Safrochain Discord](https://discord.gg/safrochain)
- **Telegram**: [Safrochain Telegram](https://t.me/safrochain)

---

📅 **Son güncelleme**: 07.10.2025  
✅ **Test edildi ve çalışıyor**  
⚠️ **Uyarı**: Bu testnet bir projedir. Ana paranızı riske atmayın.
