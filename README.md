# RBPI Network-visualization-stack
## Goal
We are trying to achieve simple network visualization, with Raspberry Pi and the Mikrotik's Traffic-Flow system. These instructions provide a step by step guide in how to download and configure the stack and Mikrotik router. So you end with a working network visualization stack on Raspberry Pi accessible from your local network.

!Follow these instructions precisely and you can just copy and paste!

## Steps
* [x] - 1. Prepare RBPI

* [x] - 2. Configure router

* [x] - 3. Get nfacctd working

* [x] - 4. Get the ELK stack working

* [x] - 5. Run everything on startup

* [x] - 6. Get Kibana to display desired data


## Step 1 - Prepare Raspberry PI

  1. Complete guided starting proces

  - Create user with username: rbpi
  - Connect to ethernet or WiFi of the Mikrotik router

  2. Enable SSH for easier configuration

    sudo raspi-config
  
  -  Go to "Interface options" > "SSH" and select **Yes**.

  - If you couldn't connect to WiFi on starting proces, do it here in "System Options" > "Wireless LAN"

  - Select **FINISH**.

  - Now you can SSH to the Raspberry PI from your computer if they are connected to the same network.

   3. Update Raspberry PI

    sudo apt update

```bash
sudo apt upgrade -y
```
  4. Install Java version 11

    sudo apt update && sudo apt install -y wget gnupg && wget -qO - https://repos.azul.com/azul-repo.key | sudo apt-key add - && echo "deb http://repos.azul.com/zulu/deb stable main" | sudo tee /etc/apt/sources.list.d/zulu.list && sudo apt update && sudo apt install -y zulu11-jdk

  5. Install UFW

    sudo apt-get install ufw -y
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
  
  Press **Apply**
    
  4. Move to menu IP > DHCP server

  Select tab **Leases**.
  In table select IP address of Raspbbery Pi and in the new oppened window on the right side press button **Make static**.

## Step 3 - Install and configure NFACCTD

  1. Install dependencies

    sudo apt update && sudo apt install -y git curl wget build-essential libpcap-dev libtool libtool-bin autoconf automake pkg-config

  2. Download

    git clone https://github.com/pmacct/pmacct.git

  3. Move to folder

    cd pmacct
  4. Enable features

    ./configure --enable-l2 --enable-traffic-bins --enable-bgp-bins --enable-bmp-bins --enable-st-bins

  5. Build

    make -j$(nproc)

    sudo make install
  6. Make folder

    sudo mkdir -p /etc/pmacct
  7. Create configuration file and enter the config

    sudo nano /etc/pmacct/nfacctd.conf

Config:

    plugins: print
    daemonize: true
    pidfile: /home/rbpi/nfacctd.pid
    
    nfacctd_port: 2055
    nfacctd_ip: 0.0.0.0
    aggregate: src_host, dst_host, src_port, dst_port, proto
    
    print_output_file: /home/rbpi/flows.log
    print_output: formatted
    print_output_file_append: true
    print_refresh_time: 1

## Step 4 - Get the ELK stack working
We got the ELK stack workig by using version 7.10.2 that are deffinetly compatible troughout the stack.
###  - ELASTICSEARCH
  0. Move to home folder

    cd --
  1. Download

    wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.10.2-linux-aarch64.tar.gz
    
  2. Extract

    tar -xzf elasticsearch-7.10.2-linux-aarch64.tar.gz

  3. Move to folder

    cd elasticsearch-7.10.2/

  4. Create configuration file and enter the config

    nano config/elasticsearch.yml

Config:

    network.host: 0.0.0.0
    discovery.type: single-node

###  - LOGSTASH
  0. Move to home folder

    cd --
  1. Download

    wget https://artifacts.elastic.co/downloads/logstash/logstash-7.10.2-linux-aarch64.tar.gz

  2. Extract

    tar -xzf logstash-7.10.2-linux-aarch64.tar.gz

  3. Move to folder

    cd logstash-7.10.2

  4. Create configuration file and enter the config

    nano config/nfacctd.conf
    
Config:

    input {
      file {
    	path => "/home/rbpi/flows.log"
    	start_position => "beginning"
    	sincedb_path => "/dev/null"
    	stat_interval => 1
    	discover_interval => 1
      }
    }
    
    filter {
      grok {
    	match => {
      	"message" => [
        	"%{IPV4:src_ip}\s+%{IPV4:dst_ip}\s+%{INT:src_port}\s+%{INT:dst_port}\s+%{WORD:protocol}\s+%{INT:packets}\s+%{INT:bytes}",
        	"%{IPV6:src_ip}\s+%{IPV6:dst_ip}\s+%{INT:src_port}\s+%{INT:dst_port}\s+%{NOTSPACE:protocol}\s+%{INT:packets}\s+%{INT:bytes}"
      	]
    	}
      }
    
      mutate {
    	convert => {
      	"src_port" => "integer"
      	"dst_port" => "integer"
      	"packets"  => "integer"
      	"bytes"	=> "integer"
    	}
      }
    }
    
    
    output {
      elasticsearch {
    	hosts => ["http://localhost:9200"]
    	index => "nfacctd-flows"
      }
      stdout { codec => rubydebug }
    }

