# Server Setup Guide: LVM, Ethereum Node (Sepolia), and Aztec Sequencer

This guide walks you through setting up a Linux server, configuring LVM for disk management, installing and syncing an Ethereum Sepolia testnet node (Geth and Lighthouse), and finally setting up an Aztec Sequencer.

**Assumptions:**
* You have root or sudo access to the server.
* You are comfortable with the Linux command line.
* The initial disk setup commands (`installimage`) are specific to a hosting provider (e.g., Hetzner). Adapt as necessary for your environment.

## Phase 1: Initial Server OS Installation and LVM Configuration

This phase is typically done via a rescue system or a specialized installation image provided by your server host.

1.  **Enter Rescue Mode & Prepare for OS Installation:**
    * Boot your server into its rescue system.
    * Initiate the OS installation process (e.g., using `installimage`).

2.  **Configure Disk Partitioning with LVM:**
    Modify the installation configuration file to use LVM. The goal is to create a flexible storage layout.

    ```ini
    # Example installimage configuration for LVM
    # This will create a software RAID 0 if you have multiple identical drives specified for it.
    # If SWRAID is not desired or not applicable, remove the SWRAID line and adjust PART lines.
    # This example assumes you have drives that will be aggregated into vg0.

    # DRIVE1 /dev/nvme0n1 # Example, adjust to your actual primary drive for OS
    # DRIVE2 /dev/nvme1n1 # Example, spare disk 1
    # DRIVE3 /dev/nvme2n1 # Example, spare disk 2

    SWRAID 0 # Set to 1 for RAID 1 if desired and applicable

    # Boot partition
    PART /boot ext3 1024M

    # LVM physical volume, uses all remaining space on the drive(s) included in SWRAID
    # or the specified drive if not using SWRAID.
    PART lvm vg0 all

    # Logical Volumes within the vg0 volume group
    LV vg0 root / ext4 50G
    LV vg0 swap swap swap 32G
    LV vg0 home /home ext4 all # Assign all remaining space in vg0 to /home
    ```

    * **`SWRAID 0`**: Configures software RAID 0 (striping). Adjust as needed.
    * **`PART /boot ext3 1024M`**: Creates a 1GB boot partition.
    * **`PART lvm vg0 all`**: Dedicates remaining disk space to an LVM physical volume in volume group `vg0`.
    * **`LV vg0 root / ext4 50G`**: 50GB logical volume for root.
    * **`LV vg0 swap swap swap 32G`**: 32GB swap logical volume.
    * **`LV vg0 home /home ext4 all`**: Logical volume for `/home` using all remaining space in `vg0`.

3.  **Complete OS Installation and Reboot:**
    * Save the configuration and proceed with the OS installation.
    * Reboot into the newly installed Ubuntu system.

## Phase 2: Expanding LVM with Spare Disks (Post-OS Install)

1.  **Identify Spare Disks:**
    Use `lsblk` and `fdisk -l` to identify spare disks (e.g., `/dev/nvme1n1`, `/dev/nvme2n1`).

    ```bash
    lsblk
    sudo fdisk -l
    ```

2.  **Wipe Filesystem Signatures (Caution!):**
    Ensure disks do not contain needed data.

    ```bash
    sudo wipefs -a /dev/nvme1n1 # Replace with your actual spare disk name
    sudo wipefs -a /dev/nvme2n1 # Replace with your actual spare disk name
    ```

3.  **Extend the Volume Group:**
    Add new physical volumes to `vg0`.

    ```bash
    sudo vgextend vg0 /dev/nvme1n1 /dev/nvme2n1 # Replace with your actual spare disk names
    ```

4.  **Verify LVM Status:**

    ```bash
    vgs
    pvs
    ```

5.  **Extend the Logical Volume (`/home`):**
    * **Extend by a specific amount (e.g., 100GB):**
        ```bash
        sudo lvextend -L +100G /dev/vg0/home
        sudo resize2fs /dev/vg0/home # Resize the ext4 filesystem
        ```

    * **Extend by all available free space:**
        ```bash
        sudo lvextend -l +100%FREE /dev/vg0/home
        sudo resize2fs /dev/vg0/home # Resize the ext4 filesystem
        ```

6.  **Verify Disk Space:**

    ```bash
    df -h
    ```

