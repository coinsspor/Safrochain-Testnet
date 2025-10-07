# 🌌 SAFROCHAIN Node Kurulum Rehberi

![Cosmos SDK](https://img.shields.io/badge/Cosmos%20SDK-Compatible-blue)
![Ubuntu](https://img.shields.io/badge/Ubuntu-20.04%2F22.04-orange)
![Go](https://img.shields.io/badge/Go-1.23.0-00ADD8)
![Cosmovisor](https://img.shields.io/badge/Cosmovisor-Latest-green)

Bu rehber, **safrochain** blockchain node'unuzun Cosmovisor ile profesyonel kurulumunu adım adım açıklar.

## 📋 İçindekiler

- [Sistem Gereksinimleri](#sistem-gereksinimleri)
- [Kurulum Bilgileri](#kurulum-bilgileri)
- [Port Konfigürasyonu](#port-konfigürasyonu)
- [Kurulum Adımları](#kurulum-adımları)
- [Yapılandırma Detayları](#yapılandırma-detayları)
- [Port Değiştirme](#port-değiştirme)
- [Servis Yönetimi](#servis-yönetimi)
- [Faydalı Komutlar](#faydalı-komutlar)
- [Güvenlik Önerileri](#güvenlik-önerileri)
- [Sorun Giderme](#sorun-giderme)

## 🖥️ Sistem Gereksinimleri

| Bileşen | Minimum | Önerilen |
|---------|---------|----------|
| **CPU** | 4 vCPU | 8 vCPU |
| **RAM** | 8 GB | 16 GB |
| **Depolama** | 200 GB SSD | 500 GB+ NVMe SSD |
| **İşletim Sistemi** | Ubuntu 20.04 LTS | Ubuntu 22.04 LTS |
| **Ağ** | 100 Mbps | 1 Gbps |

## 📊 Kurulum Bilgileri

| Parametre | Değer |
|-----------|-------|
| **Proje Adı** | safrochain |
| **Chain ID** | `safro-testnet-1` |
| **Daemon Adı** | `safrochaind` |
| **Moniker** | `coinsspor` |
| **Go Versiyonu** | 1.23.0 |
| **Cosmovisor Versiyonu** | latest |
| **Port Prefix** | 53 |
| **Home Directory** | `$HOME/.safrochain` |

## 🔌 Port Konfigürasyonu

### Kullanılacak Portlar

| Servis | Port | Varsayılan | Açıklama |
|--------|------|------------|----------|
| **P2P** | `53656` | 26656 | Node'lar arası iletişim |
| **RPC** | `53657` | 26657 | RPC endpoint |
| **Proxy App** | `53658` | 26658 | Uygulama proxy'si |
| **Prometheus** | `53660` | 26660 | Metrik toplama |
| **Pprof** | `53060` | 6060 | Performans profilleme |
| **API** | `53317` | 1317 | REST API |
| **gRPC** | `53090` | 9090 | gRPC servisi |
| **gRPC Web** | `53091` | 9091 | Web gRPC gateway |
| **Rosetta** | `53080` | 8080 | Rosetta API (opsiyonel) |

## 🚀 Kurulum Adımları

### 1️⃣ Sistem Güncellemeleri ve Gerekli Paketler

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

### 2️⃣ Go Kurulumu

```bash
cd $HOME
sudo rm -rf /usr/local/go
wget "https://golang.org/dl/go1.23.0.linux-amd64.tar.gz"
sudo tar -C /usr/local -xzf "go1.23.0.linux-amd64.tar.gz"
rm "go1.23.0.linux-amd64.tar.gz"

# Go environment ayarları
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
```

Go kurulumunu doğrulayın:
```bash
go version
# Çıktı: go version go1.23.0 linux/amd64
```

### 3️⃣ Cosmovisor Kurulumu

```bash
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@latest
```

### 4️⃣ Dizin Yapısını Oluşturma

```bash
mkdir -p $HOME/.safrochain/cosmovisor/genesis/bin
mkdir -p $HOME/.safrochain/cosmovisor/upgrades
```

### 5️⃣ Binary Kurulumu

#### Kaynak Koddan Derleme Metodu

```bash
cd $HOME
git clone https://github.com/Safrochain-Org/safrochain-node.git safrochain_source
cd safrochain_source
git checkout v0.1.0
make install

# Binary'yi cosmovisor dizinine kopyalama
cp $HOME/go/bin/safrochaind $HOME/.safrochain/cosmovisor/genesis/bin/
```

### 6️⃣ Symlink Oluşturma

```bash
sudo ln -s $HOME/.safrochain/cosmovisor/genesis $HOME/.safrochain/cosmovisor/current -f
sudo ln -s $HOME/.safrochain/cosmovisor/current/bin/safrochaind /usr/local/bin/safrochaind -f
```

### 7️⃣ Node Başlatma ve Konfigürasyon

```bash
# Chain ID ve keyring ayarları
safrochaind config chain-id safro-testnet-1
safrochaind config keyring-backend file
safrochaind config node tcp://localhost:53657

# Node'u initialize etme
safrochaind init "coinsspor" --chain-id safro-testnet-1
```

> **💡 Not:** Moniker isminizi kendi belirlediğiniz isim ile değiştirin!

### 8️⃣ Genesis ve Addrbook Dosyaları

#### Genesis Dosyası
```bash
wget "https://vault2.astrostake.xyz/testnet/safrochain/genesis.json" -O $HOME/.safrochain/config/genesis.json
```

#### Addrbook Dosyası
```bash
wget "https://vault2.astrostake.xyz/testnet/safrochain/addrbook.json" -O $HOME/.safrochain/config/addrbook.json
```

### 🌐 Network Konfigürasyonu

#### Seed Nodes
```bash
SEEDS="70a40a48577174a95ef920fcc894bc048929ce80@5.189.147.191:26656,b4b711560e62b3a850193f3fa85c82e6ccf4c013@135.181.178.120:12656"
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/" $HOME/.safrochain/config/config.toml
```

### ⛽ Gas Price Ayarı

```bash
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.001usaf\"|" $HOME/.safrochain/config/app.toml
```

## ⚙️ Yapılandırma Detayları

### Pruning Ayarları

```bash
# Custom pruning ayarları
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.safrochain/config/app.toml
```

### 📸 Snapshot Kurulumu

```bash
# Validator state'ini yedekleme
mv $HOME/.safrochain/data/priv_validator_state.json $HOME/.safrochain/priv_validator_state.json.backup

# Mevcut verileri temizleme
safrochaind tendermint unsafe-reset-all --home $HOME/.safrochain --keep-addr-book

# Snapshot'ı indirme ve açma
curl -L https://vault2.astrostake.xyz/testnet/safrochain/safrochain_testnet_snapshot.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.safrochain

# Validator state'ini geri yükleme
cp $HOME/.safrochain/priv_validator_state.json.backup $HOME/.safrochain/data/priv_validator_state.json
```

## 🔧 Port Değiştirme

Cosmos SDK varsayılan portlarını özel bir prefix ile değiştirmek için:

```bash
# Port prefix'inizi belirleyin (10-64 arası)
CUSTOM_PORT="53"

# config.toml dosyasındaki portları değiştirme
sed -i.bak -e "s%:26658%:${CUSTOM_PORT}658%g; \
s%:26657%:${CUSTOM_PORT}657%g; \
s%:6060%:${CUSTOM_PORT}060%g; \
s%:26656%:${CUSTOM_PORT}656%g; \
s%:26660%:${CUSTOM_PORT}660%g" $HOME/.safrochain/config/config.toml

# External address güncelleme (Public node'lar için)
sed -i -e "s%^external_address = \"\".*%external_address = \"$(wget -qO- eth0.me):${CUSTOM_PORT}656\"%g" $HOME/.safrochain/config/config.toml

# app.toml dosyasındaki portları değiştirme
sed -i.bak -e "s%:1317%:${CUSTOM_PORT}317%g; \
s%:8080%:${CUSTOM_PORT}080%g; \
s%:9090%:${CUSTOM_PORT}090%g; \
s%:9091%:${CUSTOM_PORT}091%g" $HOME/.safrochain/config/app.toml

# EVM uyumlu chainler için ek portlar (varsa)
# sed -i -e "s%:8545%:${CUSTOM_PORT}545%g; \
# s%:8546%:${CUSTOM_PORT}546%g; \
# s%:6065%:${CUSTOM_PORT}065%g" $HOME/.safrochain/config/app.toml

# client.toml dosyasını güncelleme
sed -i -e "s%localhost:26657%localhost:${CUSTOM_PORT}657%g" $HOME/.safrochain/config/client.toml

# Node yapılandırmasını güncelleme
safrochaind config node tcp://localhost:${CUSTOM_PORT}657
```

> **💡 Not:** Bu komutlar Cosmos SDK'nın varsayılan portlarını (26656, 26657, 1317 vb.) sizin belirlediğiniz prefix ile değiştirir.

## 🎮 Servis Yönetimi

### Systemd Servisi Oluşturma

```bash
sudo tee /etc/systemd/system/safrochaind.service > /dev/null <<EOF
[Unit]
Description=safrochain node service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.safrochain"
Environment="DAEMON_NAME=safrochaind"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.safrochain/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF
```

### Servisi Başlatma

```bash
sudo systemctl daemon-reload
sudo systemctl enable safrochaind
sudo systemctl start safrochaind
```

### 🔄 Peer Güncelleme (Evrensel Cosmos Peer Güncelleme)

Eğer node'unuz peer bulamıyorsa veya mevcut peer'lerinizi güncellemek isterseniz, aşağıdaki komutu kullanabilirsiniz:

```bash
# RPC URL'sini ve node dizinini projenize göre ayarlayın
RPC="RPC_URL_BURAYA"  # Örnek: https://cosmos-rpc.polkachu.com
NODE_HOME="$HOME/.safrochain"

# Peer güncelleme komutu
sed -i "s/^persistent_peers *=.*/persistent_peers = \"$(curl -s $RPC/net_info | jq -r '.result.peers[] | select(.node_info.listen_addr | contains(":")) | .node_info.id + "@" + .remote_ip + ":" + (.node_info.listen_addr | split(":")[2])' | grep -v ":$" | tr '\n' ',' | sed 's/,$//')\"/" $NODE_HOME/config/config.toml

# Node'u yeniden başlatma
sudo systemctl restart safrochaind
```

> **💡 Not:** RPC URL'sini projenize uygun bir public RPC sunucusundan alabilirsiniz. Örneğin:
> - Cosmos Hub: `https://cosmos-rpc.polkachu.com`
> - Osmosis: `https://osmosis-rpc.polkachu.com`
> - safrochain: Projenizin RPC URL'sini buraya yazın

## 📝 Faydalı Komutlar

### Node Yönetimi

| Komut | Açıklama |
|-------|----------|
| `sudo systemctl start safrochaind` | Node'u başlatır |
| `sudo systemctl stop safrochaind` | Node'u durdurur |
| `sudo systemctl restart safrochaind` | Node'u yeniden başlatır |
| `sudo systemctl status safrochaind` | Node durumunu gösterir |

### Log Takibi

```bash
# Canlı log takibi
journalctl -fu safrochaind -o cat

# Son 100 satır log
journalctl -u safrochaind -n 100 --no-pager

# Belirli zaman aralığındaki loglar
journalctl -u safrochaind --since "2024-01-01" --until "2024-01-02"
```

### Node Bilgileri

```bash
# Sync durumu kontrolü
safrochaind status 2>&1 | jq .SyncInfo

# Node ID öğrenme
safrochaind tendermint show-node-id

# Validator bilgileri
safrochaind tendermint show-validator

# Peer listesi
safrochaind tendermint show-peers
```

### Cüzdan İşlemleri

```bash
# Yeni cüzdan oluşturma
safrochaind keys add wallet

# Mevcut cüzdanı kurtarma
safrochaind keys add wallet --recover

# Cüzdan listesi
safrochaind keys list

# Cüzdan bakiyesi
safrochaind query bank balances $(safrochaind keys show wallet -a)
```

### Validator İşlemleri

```bash
# Validator oluşturma
safrochaind tx staking create-validator \
  --amount 1000000usaf \
  --from wallet \
  --commission-max-change-rate "0.01" \
  --commission-max-rate "0.2" \
  --commission-rate "0.05" \
  --min-self-delegation "1" \
  --pubkey $(safrochaind tendermint show-validator) \
  --moniker "coinsspor" \
  --chain-id safro-testnet-1 \
  --gas auto \
  --gas-adjustment 1.5 \
  --gas-prices 0.001usaf

# Validator durumu
safrochaind query staking validator $(safrochaind keys show wallet --bech val -a)
```

## 🛡️ Güvenlik Önerileri

### ✅ Yapmanız Gerekenler

1. **Düzenli Yedekleme**
   ```bash
   # Validator key yedekleme
   cp $HOME/.safrochain/config/priv_validator_key.json ~/validator_key_backup.json
   cp $HOME/.safrochain/data/priv_validator_state.json ~/validator_state_backup.json
   ```

2. **Firewall Kuralları**
   - Sadece gerekli portları açın
   - SSH portunu değiştirin
   - Fail2ban kullanın

3. **Monitoring Kurulumu**
   - Prometheus + Grafana
   - Alert sistemleri
   - Uptime monitoring

4. **Güvenlik Güncellemeleri**
   ```bash
   sudo apt update && sudo apt upgrade -y
   sudo apt autoremove -y
   ```

### ❌ Yapmamanız Gerekenler

- Private key'leri paylaşmayın
- Validator key'ini birden fazla node'da kullanmayın
- Production'da test cüzdanları kullanmayın
- API portlarını internet'e açmayın

## 🔧 Sorun Giderme

### Node Sync Olmuyor

```bash
# Peer sayısını kontrol edin
curl -s localhost:53657/net_info | jq .result.n_peers

# Genesis hash kontrolü
sha256sum $HOME/.safrochain/config/genesis.json
```

### Port Çakışması

```bash
# Kullanılan portları kontrol edin
sudo lsof -i -P -n | grep LISTEN
```

### Disk Doluluk Problemi

```bash
# Disk kullanımını kontrol edin
df -h

# Node veri boyutu
du -sh $HOME/.safrochain/
```

## 📚 Ek Kaynaklar

- [Cosmos SDK Dokümantasyonu](https://docs.cosmos.network/)
- [Cosmovisor Rehberi](https://docs.cosmos.network/main/tooling/cosmovisor)
- [Tendermint Dokümantasyonu](https://docs.tendermint.com/)

## 🤝 Destek

Kurulum sırasında sorun yaşarsanız:
- Discord: [Proje Discord Sunucusu]
- Telegram: [Proje Telegram Grubu]
- GitHub Issues: [Proje GitHub Repo]

---

> **Not**: Bu rehber 07.10.2025 tarihinde oluşturulmuştur. En güncel bilgiler için resmi dokümantasyonu kontrol ediniz.

> **Uyarı**: ✅ Otomatik binary indirme kapalı - validator için güvenli mod

---

*🌟 Başarılı bir kurulum dileriz!*
