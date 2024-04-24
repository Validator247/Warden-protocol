# Warden-protocol


API:  https://warden-api.validator247.com/

RPC:  https://warden-rpc.validator247.com/

Explorer: https://explorer.validator247.com/Warden-Testnet

Update system

    sudo apt update
    sudo apt-get install git curl build-essential make jq gcc snapd chrony lz4 tmux unzip bc -y

Install Go

    rm -rf $HOME/go
    sudo rm -rf /usr/local/go
    cd $HOME
    curl https://dl.google.com/go/go1.22.1.linux-amd64.tar.gz | sudo tar -C/usr/local -zxvf -
    cat <<'EOF' >>$HOME/.profile
    export GOROOT=/usr/local/go
    export GOPATH=$HOME/go
    export GO111MODULE=on
    export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
    EOF
    source $HOME/.profile
    go version

Install node

        cd $HOME
        rm -rf wardenprotocol
        git clone https://github.com/warden-protocol/wardenprotocol.git
        cd wardenprotocol
        git checkout v0.3.0
        make install-wardend
        wardend version


Initialize Node

        wardend init NodeName --chain-id=buenavista-1

Download Genesis

        wget -O genesis.json https://raw.githubusercontent.com/Validator247/Warden-protocol/main/genesis.json

Download addrbook

        wget -O addrbook.json https://raw.githubusercontent.com/Validator247/Warden-protocol/main/addrbook.json

 set seeds and peers

    SEEDS="8288657cb2ba075f600911685670517d18f54f3b@warden-testnet-seed.itrocket.net:18656"
    PEERS="b14f35c07c1b2e58c4a1c1727c89a5933739eeea@warden-testnet- peer.itrocket.net:18656,61446070887838944c455cb713a7770b41f35ac5@37.60.249.101:26656,dc0122e37c203dec43306430a1f1879650653479@37.27.97.16:26656,846bef6b31b20b075ff5b574d9733163b9b1958c@62.169.21.90:26656,8fa927f5255b24b28757008ae279ff6ef9a6eeed@65.21.69.53:18656,6fb5cf2179ca9dd98ababd1c8d29878b2021c5c3@146.19.24.175:26856,aa59909de26202791f252365b6d62fdd766e11df@163.172.64.81:26656,de523309ae6b47db097355f18f98ec7b65b8d373@78.47.92.231:26656,bda08962882048fea4331fcf96ad02789671700e@65.21.202.124:35656,8fa48bbd20d316382f339bd00f31d3a2678682d2@144.76.29.90:26656"
    sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.warden/config/config.toml



set minimum gas price, enable prometheus and disable indexing

        sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.0025uward"|g' $HOME/.warden/config/app.toml
        sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.warden/config/config.toml
        sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.warden/config/config.toml  

Create Service

        sudo tee /etc/systemd/system/wardend.service > /dev/null <<EOF
        [Unit]
        Description=wardend Daemon
        After=network-online.target

        [Service]
        User=$USER
        ExecStart=$(which wardend) start
        Restart=always
        RestartSec=3
        LimitNOFILE=65535

        [Install]
        WantedBy=multi-user.target
        EOF

        sudo systemctl daemon-reload
        sudo systemctl enable wardend

Download Snapshot

       sudo systemctl stop wardend

       cp $HOME/.warden/data/priv_validator_state.json $HOME/.warden/priv_validator_state.json.backup

       rm -rf $HOME/.warden/data $HOME/.warden/wasmPath
       curl https://snapshot.validatorvn.com/warden/data.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.warden

       mv $HOME/.warden/priv_validator_state.json.backup $HOME/.warden/data/priv_validator_state.json

       sudo systemctl restart wardend && sudo journalctl -u wardend -f -o cat