## Phase 3: Basic Server Setup & Firewall

1.  **Update System and Install Essential Packages:**

    ```bash
    sudo apt update && sudo apt upgrade -y
    sudo apt install -y curl wget git jq build-essential libssl-dev pkg-config screen htop glances unattended-upgrades
    ```

2.  **Install and Configure Chrony (Time Synchronization):**

    ```bash
    sudo apt install -y chrony
    sudo systemctl enable chrony
    sudo systemctl start chrony
    chronyc sources # Verify sync status
    ```

3.  **Configure Firewall (UFW):**

    * **Allow SSH (Essential):**
        ```bash
        sudo ufw allow 22/tcp
        ```

    * **Allow Aztec Sequencer Ports (Check Aztec documentation for current ports):**
        ```bash
        sudo ufw allow 40400/tcp # Example p2p for sequencer
        sudo ufw allow 8080/tcp   # Example rpc for sequencer
        ```

    * **Allow Ethereum Consensus Layer (CL) & Execution Layer (EL) Ports:**
        ```bash
        sudo ufw allow 9000/tcp   # Lighthouse default p2p (TCP)
        sudo ufw allow 9000/udp   # Lighthouse default p2p (UDP)
        sudo ufw allow 30303/tcp  # Geth default p2p (TCP)
        sudo ufw allow 30303/udp  # Geth default p2p (UDP)
        ```

    * **(Optional) Deleting a Firewall Rule:**
        ```bash
        sudo ufw status numbered
        # sudo ufw delete <rule_number> # Example: sudo ufw delete 9
        ```

    * **Restricted Access for RPC/API Ports (Recommended):**
        Replace `YOUR_TRUSTED_IP_HERE` with the specific IP.
        ```bash
        sudo ufw allow from YOUR_TRUSTED_IP_HERE to any port 8545 proto tcp # Geth HTTP RPC
        sudo ufw allow from YOUR_TRUSTED_IP_HERE to any port 8546 proto tcp # Geth WS RPC
        sudo ufw allow from YOUR_TRUSTED_IP_HERE to any port 5052 proto tcp # Lighthouse Beacon HTTP API
        ```

    * **Enable UFW:**
        ```bash
        sudo ufw enable
        sudo ufw status verbose
        ```

## Phase 4: Ethereum Execution Layer (Geth - Sepolia Testnet)

1.  **Add Ethereum PPA and Install Geth:**

    ```bash
    sudo add-apt-repository -y ppa:ethereum/ethereum
    sudo apt update
    sudo apt install geth -y
    geth version
    ```

2.  **Create Directories:**
    Using `/home` due to LVM setup for ample space.

    ```bash
    mkdir -p /home/ethereum_sepolia/execution
    mkdir -p /home/ethereum_sepolia/jwt
    ```

3.  **Generate JWT Secret:**
    For secure EL/CL communication.

    ```bash
    openssl rand -hex 32 | tr -d "\n" > /home/ethereum_sepolia/jwt/jwt.hex
    sudo chmod 640 /home/ethereum_sepolia/jwt/jwt.hex
    ```

4.  **Create Geth Systemd Service File:**
    Path: `/etc/systemd/system/geth-sepolia.service`

    ```ini
    [Unit]
    Description=Geth Execution Client (Sepolia)
    After=network-online.target
    Wants=network-online.target

    [Service]
    User=root # Or a dedicated user e.g., 'geth'
    Group=root # Or a dedicated group e.g., 'geth'
    Restart=always
    RestartSec=5
    ExecStart=/usr/bin/geth \
        --sepolia \
        --datadir /home/ethereum_sepolia/execution \
        --http \
        --http.addr 0.0.0.0 \
        --http.port 8545 \
        --http.api eth,net,engine,web3,admin \
        --ws \
        --ws.addr 0.0.0.0 \
        --ws.port 8546 \
        --ws.api eth,net,engine,web3 \
        --authrpc.addr 127.0.0.1 \
        --authrpc.port 8551 \
        --authrpc.vhosts "*" \
        --authrpc.jwtsecret /home/ethereum_sepolia/jwt/jwt.hex \
        --http.corsdomain "*" \
        --http.vhosts "*" \
        --cache 24576 \
        --maxpeers 125
    LimitNOFILE=65535

    [Install]
    WantedBy=multi-user.target
    ```
    *Adjust `--cache` (e.g., 24576MB for ~24GB RAM) and `--maxpeers` as needed.*

