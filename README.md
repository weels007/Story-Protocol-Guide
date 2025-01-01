# Story-Node-Guide

## install dependencies, if needed
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

## install go, if needed
```
cd $HOME
VER="1.22.3"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

## set vars
```
echo "export MONIKER="yourmoniker"" >> $HOME/.bash_profile
echo "export STORY_CHAIN_ID="odyssey-0"" >> $HOME/.bash_profile
echo "export STORY_PORT="52"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

## story-geth
```
cd
wget -O story-geth https://github.com/piplabs/story-geth/releases/download/v0.10.1/geth-linux-amd64
chmod +x story-geth
mv story-geth $HOME/go/bin/story-geth
```

## install Story
```
cd
git clone https://github.com/piplabs/story && cd story
git checkout v0.13.0
go build -o story ./client
mv $HOME/story/story $HOME/go/bin/
```

## init story node
```
story init --moniker yourmoniker --network odyssey
```

## set seeds and peers
```
external_address=$(wget -qO- eth0.me)
sed -i.bak -e "s/^external_address *=.*/external_address = \"$external_address:26656\"/" $HOME/.story/story/config/config.toml
```
```
peers="c5c214377b438742523749bb43a2176ec9ec983c@176.9.54.69:26656,5dec0b793789d85c28b1619bffab30d5668039b7@150.136.113.152:26656,89a07021f98914fbac07aae9fbb12a92c5b6b781@152.53.102.226:26656,443896c7ec4c695234467da5e503c78fcd75c18e@80.241.215.215:26656,2df2b0b66f267939fea7fe098cfee696d6243cec@65.108.193.224:23656,7cc415203fc4c1a6e534e5fed8292467cf14d291@65.21.29.250:3610,fa294c4091379f84d0fc4a27e6163c956fc08e73@65.108.103.184:26656,81eaee3be00b21d0a124016b62fb7176fa05a4f9@185.198.49.133:33556,3508ef280392bd431ea078dec16dcfae89e8eb78@213.239.192.18:26656,b04bae4f88ca12d45fc14be29ce96837b61a72b8@65.109.49.115:26656"
sed -i -e "s|^persistent_peers *=.*|persistent_peers = \"$peers\"|" $HOME/.story/story/config/config.toml
seeds="434af9dae402ab9f1c8a8fc15eae2d68b5be3387@story-testnet-seed.itrocket.net:29656"
sed -i.bak -e "s/^seeds =.*/seeds = \"$seeds\"/" $HOME/.story/story/config/config.toml
sed -i -e "s/^filter_peers *=.*/filter_peers = \"true\"/" $HOME/.story/story/config/config.toml
```

## enable prometheus and disable indexing

```
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.story/story/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.story/story/config/config.toml
```

## create geth servie file
```
tee /etc/systemd/system/story-geth.service > /dev/null <<EOF
[Unit]
Description=Story Geth Client
After=network.target

[Service]
User=$USER
ExecStart=$HOME/go/bin/story-geth --odyssey --syncmode full --http --http.api eth,net,web3,engine --http.vhosts '*' --http.addr 127.0.0.1 --http.port 8545 --ws --ws.api eth,web3,net,txpool --ws.addr 127.0.0.1 --ws.port 8546
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

# Cosmovisor Setup

## Install Cosmovisor:
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@latest
echo "export DAEMON_NAME="story"" >> $HOME/.bash_profile
echo "export DAEMON_HOME="$HOME/.story/story"" >> $HOME/.bash_profile
source $HOME/.bash_profile
cosmovisor init $(which story)
```

## Create a directory and download the current version of story
```
mkdir -p $HOME/.story/story/cosmovisor/upgrades/v0.13.0/bin
wget -O $HOME/.story/story/cosmovisor/upgrades/v0.13.0/bin/story https://github.com/piplabs/story/releases/download/v0.13.0/story-linux-amd64
chmod +x $HOME/.story/story/cosmovisor/upgrades/v0.13.0/bin/story
```

## Update service file
```
sudo tee /etc/systemd/system/story.service > /dev/null << EOF
[Unit]
Description=story node service
After=network-online.target

[Service]
User=$USER
Environment="DAEMON_NAME=story"
Environment="DAEMON_HOME=$HOME/.story/story"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_DATA_BACKUP_DIR=$HOME/.story/story/data"
ExecStart=$(which cosmovisor) run run
Restart=on-failure
RestartSec=10
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

## enable and start geth, story
```
sudo systemctl daemon-reload
sudo systemctl enable story story-geth
sudo systemctl restart story-geth && sudo systemctl restart story
```

## check logs
```
journalctl -u story -u story-geth -f
```
```
journalctl -u story -f -o cat
```

# Create a validator

## View your validator key
```
story validator export
```

## Export EVM private key
```
story validator export --export-evm-key
```

## View EVM private key and make a key backup
```
cat $HOME/.story/story/config/private_key.txt
```

## Delete Node if you are not selected to be a validator
```
sudo systemctl stop story story-geth
sudo systemctl disable story story-geth
rm -rf $HOME/.story
sudo rm /etc/systemd/system/story.service /etc/systemd/system/story-geth.service
sudo systemctl daemon-reload
```
