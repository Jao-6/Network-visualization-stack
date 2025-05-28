# RBPI Network-visualization-stack
## Goal
We are trying to achieve simple network visualization, with Raspberry Pi and the Mikrotik's Traffic-Flow system. These instructions provide a step by step guide in how to download and configure the stack and Mikrotik router. So you end with a working network visualization stack on Raspberry Pi accessible from your local network.

!Follow these instructions precisely and you can just copy and paste!

## Steps
[ ] - 1. Prepare RBPI

[ ] - 2. Configure router

[ ] - 3. Get the ELK stack working

[ ] - 4. Get nfacctd working

[ ] - 5. Get Kibana to display desired data


## Step 1 - Prepare Raspberry PI

  1. Create user named rbpi
  2. Connect to ethernet or WiFi
  3. Update Raspberry PI

    sudo apt update
  4. Install Java version 11

    sudo apt install openjdk-11-jdk -y
  5. Install UFW

    sudo apt-get install ufw
  6. Open ports

    sudo ufw allow 5601
    sudo ufw allow ssh
    sudo ufw allow 9200
    sudo ufw allow 2055


## Step 2 - Configure router

  1. Login to Webfig at 192.168.88.1
  2. Move to menu IP > Traffic Flow

    Enabled: [x]
    Interfaces: all
    Cache Entries: 32k
    Active Flow Timeout: 00:30:00
    Inactive Flow Timeout: 00:00:15
    Packet Sampling: [ ]
  
  In section IPFIX select everything.
  3. Press button **Targets**, and button **Add New**

    Enabled: [x]
    Src. Address: 192.168.88.1 (IP of Mikrotik router)
    Dst. Address 192.168.88.251 (IP of Raspberry PI)
    Port: 2055
    Version: 9,
    v9/IPFIX Template Refresh: 20
    v9/IPFIX Template Timeout 1800
  Press Apply
    4. Move to menu IP > DHCP server


## Step 3 - Get the ELK stack working
We got the ELK stack workig by using version 7.10.2 that are deffinetly compatible troughout the stack.
### + ELASTICSEARCH



## Step 4 - Get nfacctd working


## Step 5 - Get Kibana to display desired data
