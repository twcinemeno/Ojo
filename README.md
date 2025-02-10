Hardware Requirements

Minimum

4CPU 8RAM 100GB
Recommended

4CPU 16RAM 200GB
Rent On Hetzner | Rent On OVH
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
cd && rm -rf ojo
git clone https://github.com/ojo-network/ojo
cd ojo
git checkout v0.1.2
```

**Build binary**
```
make install
```

# Prepare cosmovisor directories
mkdir -p $HOME/.ojo/cosmovisor/genesis/bin
ln -s $HOME/.ojo/cosmovisor/genesis $HOME/.ojo/cosmovisor/current -f

# Copy binary to cosmovisor directory
cp $(which ojod) $HOME/.ojo/cosmovisor/genesis/bin

# Set node CLI configuration
ojod config chain-id ojo-devnet
ojod config keyring-backend test
ojod config node tcp://localhost:21657

# Initialize the node
ojod init "Your Node Name" --chain-id ojo-devnet

# Download genesis and addrbook files
curl -L https://snapshots-testnet.nodejumper.io/ojo/genesis.json > $HOME/.ojo/config/genesis.json
curl -L https://snapshots-testnet.nodejumper.io/ojo/addrbook.json > $HOME/.ojo/config/addrbook.json

# Set seeds
sed -i -e 's|^seeds *=.*|seeds = "7186f24ace7f4f2606f56f750c2684d387dc39ac@ojo-testnet-seed.itrocket.net:12656"|' $HOME/.ojo/config/config.toml

# Set minimum gas price
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.01uojo"|' $HOME/.ojo/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.ojo/config/app.toml

# Enable prometheus
sed -i -e 's|^prometheus *=.*|prometheus = true|' $HOME/.ojo/config/config.toml

# Change ports
sed -i -e "s%:1317%:21617%; s%:8080%:21680%; s%:9090%:21690%; s%:9091%:21691%; s%:8545%:21645%; s%:8546%:21646%; s%:6065%:21665%" $HOME/.ojo/config/app.toml
sed -i -e "s%:26658%:21658%; s%:26657%:21657%; s%:6060%:21660%; s%:26656%:21656%; s%:26660%:21661%" $HOME/.ojo/config/config.toml

# Download latest chain data snapshot
curl "https://snapshots-testnet.nodejumper.io/ojo/ojo_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.ojo"

# Install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.7.0

# Create a service
sudo tee /etc/systemd/system/ojo.service > /dev/null << EOF
[Unit]
Description=Ojo node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.ojo
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.ojo"
Environment="DAEMON_NAME=ojod"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable ojo.service

# Start the service and check the logs
sudo systemctl start ojo.service
sudo journalctl -u ojo.service -f --no-hostname -o cat
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