5.  **Reload Systemd, Enable and Start Geth:**

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable geth-sepolia
    sudo systemctl start geth-sepolia
    sudo systemctl status geth-sepolia
    journalctl -u geth-sepolia -f --no-pager # Follow logs
    ```

## Phase 5: Ethereum Consensus Layer (Lighthouse - Sepolia Testnet)

1.  **Download and Install Lighthouse:**
    Check [Lighthouse releases](https://github.com/sigp/lighthouse/releases) for the latest version. Example uses `v7.0.1`.

    ```bash
    cd /tmp
    curl -LO [https://github.com/sigp/lighthouse/releases/download/v7.0.1/lighthouse-v7.0.1-x86_64-unknown-linux-gnu.tar.gz](https://github.com/sigp/lighthouse/releases/download/v7.0.1/lighthouse-v7.0.1-x86_64-unknown-linux-gnu.tar.gz)
    tar -xvf lighthouse-v7.0.1-x86_64-unknown-linux-gnu.tar.gz
    sudo cp lighthouse /usr/local/bin/
    lighthouse --version
    ```

2.  **Create Directories:**

    ```bash
    mkdir -p /home/ethereum_sepolia/consensus
    ```

3.  **Create Lighthouse Systemd Service File:**
    Path: `/etc/systemd/system/lighthouse-sepolia.service`

    ```ini
    [Unit]
    Description=Lighthouse Consensus Client (Sepolia Beacon Node)
    Wants=geth-sepolia.service
    After=geth-sepolia.service

    [Service]
    User=root # Or a dedicated user e.g., 'lighthouse'
    Group=root # Or a dedicated group e.g., 'lighthouse'
    Restart=always
    RestartSec=5
    ExecStart=/usr/local/bin/lighthouse beacon \
        --network sepolia \
        --datadir /home/ethereum_sepolia/consensus \
        --execution-endpoint [http://127.0.0.1:8551](http://127.0.0.1:8551) \
        --execution-jwt /home/ethereum_sepolia/jwt/jwt.hex \
        --http \
        --http-address 0.0.0.0 \
        --http-port 5052 \
        --metrics \
        --metrics-address 0.0.0.0 \
        --metrics-port 5054 \
        --checkpoint-sync-url [https://checkpoint-sync.sepolia.ethpandaops.io](https://checkpoint-sync.sepolia.ethpandaops.io) \
        --target-peers 125
    LimitNOFILE=65535

    [Install]
    WantedBy=multi-user.target
    ```
    *`--http-address 0.0.0.0` makes API external; use `127.0.0.1` for local only. Adjust `--target-peers` (default ~75) if needed.*

4.  **Reload Systemd, Enable and Start Lighthouse:**

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable lighthouse-sepolia
    sudo systemctl start lighthouse-sepolia
    sudo systemctl status lighthouse-sepolia
    ```

## Phase 6: Verify Ethereum Node Sync Status

1.  **Check External Beacon Chain Explorer:**
    Visit `https://light-sepolia.beaconcha.in/` for current slot.

2.  **Check Geth Sync Status:**

    ```bash
    geth attach [http://127.0.0.1:8545](http://127.0.0.1:8545)
    ```
    In Geth console:
    ```javascript
    eth.syncing
    ```
    *`false` means synced. An object with block numbers means still syncing.*

3.  **Check Lighthouse Sync Status & Logs:**

    ```bash
    journalctl -u lighthouse-sepolia -f --no-pager # Follow Lighthouse logs
    curl -s http://localhost:5052/eth/v1/node/syncing
    ```
    *Look for `is_syncing: false` and `sync_distance: 0`.*

4.  **Check Geth Block Number via RPC:**

    ```bash
    curl -X POST -H "Content-Type: application/json" \
    --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
    http://localhost:8545
    ```
    *Compare hex output (converted to decimal) with a Sepolia block explorer.*

**Wait for full sync before Aztec setup.**

## Phase 7: Docker Installation and Configuration

1.  **Remove Old Docker Versions (If Any):**

    ```bash
    sudo systemctl stop docker
    sudo apt-get purge -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras docker.io docker-engine
    sudo rm -rf /var/lib/docker /etc/docker
    ```

2.  **Create Docker Data Directory on LVM:**
    Store Docker data on `/home` LVM.

    ```bash
    sudo mkdir -p /home/docker-data
    sudo chown root:root /home/docker-data
    ```

3.  **Install Docker Engine:**

    ```bash
    sudo apt-get update
    sudo apt-get install -y ca-certificates curl gnupg
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg

    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update
    sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```

4.  **Configure Docker Data Root:**
    Create `/etc/docker/daemon.json`:

    ```json
    {
      "data-root": "/home/docker-data"
    }
    ```

5.  **Start and Verify Docker:**

    ```bash
    sudo systemctl restart docker # Use restart to apply daemon.json
    sudo systemctl enable docker
    sudo systemctl status docker --no-pager
    sudo docker info | grep "Docker Root Dir" # Should show /home/docker-data
    sudo docker run hello-world # Test
    ```

## Phase 8: Aztec Sequencer Setup

1.  **Install Aztec:**

    ```bash
    bash -i <(curl -s [https://install.aztec.network](https://install.aztec.network))
    ```

2.  **Update PATH:**

    ```bash
    echo 'export PATH="$HOME/.aztec/bin:$PATH"' >> ~/.bashrc
    source ~/.bashrc
    ```
    *Adds to `/root/.bashrc` if running as root.*

3.  **Initialize Aztec for Alpha Testnet:**

    ```bash
    aztec-up alpha-testnet
    ```

4.  **Adjust Firewall for Docker to Host Communication:**

    ```bash
    sudo ufw allow in on docker0 to any port 8545 proto tcp comment 'Allow Docker containers to Geth RPC'
    sudo ufw allow in on docker0 to any port 5052 proto tcp comment 'Allow Docker containers to Lighthouse RPC'
    # sudo ufw reload # If needed
    ```

5.  **Create Aztec Sequencer Systemd Service File:**
    Path: `/etc/systemd/system/aztec-sequencer.service`

    **IMPORTANT:**
    * Replace `YOUR_SEQUENCER_VALIDATOR_PRIVATE_KEY_HERE` with your actual private key.
    * Replace `YOUR_ETH_COINBASE_ADDRESS_HERE` with your Ethereum coinbase address.
    * Replace `YOUR_SERVER_PUBLIC_IP_HERE` with your server's public IP.

    ```ini
    [Unit]
    Description=Aztec Sequencer Node
    After=network-online.target docker.service geth-sepolia.service lighthouse-sepolia.service
    Wants=network-online.target docker.service geth-sepolia.service lighthouse-sepolia.service

    [Service]
    User=root # Or dedicated user with access to ~/.aztec
    Group=root
    Restart=always
    RestartSec=5
    WorkingDirectory=/root/ # Or $HOME of the user running aztec
    # Ensure ExecStart path is correct, typically $HOME/.aztec/bin/aztec
    ExecStart=/root/.aztec/bin/aztec start --node --archiver --sequencer --network alpha-testnet \
      --l1-rpc-urls=[http://host.docker.internal:8545](http://host.docker.internal:8545) \
      --l1-consensus-host-urls=[http://host.docker.internal:5052](http://host.docker.internal:5052) \
      --sequencer.validatorPrivateKey=YOUR_SEQUENCER_VALIDATOR_PRIVATE_KEY_HERE \
      --sequencer.coinbase=YOUR_ETH_COINBASE_ADDRESS_HERE \
      --p2p.p2pIp=YOUR_SERVER_PUBLIC_IP_HERE \
      --p2p.maxTxPoolSize=1000000000
      # Add other necessary flags as per Aztec documentation
    LimitNOFILE=65535

    [Install]
    WantedBy=multi-user.target
    ```

6.  **Reload Systemd, Enable and Start Aztec Sequencer:**

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable aztec-sequencer
    sudo systemctl start aztec-sequencer
    sudo systemctl status aztec-sequencer
    journalctl -u aztec-sequencer -f --no-pager # Follow logs
    ```

---

This guide provides steps for setting up your server. Always replace placeholder values and consult official documentation for Geth, Lighthouse, and Aztec for the latest configurations.
