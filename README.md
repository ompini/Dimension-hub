Dependencies Installation

**Install dependencies for building from source**
```
sudo apt update
sudo apt install -y curl git jq lz4 build-essential
```

**Install Go**
```
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.22.7.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.profile
source .profile
```

**Clone project repository**
```
cd && rm -rf dymension
git clone https://github.com/dymensionxyz/dymension
cd dymension
git checkout v3.1.0
```

**Build binary**
```
make install
```

**Prepare cosmovisor directories**
```
mkdir -p $HOME/.dymension/cosmovisor/genesis/bin
sudo ln -s $HOME/.dymension/cosmovisor/genesis $HOME/.dymension/cosmovisor/current -f
sudo ln -s $HOME/.dymension/cosmovisor/current/bin/dymd /usr/local/bin/dymd -f
```

**Move binary to cosmovisor directory**
```
mv $(which dymd) $HOME/.dymension/cosmovisor/genesis/bin
```

**Set node CLI configuration**
```
dymd config chain-id dymension_1100-1
dymd config keyring-backend file
dymd config node tcp://localhost:20557
```

**Initialize the node**
```
dymd init "Your Node Name" --chain-id dymension_1100-1
```

**Download genesis and addrbook files**
```
curl -L https://snapshots.nodejumper.io/dymension/genesis.json > $HOME/.dymension/config/genesis.json
curl -L https://snapshots.nodejumper.io/dymension/addrbook.json > $HOME/.dymension/config/addrbook.json
```

# Set seeds
sed -i -e 's|^seeds *=.*|seeds = "45bffa41836302b06310af67f012500cc0d1da31@rpc.dymension.nodestake.org:666,ebc272824924ea1a27ea3183dd0b9ba713494f83@dymension-mainnet-seed.autostake.com:27086,20e1000e88125698264454a884812746c2eb4807@seeds.lavenderfive.com:20556,400f3d9e30b69e78a7fb891f60d76fa3c73f0ecc@dymension.rpc.kjnodes.com:14659,193262e32a9d7d3fffe14073160cabc4cdfef26b@dymension-rpc.stakeandrelax.net:20556,8542cd7e6bf9d260fef543bc49e59be5a3fa9074@seed.publicnode.com:26656,c28827cb96c14c905b127b92065a3fb4cd77d7f6@seeds.whispernode.com:20556,10ed1e176d874c8bb3c7c065685d2da6a4b86475@seed-dymension.ibs.team:16676,86bd5cb6e762f673f1706e5889e039d5406b4b90@seed.dymension.node75.org:10956,258f523c96efde50d5fe0a9faeea8a3e83be22ca@seed.mainnet.dymension.aviaone.com:10290"|' $HOME/.dymension/config/config.toml

# Set minimum gas price
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "5000000000adym"|' $HOME/.dymension/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.dymension/config/app.toml

# Change ports
sed -i -e "s%:1317%:20517%; s%:8080%:20580%; s%:9090%:20590%; s%:9091%:20591%; s%:8545%:20545%; s%:8546%:20546%; s%:6065%:20565%" $HOME/.dymension/config/app.toml
sed -i -e "s%:26658%:20558%; s%:26657%:20557%; s%:6060%:20560%; s%:26656%:20556%; s%:26660%:20561%" $HOME/.dymension/config/config.toml

# Download latest chain data snapshot
curl "https://snapshots.nodejumper.io/dymension/dymension_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.dymension"

# Install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.6.0

# Create a service
sudo tee /etc/systemd/system/dymension.service > /dev/null << EOF
[Unit]
Description=Dymension Hub node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.dymension
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.dymension"
Environment="DAEMON_NAME=dymd"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable dymension.service

# Start the service and check the logs
sudo systemctl start dymension.service
sudo journalctl -u dymension.service -f --no-hostname -o cat
Secure Server Setup (Optional)

# generate ssh keys, if you don't have them already, DO IT ON YOUR LOCAL MACHINE
ssh-keygen -t rsa

# save the output, we'll use it later on instead of YOUR_PUBLIC_SSH_KEY
cat ~/.ssh/id_rsa.pub
# upgrade system packages
sudo apt update
sudo apt upgrade -y

# add new admin user
sudo adduser admin --disabled-password -q

# upload public ssh key, replace YOUR_PUBLIC_SSH_KEY with the key above
mkdir /home/admin/.ssh
echo "YOUR_PUBLIC_SSH_KEY" >> /home/admin/.ssh/authorized_keys
sudo chown admin: /home/admin/.ssh
sudo chown admin: /home/admin/.ssh/authorized_keys

echo "admin ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# disable root login, disable password authentication, use ssh keys only
sudo sed -i 's|^PermitRootLogin .*|PermitRootLogin no|' /etc/ssh/sshd_config
sudo sed -i 's|^ChallengeResponseAuthentication .*|ChallengeResponseAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PasswordAuthentication .*|PasswordAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PermitEmptyPasswords .*|PermitEmptyPasswords no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PubkeyAuthentication .*|PubkeyAuthentication yes|' /etc/ssh/sshd_config

sudo systemctl restart sshd

# install fail2ban
sudo apt install -y fail2ban

# install and configure firewall
sudo apt install -y ufw
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 9100
sudo ufw allow 26656

# make sure you expose ALL necessary ports, only after that enable firewall
sudo ufw enable

# make terminal colorful
sudo su - admin
source <(curl -s https://raw.githubusercontent.com/nodejumper-org/cosmos-scripts/master/utils/enable_colorful_bash.sh)

# update servername, if needed, replace YOUR_SERVERNAME with wanted server name
sudo hostnamectl set-hostname YOUR_SERVERNAME

# now you can logout (exit) and login again using ssh admin@YOUR_SERVER_IP
