# 🌌 SAFROCHAIN Node Kurulum Rehberi

![Cosmos SDK](https://img.shields.io/badge/Cosmos%20SDK-Compatible-blue)
![Ubuntu](https://img.shields.io/badge/Ubuntu-20.04%2F22.04-orange)
![Go](https://img.shields.io/badge/Go-1.23.0-00ADD8)
![Cosmovisor](https://img.shields.io/badge/Cosmovisor-Latest-green)

Bu rehber, **safrochain** node'unuzun Cosmovisor ile kurulumunu açıklar.

## 📊 Kurulum Bilgileri

| Parametre | Değer |
|---|---|
| **Proje Adı** | safrochain |
| **Chain ID** | `safro-testnet-1` |
| **Daemon Adı** | `safrochaind` |
| **Moniker** | `coinsspor` |
| **Go Versiyonu** | 1.23.0 |
| **Cosmovisor Versiyonu** | latest |
| **Port Prefix** | 53 |
| **Home Directory** | `$HOME/.safrochain` |

## 🔌 Portlar
| Servis | Port | Varsayılan |
|---|---|---|
| P2P | `53656` | 26656 |
| RPC | `53657` | 26657 |
| Proxy App | `53658` | 26658 |
| Prometheus | `53660` | 26660 |
| Pprof | `53060` | 6060 |
| API | `53317` | 1317 |
| gRPC | `53090` | 9090 |
| gRPC Web | `53091` | 9091 |
| Rosetta | `53080` | 8080 |

## 🚀 Kurulum

### 1) Paketler
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git wget htop tmux build-essential jq make lz4 gcc unzip
```

### 2) Go
```bash
cd $HOME
sudo rm -rf /usr/local/go
wget "https://golang.org/dl/go1.23.0.linux-amd64.tar.gz"
sudo tar -C /usr/local -xzf "go1.23.0.linux-amd64.tar.gz"
rm "go1.23.0.linux-amd64.tar.gz"
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' | tee -a ~/.bashrc ~/.profile >/dev/null
source ~/.bashrc || true
go version
```

### 3) Cosmovisor
```bash
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@latest
```

### 4) Dizinler
```bash
mkdir -p $HOME/.safrochain/cosmovisor/genesis/bin
mkdir -p $HOME/.safrochain/cosmovisor/upgrades
```

### 5) Binary (Kaynak Koddan Derle)
```bash
cd $HOME
git clone https://github.com/Safrochain-Org/safrochain-node.git safrochain_source
cd safrochain_source
git checkout v0.1.0

# En yaygın hedefler: make install / make build / go build
(make install) || (make build) || (go build -o build/safrochaind ./cmd/safrochaind || go build -o build/safrochaind ./...)

# Binary'i bul ve yerleştir
BIN_PATH=""
for p in "build/safrochaind" "$HOME/go/bin/safrochaind" "$(go env GOPATH)/bin/safrochaind"; do
  if [ -f "$p" ]; then BIN_PATH="$p"; break; fi
done
if [ -z "$BIN_PATH" ]; then echo "Binary build edilemedi"; exit 1; fi
install -m 0755 "$BIN_PATH" "$HOME/.safrochain/cosmovisor/genesis/bin/safrochaind"
```

### 6) Symlink
```bash
ln -sfn $HOME/.safrochain/cosmovisor/genesis $HOME/.safrochain/cosmovisor/current
sudo ln -sfn $HOME/.safrochain/cosmovisor/current/bin/safrochaind /usr/local/bin/safrochaind
```

### 7) Init & Config
```bash
safrochaind config chain-id safro-testnet-1
safrochaind config keyring-backend file
safrochaind config node tcp://localhost:53657
safrochaind init "coinsspor" --chain-id safro-testnet-1
```

### Genesis
```bash
wget "https://vault2.astrostake.xyz/testnet/safrochain/genesis.json" -O $HOME/.safrochain/config/genesis.json
```

### Addrbook
```bash
wget "https://vault2.astrostake.xyz/testnet/safrochain/addrbook.json" -O $HOME/.safrochain/config/addrbook.json
```

### Ağ
```bash

PEERS="70a40a48577174a95ef920fcc894bc048929ce80@5.189.147.191:26656,b4b711560e62b3a850193f3fa85c82e6ccf4c013@135.181.178.120:12656"
sed -i -e 's/^persistent_peers *=.*/persistent_peers = "70a40a48577174a95ef920fcc894bc048929ce80@5.189.147.191:26656,b4b711560e62b3a850193f3fa85c82e6ccf4c013@135.181.178.120:12656"/' $HOME/.safrochain/config/config.toml
```

### Gas Price
```bash
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.001usaf\"|" $HOME/.safrochain/config/app.toml
```

## ⚙️ Port Değiştirme
```bash
CUSTOM_PORT="53"
sed -i.bak -e "s%:26658%:53658%g; s%:26657%:53657%g; s%:6060%:53060%g; s%:26656%:53656%g; s%:26660%:53660%g" $HOME/.safrochain/config/config.toml
sed -i.bak -e "s%:1317%:53317%g; s%:8080%:53080%g; s%:9090%:53090%g; s%:9091%:53091%g" $HOME/.safrochain/config/app.toml

sed -i -e "s%localhost:26657%localhost:53657%g" $HOME/.safrochain/config/client.toml
safrochaind config node tcp://localhost:53657
```

## 🎮 Servis
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
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:$HOME/.safrochain/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable safrochaind
sudo systemctl start safrochaind
```

## 📝 Faydalı Komutlar
```bash
journalctl -fu safrochaind -o cat
safrochaind status 2>&1 | jq .SyncInfo
safrochaind tendermint show-node-id
safrochaind tendermint show-validator
```

---
> Not: Bu rehber 07.10.2025 tarihinde üretildi.