State Sync

        sudo systemctl stop wardend

        SNAP_RPC="https://warden-rpc.validatorvn.com:443"
        cp $HOME/.warden/data/priv_validator_state.json $HOME/.warden/priv_validator_state.json.backup
        wardend tendermint unsafe-reset-all --home ~/.warden/ --keep-addr-book

        LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height); \
        BLOCK_HEIGHT=$((LATEST_HEIGHT - 2000)); \
        TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)
        echo $LATEST_HEIGHT $BLOCK_HEIGHT $TRUST_HASH

        sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
        s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"| ; \
        s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
        s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"|" ~/.warden/config/config.toml
        more ~/.warden/config/config.toml | grep 'rpc_servers'
        more ~/.warden/config/config.toml | grep 'trust_height'
        more ~/.warden/config/config.toml | grep 'trust_hash'

        sudo mv $HOME/.warden/priv_validator_state.json.backup $HOME/.warden/data/priv_validator_state.json

        sudo systemctl restart wardend && journalctl -u wardend -f -o cat
        


# Create a validator
If you want to create a validator in the testnet, follow the instructions in the Creating a validator section.

Create Wallet

    wardend keys add wallet



faucet token 

    curl -XPOST -d '{"address": "your_wallet"}' https://faucet.buenavista.wardenprotocol.org

Check Banlance

    wardend q bank balances $(wardend keys show wallet -a)

# Create a new validator
Once the node is synced and you have the required WARD, you can become a validator.

To create a validator and initialize it with a self-delegation, you need to create a validator.json file and submit a create-validator transaction.

Obtain your validator public key by running the following command:

    wardend comet show-validator

The output will be similar to this (with a different key):

    {"@type":"/cosmos.crypto.ed25519.PubKey","key":"lR1d7YBVK5jYijOfWVKRFoWCsS4dg3kagT7LB9GnG8I="}

Create validator.json file

    nano validator.json

The validator.json file has the following format: Change your personal information accordingly

    {    
    "pubkey": {"@type":"/cosmos.crypto.ed25519.PubKey","key":"lR1d7YBVK5jYijOfWVKRFoWCsS4dg3kagT7LB9GnG8I="},
    "amount": "1000000uward",
    "moniker": "your-node-moniker",
    "identity": "your_Keybase",
    "website": "your_website",
    "security": "your_email",
    "details": "I love validator247",
    "commission-rate": "0.1",
    "commission-max-rate": "0.2",
    "commission-max-change-rate": "0.01",
    "min-self-delegation": "1"
    }

Finally, we're ready to submit the transaction to create the validator:

    wardend tx staking create-validator validator.json \
    --from=<key-name> \
    --chain-id=buenavista-1 \
    --fees=500uward

Explorer

    https://explorer.validator247.com/Warden-Testnet


Delegate Token to your own validator

        wardend tx staking delegate $(wardend keys show wallet --bech val -a)  1000000uward \
        --from=wallet \
        --chain-id=buenavista-1 \
        --fees=500uward

Withdraw rewards and commission from your validator

        wardend tx distribution withdraw-rewards $(wardend keys show wallet --bech val -a) \
        --from wallet \
        --commission \
        --chain-id=buenavista-1 \
        --fees=500uward

Unjail validator

        wardend tx slashing unjail \
        --from=wallet \
        --chain-id=buenavista-1 \
        --fees=500uward


Services Management

        # Reload Service
        sudo systemctl daemon-reload
        # Enable Service
        sudo systemctl enable wardend
        # Disable Service
        sudo systemctl disable wardend
        # Start Service
        sudo systemctl start wardend
        # Stop Service
        sudo systemctl stop wardend
        # Restart Service
        sudo systemctl restart wardend
        # Check Service Status
        sudo systemctl status wardend
        # Check Service Logs
        sudo journalctl -u wardend -f --no-hostname -o cat

 Backup Validator

         cat $HOME/.warden/config/priv_validator_key.json

Remove node

        sudo systemctl stop wardend && sudo systemctl disable wardend && sudo rm /etc/systemd/system/wardend.service && sudo systemctl daemon-reload && rm -rf $HOME/.warden && $HOME/wardenprotocol
  # DONE 
    

        
