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

Node Installation

Node Name

Your Node Name
Port prefix

156
# Clone project repository
cd && rm -rf Passage3D
git clone https://github.com/envadiv/Passage3D
cd Passage3D
git checkout v2.4.0

# Build binary
make install

# Prepare cosmovisor directories
mkdir -p $HOME/.passage/cosmovisor/genesis/bin
ln -s $HOME/.passage/cosmovisor/genesis $HOME/.passage/cosmovisor/current -f

# Copy binary to cosmovisor directory
cp $(which passage) $HOME/.passage/cosmovisor/genesis/bin

# Set node CLI configuration
passage config chain-id passage-2
passage config keyring-backend file
passage config node tcp://localhost:15657

# Initialize the node
passage init "Your Node Name" --chain-id passage-2

# Download genesis and addrbook files
curl -L https://snapshots.nodejumper.io/passage/genesis.json > $HOME/.passage/config/genesis.json
curl -L https://snapshots.nodejumper.io/passage/addrbook.json > $HOME/.passage/config/addrbook.json

# Set seeds
sed -i -e 's|^seeds *=.*|seeds = "aebb8431609cb126a977592446f5de252d8b7fa1@104.236.201.138:26656,b6beabfb9309330944f44a1686742c2751748b83@5.161.47.163:26656,7a9a36630523f54c1a0d56fc01e0e153fd11a53d@167.235.24.145:26656,ade4d8bc8cbe014af6ebdf3cb7b1e9ad36f412c0@seeds.polkachu.com:15656,20e1000e88125698264454a884812746c2eb4807@seeds.lavenderfive.com:15656,ebc272824924ea1a27ea3183dd0b9ba713494f83@passage-mainnet-seed.autostake.com:26916,8542cd7e6bf9d260fef543bc49e59be5a3fa9074@seed.publicnode.com:26656,df949a46ae6529ae1e09b034b49716468d5cc7e9@seeds.stakerhouse.com:10556,2b238d2c05c47629e03608a6107e156fcb50344c@65.108.101.158:20556,526d07b882df4cb820a8b9df819e14532d1811b0@seed-passage.ibs.team:16666"|' $HOME/.passage/config/config.toml

# Set minimum gas price
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.001upasg"|' $HOME/.passage/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.passage/config/app.toml

# Enable prometheus
sed -i -e 's|^prometheus *=.*|prometheus = true|' $HOME/.passage/config/config.toml

# Change ports
sed -i -e "s%:1317%:15617%; s%:8080%:15680%; s%:9090%:15690%; s%:9091%:15691%; s%:8545%:15645%; s%:8546%:15646%; s%:6065%:15665%" $HOME/.passage/config/app.toml
sed -i -e "s%:26658%:15658%; s%:26657%:15657%; s%:6060%:15660%; s%:26656%:15656%; s%:26660%:15661%" $HOME/.passage/config/config.toml

# Download latest chain data snapshot
curl "https://snapshots.nodejumper.io/passage/passage_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.passage"

# Install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.7.0

# Create a service
sudo tee /etc/systemd/system/passage.service > /dev/null << EOF
[Unit]
Description=Passage node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.passage
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.passage"
Environment="DAEMON_NAME=passage"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable passage.service

# Start the service and check the logs
sudo systemctl start passage.service
sudo journalctl -u passage.service -f --no-hostname -o cat
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
