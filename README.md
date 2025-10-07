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
- [Yapılandırma](#yapılandırma)
- [Servis Yönetimi](#servis-yönetimi)
- [Faydalı Komutlar](#faydalı-komutlar)

## 🖥️ Sistem Gereksinimleri

| Bileşen | Minimum | Önerilen |
|---------|---------|----------|
| CPU | 4 vCPU | 8 vCPU |
| RAM | 8 GB | 16 GB |
| Depolama | 200 GB SSD | 500 GB+ NVMe SSD |
| İşletim Sistemi | Ubuntu 20.04 LTS | Ubuntu 22.04 LTS |
| Ağ | 100 Mbps | 1 Gbps |

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

| Servis | Port | Varsayılan | Açıklama |
|--------|------|------------|----------|
| P2P | `53656` | 26656 | Node'lar arası iletişim |
| RPC | `53657` | 26657 | RPC endpoint |
| Proxy App | `53658` | 26658 | Uygulama proxy'si |
| Prometheus | `53660` | 26660 | Metrik toplama |
| Pprof | `53060` | 6060 | Performans profilleme |
| API | `53317` | 1317 | REST API |
| gRPC | `53090` | 9090 | gRPC servisi |
| gRPC Web | `53091` | 9091 | Web gRPC gateway |
| Rosetta | `53080` | 8080 | Rosetta API |

## 🚀 Kurulum Adımları

### 1️⃣ Sistem Güncellemeleri ve Gerekli Paketler
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git wget htop tmux build-essential jq make lz4 gcc unzip
```

### 2️⃣ Go Kurulumu
```bash
cd $HOME
# Mevcut Go kurulumunu temizle
sudo rm -rf /usr/local/go

# Go'yu indir ve kur
wget "https://golang.org/dl/go1.23.0.linux-amd64.tar.gz"
sudo tar -C /usr/local -xzf "go1.23.0.linux-amd64.tar.gz"
rm "go1.23.0.linux-amd64.tar.gz"

# Go environment ayarları
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> ~/.bash_profile
source $HOME/.bash_profile

# Go kurulumunu doğrulayın
go version
```

### 3️⃣ Cosmovisor Kurulumu
```bash
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@latest

# Cosmovisor kurulumunu doğrula
cosmovisor version
```

### 4️⃣ Dizin Yapısını Oluşturma
```bash
# Cosmovisor dizinlerini oluştur
mkdir -p $HOME/.safrochain/cosmovisor/genesis/bin
mkdir -p $HOME/.safrochain/cosmovisor/upgrades

# Doğrulama
ls -la $HOME/.safrochain/cosmovisor/
```

### 5️⃣ Binary Kurulumu (Kaynak Koddan Derleme)
```bash
cd $HOME

# Kaynak kodunu klonla
git clone https://github.com/Safrochain-Org/safrochain-node.git safrochain_source
cd safrochain_source

# Belirtilen versiyona geç
git checkout v0.1.0

# Derle (projeden projeye değişebilir)
make install || make build || go install ./cmd/safrochaind

# Binary'nin kurulduğu yeri bul
BINARY_PATH=""
if [ -f "$HOME/go/bin/safrochaind" ]; then
    BINARY_PATH="$HOME/go/bin/safrochaind"
elif [ -f "./build/safrochaind" ]; then
    BINARY_PATH="./build/safrochaind"
elif [ -f "./bin/safrochaind" ]; then
    BINARY_PATH="./bin/safrochaind"
else
    echo "Binary bulunamadı, manuel olarak bulun:"
    find . -name "safrochaind" -type f 2>/dev/null
    find $HOME/go -name "safrochaind" -type f 2>/dev/null
fi

# Binary'yi cosmovisor dizinine kopyala
if [ -n "$BINARY_PATH" ]; then
    cp "$BINARY_PATH" $HOME/.safrochain/cosmovisor/genesis/bin/
    chmod +x $HOME/.safrochain/cosmovisor/genesis/bin/safrochaind
    echo "Binary başarıyla kopyalandı: $BINARY_PATH"
else
    echo "Binary otomatik bulunamadı, manuel kopyalayın"
fi

# WASM desteği kontrolü ve libwasmvm kurulumu
echo "Binary bağımlılıkları kontrol ediliyor..."
if ldd $HOME/.safrochain/cosmovisor/genesis/bin/safrochaind 2>/dev/null | grep -q wasmvm; then
    echo "📦 WASM desteği tespit edildi. libwasmvm yükleniyor..."
    
    # Önce sistemde var mı kontrol et
    if [ ! -f "/usr/lib/libwasmvm.x86_64.so" ] && [ ! -f "/usr/lib/x86_64-linux-gnu/libwasmvm.x86_64.so" ]; then
        # Farklı versiyonları dene
        WASMVM_VERSION="v2.1.3"
        wget -q "https://github.com/CosmWasm/wasmvm/releases/download/${WASMVM_VERSION}/libwasmvm.x86_64.so" -O /tmp/libwasmvm.x86_64.so
        
        if [ ! -f "/tmp/libwasmvm.x86_64.so" ]; then
            echo "v2.1.3 bulunamadı, v2.0.1 deneniyor..."
            WASMVM_VERSION="v2.0.1"
            wget -q "https://github.com/CosmWasm/wasmvm/releases/download/${WASMVM_VERSION}/libwasmvm.x86_64.so" -O /tmp/libwasmvm.x86_64.so
        fi
        
        if [ -f "/tmp/libwasmvm.x86_64.so" ]; then
            sudo mv /tmp/libwasmvm.x86_64.so /usr/lib/
            sudo ldconfig
            echo "✅ libwasmvm ${WASMVM_VERSION} başarıyla yüklendi"
        else
            echo "⚠️ libwasmvm indirilemedi, manuel kurulum gerekebilir"
        fi
    else
        echo "✅ libwasmvm zaten yüklü"
    fi
fi

# Diğer eksik kütüphaneleri kontrol et
MISSING_LIBS=$(ldd $HOME/.safrochain/cosmovisor/genesis/bin/safrochaind 2>/dev/null | grep "not found" | awk '{print $1}')
if [ -n "$MISSING_LIBS" ]; then
    echo "⚠️ Eksik kütüphaneler tespit edildi:"
    echo "$MISSING_LIBS"
    echo "Sistem kütüphanelerini güncelliyorum..."
    sudo apt update && sudo apt install -y build-essential libc6-dev libssl-dev
    sudo ldconfig
fi

# Binary'yi test et
echo "Binary test ediliyor..."
if $HOME/.safrochain/cosmovisor/genesis/bin/safrochaind version 2>/dev/null; then
    echo "✅ Binary başarıyla çalışıyor"
else
    echo "⚠️ Binary çalışmıyor. Olası çözümler:"
    echo "1. libwasmvm versiyonunu kontrol edin"
    echo "2. CGO_ENABLED=1 ile yeniden derleyin"
    echo "3. ldd ile eksik kütüphaneleri kontrol edin"
    
    # Alternatif derleme önerisi
    echo ""
    echo "Alternatif derleme komutları:"
    echo "CGO_ENABLED=1 make install"
    echo "veya"
    echo "go build -tags 'netgo ledger' ./cmd/safrochaind"
fi
```

### 6️⃣ Symlink Oluşturma
```bash
# Cosmovisor için symlink
sudo ln -s $HOME/.safrochain/cosmovisor/genesis $HOME/.safrochain/cosmovisor/current -f

# Sistem genelinde kullanım için symlink
sudo ln -s $HOME/.safrochain/cosmovisor/current/bin/safrochaind /usr/local/bin/safrochaind -f

# Doğrulama
ls -la $HOME/.safrochain/cosmovisor/current/bin/
which safrochaind
safrochaind version
```

### 7️⃣ Node Başlatma ve Konfigürasyon
```bash
# Chain ID ayarı
safrochaind config chain-id safro-testnet-1

# Keyring backend ayarı
safrochaind config keyring-backend file

# Node RPC ayarı
safrochaind config node tcp://localhost:53657

# Node'u initialize etme
safrochaind init "coinsspor" --chain-id safro-testnet-1

# Init başarılı mı kontrol et
ls -la $HOME/.safrochain/config/
```
### 📜 Genesis Dosyası
```bash
wget "https://vault2.astrostake.xyz/testnet/safrochain/genesis.json" -O $HOME/.safrochain/config/genesis.json
```
### 📖 Addrbook Dosyası
```bash
wget "https://vault2.astrostake.xyz/testnet/safrochain/addrbook.json" -O $HOME/.safrochain/config/addrbook.json
```

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
sudo systemctl status safrochaind
```

## 📝 Faydalı Komutlar

### Node Yönetimi
```bash
# Log takibi
journalctl -fu safrochaind -o cat

# Sync durumu
safrochaind status 2>&1 | jq .SyncInfo

# Node ID
safrochaind tendermint show-node-id
```

---
> 📅 Bu rehber 07.10.2025 tarihinde oluşturulmuştur.