###  - KIBANA
  0. Move to home folder

    cd --
  1. Download

    wget https://artifacts.elastic.co/downloads/kibana/kibana-7.10.2-linux-aarch64.tar.gz

  2. Extract

    tar -xzf kibana-7.10.2-linux-aarch64.tar.gz

  3. Move to folder

    cd kibana-7.10.2-linux-aarch64

  4. Create configuration file and enter the config

    nano config/kibana.yml
    
Config:

    server.host: "0.0.0.0"
    elasticsearch.hosts: ["http://192.168.88.251:9200"]
    elasticsearch.username: "admin"
    elasticsearch.password: "admin"

###  - Permissions
  0. Move to home folder

    cd --
  1. Give rights to user

    sudo chown -R rbpi:rbpi /home/rbpi/elasticsearch-7.10.2 /home/rbpi/kibana-7.10.2-linux-aarch64 /home/rbpi/logstash-7.10.2

## Step 5 - Run everything on startup
###  - NFACCTD
Create configuration file and enter the config

    sudo nano /etc/systemd/system/nfacctd.service
Config:

    [Unit]
    Description=pmacct NetFlow Collector (nfacctd)
    After=network.target
    
    [Service]
    Type=forking
    ExecStart=/usr/local/sbin/nfacctd -f /etc/pmacct/nfacctd.conf
    PIDFile=/home/rbpi/nfacctd.pid
    Restart=on-failure
    User=rbpi
    WorkingDirectory=/home/rbpi
    StandardOutput=journal
    StandardError=journal
    
    [Install]
    WantedBy=multi-user.target


###  - ELASTICSEARCH
Create configuration file and enter the config

    sudo nano /etc/systemd/system/elasticsearch.service
Config:

    [Unit]
    Description=Elasticsearch
    After=network.target
    
    [Service]
    Type=simple
    User=rbpi
    ExecStart=/home/rbpi/elasticsearch-7.10.2/bin/elasticsearch
    Restart=always
    LimitNOFILE=65535
    
    [Install]
    WantedBy=multi-user.target

###  - LOGSTASH
Create configuration file and enter the config

    sudo nano /etc/systemd/system/logstash.service
Config:

    [Unit]
    Description=Logstash
    After=elasticsearch.service
    
    [Service]
    Type=simple
    User=rbpi
    ExecStart=/home/rbpi/logstash-7.10.2/bin/logstash -f /home/rbpi/logstash-7.10.2/config/nfacctd.conf
    Restart=always
    
    [Install]
    WantedBy=multi-user.target

###  - KIBANA
Create configuration file and enter the config

    sudo nano /etc/systemd/system/kibana.service
Config:

    [Unit]
    Description=Kibana
    After=logstash.service
    
    [Service]
    Type=simple
    User=rbpi
    ExecStart=/home/rbpi/kibana-7.10.2-linux-aarch64/bin/kibana
    Restart=always
    
    [Install]
    WantedBy=multi-user.target

###  - ENABLE SERVICES

    sudo systemctl daemon-reload
    sudo systemctl enable elasticsearch
    sudo systemctl enable logstash
    sudo systemctl enable kibana
    sudo systemctl enable nfacctd
    
    sudo systemctl start elasticsearch
    sudo systemctl start logstash
    sudo systemctl start kibana
    sudo systemctl start nfacctd

## Step 6 - Get Kibana to display desired data

### 1. Access Grafana

   - Open web browser and go to:

          http://192.168.88.251:5601/

### 2. Add Elasticsearch as a data source

  1. Go to menu **Configuration** > **Data Sources**
  2. Click **Add data source**
  3. Select **Elasticsearch**
  4. Configure:

            URL: http://localhost:9200
            Index name: nfacctd-flows
            Time field name: @timestamp

  5. Click **Save & Test**

### 3. Create Visualizations

#### - Outgoing Ports (Pie Chart)

  1. Go to **Dashboard** > **Add new panel**
  2. Select **Visualization**: Pie Chart
  3. Under **Query**:

                Aggregation: Count
                Group by: Terms
                Field: src_port
                Size: 10

  4. Click **Apply** and **Save**

#### - Incoming Ports (Pie Chart)

  1. Go to **Dashboard** > **Add new panel**
  2. Select **Visualization**: Pie Chart
  3. Under **Query**:

                Aggregation: Count
                Group by: Terms
                Field: dst_port
                Size: 10

  4. Click **Apply** and **Save**

#### - Sum of Packets per Source IP (Bar Chart)

  1. Go to **Dashboard** > **Add new panel**
  2. Select **Visualization**: Bar Chart
  3. Under **Query**:

                Aggregation: Sum
                Field: packets
                Group by: Terms
                Field: src_ip
                Size: 10

  4. Click **Apply** and **Save**

#### - Packets Over Time (Time Series)

  1. Go to **Dashboard** > **Add new panel**
  2. Select **Visualization**: Time Series
  3. Under **Query**:

                Aggregation: Sum
                Field: packets
                Group by: Date Histogram
                Field: @timestamp
                Interval: auto or 5s

  4. Click **Apply** and **Save**

### 4. Configure auto-refresh

  1. In the dashboard top bar, click **Refresh**
  2. Select **5s** for auto-refresh

### 5. Save the dashboard

  1. Click the **Save** icon
  2. Name the dashboard **Network Visualization**
  3. Select save with time
  4. Click **Save**

