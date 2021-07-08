![img](https://images.unsplash.com/photo-1417144800752-07cac61d4815?crop=entropy&cs=tinysrgb&fit=max&fm=jpg)

_Photo by Louis Moncouyoux from Unsplash.com_

# Visualizing Telegraf metrics using QuestDB

Telegraf is a plugin-driven server agent for collecting, processing, aggregating, and writing metrics. With more than 200 [plugins](https://docs.influxdata.com/telegraf/v1.19/plugins/) it can collect almost any kind of data about the server it is running on, applications or even files.

Although Telegraf is capable to collect outstanding amount and variety of data,  we need to store visualize this data at some point. Considering that the metrics are collected at a frequency, the incoming data is ordered in  time. The most convinient way to store time series data is using a time  series database, like [QuestDB](https://questdb.io/).

## The problem of out-of-order inserts

The only drawback of using time series databases, is out-of-order writes.  In case the data is coming from multiple sources at the same time, the  database have to sort the data before persisting it. Fortunately, with  the [release of QuestDB 6.0](https://questdb.io/blog/2021/04/20/questdb-release-6-0-alpha), this problem is solved for us and there is no need to apply workaround, just as we would need to do in case of other databases.

In this tutorial, we will set up multiple virtual machines, install  Telegraf, QuestDB and experiment how can we visualize the incoming data  about server status (load, CPU, swap, and memory usage) over time.

## Prerequisites

Celebrating the recent public market debut of DigitalOcean and QuestDB's  marketplace offering, we are going to join the celebration. Therefore,  we will need the following resources for the tutorial:

- A DigitalOcean account (get $100 credit for free by signing up using [this link](https://m.do.co/c/50d6b551562b))
- Basic `shell` knowledge
- Basic knowledge of `vi`/`vim`/`nano` or any terminal-based text editors

Enough talking, let's jump right in the middle and create our Droplets!

## Setting up the infrastructure

We will create two Droplets running the Telegraf agent and one running QuestDB.

### Creating the database Droplet

Let's get started with the database Droplet. DigitalOcean has an awesome  marketplace offering preinstalled, so called, so called 1-Click Apps  reviewed by its staff. QuestDB is available on the marketplace,  therefore its setup is less than 30 seconds:

1. Navigate to the [marketplace listing](https://marketplace.digitalocean.com/apps/questdb?refcode=50d6b551562b)
2. Click on "Create QuestDB Droplet"
3. Select the basic plan for your Droplet and the desired resources having at least 4GB RAM to avoid slow queries
4. Choose the region of your choice that is the closest to you
5. At the "Authentication" section, select your SSH key or set a password for the Droplet's root account
6. Set the hostname to `telegraf-questdb-tutorial`
7. Leave all other settings with their defaults, and click "Create Droplet" at the bottom of the page

![img](https://gaboros.hu/content/images/2021/06/Screenshot-2021-06-21-at-15.34.36.png)

In about 30  seconds, QuestDB is ready to use. To validate the setup was successful,  copy the Droplet's IP address by clicking on it and navigate to `http://<IP ADDRESS>:9000/` where `<IP ADDRESS>` is your IP address you just copied.

The interactive console loads, and we can start querying the database!

### Creating the Telegraf Droplets

At this point, we don't have any data in the database to query.

> QuestDB exposes a reader for InfluxDB line protocol which allows using QuestDB  as a drop-in replacement for InfluxDB and other systems which implement  this protocol. – [questdb.io](https://questdb.io/docs/guides/influxdb-line-protocol/)

We will utilize this functionality to directly send data from Telegraf to  QuestDB. The next step is to create Droplets running the Telegraf agent  and connected to QuestDB. Create the Droplets following the steps below:

1. Navigate to the [Droplets dashboard](https://cloud.digitalocean.com/droplets)
2. In the top-right section of the page, click "Create" and select "Droplets"
3. At the "Choose an image" section, select "Ubuntu" (20.04 LTS x64 at the time of writing)
4. Select the basic plan for your Droplet and the minimum available resources
5. Choose the region of your choice that is the closest to you
6. At the "Authentication" section, select your SSH key or set a password for the Droplet's root account
7. Set the number of Droplets to 2
8. Set the hostname to `telegraf-agent-1` and `telegraf-agent-2`
9. Leave all other settings with their defaults, and click "Create Droplet" at the bottom of the page

Now, compared to the previous Droplet creation, instead of one Droplet,  DigitalOcean will create two for us. In some seconds, the Droplets are  ready for installing the Telegraf agent.![img](https://gaboros.hu/content/images/2021/07/Screenshot-2021-07-08-at-12.54.40.png)         

## Installing and configuring the Telegraf agent

In this section, we will install Telegraf on all the three Droplets we  created. To install Telegraf, we will follow the official installation  method.

### Install and configure the agent on the QuestDB Droplet

First of all, login to `telegraf-questdb-tutorial` Droplet, by executing `ssh root@<IP ADDRESS>` where `<IP ADDRESS>` is the Droplet's IP address. Then, on the server, run the following to make Telegraf client available for installation.

```shell
# Download the signing keys from influxdata.com
curl -s https://repos.influxdata.com/influxdb.key | apt-key add -

# Source release information
source /etc/lsb-release

# Add influxdata.com APT repository to the APT repository list
echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | tee /etc/apt/sources.list.d/influxdb.list

# Fetch available repositories and read package lists
apt-get update
```

Running the above added the necessary APT repository to our list of available APT  repositories. Now, we can simply install the agent as we would do with  any packages by executing `apt-get install -y telegraf`.

The agent is installed, but not configured, yet. To configure it, let's create a new configuration file at `/etc/telegraf/telegraf.d/questdb.conf` with the content below.

```toml
# Configuration for Telegraf agent
[agent]
  ## Default data collection interval for all inputs
  interval = "5s"

# Write results to QuestDB
[[outputs.socket_writer]]
  # Write metrics to a local QuestDB instance over TCP
  address = "tcp://127.0.0.1:9009"

# Read metrics about CPU usage
[[inputs.cpu]]

# Read metrics about memory usage
[[inputs.mem]]

# Read system statistics, like load on the server
[[inputs.system]]
```

After saving the configuration file, we have one thing left: restarting the Telegraf by running `systemctl restart telegraf`. In 5 seconds, the agent will start reporting to QuestDB.

### Installing and configuring Telegraf on other Droplets

The only thing left is installing and configuring the agent on the  remaining Droplets. As you may expect, we have to do the exact same  process that we did for the QuestDB droplet. SSH into both Droplets, `telegraf-agent-1` and `telegraf-agent-2`.

Add the necessary signing keys and prepare local APT repository list:

```shell
# Download the signing keys from influxdata.com
curl -s https://repos.influxdata.com/influxdb.key | apt-key add -

# Source release information
source /etc/lsb-release

# Add influxdata.com APT repository to the APT repository list
echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | tee /etc/apt/sources.list.d/influxdb.list

# Fetch available repositories and read package lists
apt-get update
```

Install Telegraf by executing `apt-get install -y telegraf` and edit the configuration file `/etc/telegraf/telegraf.d/reporter.conf` as the following:

Note that below, the `socket_writer` address in the configuration is set to `<QUESTDB IP ADDRESS>`. Replace that with the IP address of the QuestDB Droplet. Without  changing the IP address, the agent won't be able to report to QuestDB.

```toml
# Configuration for Telegraf agent
[agent]
  ## Default data collection interval for all inputs
  interval = "5s"

# Write results to QuestDB
[[outputs.socket_writer]]
  # Write metrics to a local QuestDB instance over TCP
  address = "tcp://<QUESTDB IP ADDRESS>:9009"

# Read metrics about CPU usage
[[inputs.cpu]]

# Read metrics about memory usage
[[inputs.mem]]

# Read system statistics, like load on the server
[[inputs.system]]
```

The last bit is restarting the Telegraf agent using `systemctl restart telegraf`. Just as in the case with QuestDB Droplet, in some seconds, the agents will start reporting to QuestDB.

### Check incoming reports

At this point we have every component set up and running to visualize some incoming data. Navigate to `http://<QUESTDB IP ADDRESS>:9000` where the `<QUESTDB IP ADDRESS>` is the IP address of your QuestDB droplet, and write the following SQL statement in the SQL editor:

```SQL
SELECT 
    cpu.host,
    avg(mem.used_percent) mem_usage_average,
    avg(cpu.usage_system) cpu_average,
    avg(system.load1) load1_average,
    cpu.timestamp as timestamp 
FROM cpu
INNER JOIN mem ON mem.host = cpu.host
INNER JOIN system ON system.host = cpu.host
SAMPLE BY 5m
ORDER BY timestamp DESC
```

We could also generate a graph from memory usage using the following SQL:

```sql
SELECT 
    host,
    avg(mem.used_percent) usage_average,
    timestamp 
FROM mem
WHERE host = 'telegraf-agent-1'
SAMPLE BY 30s
ORDER BY timestamp DESC
```

## Summary

We've installed QuestDB on DigitalOcean using the new Marketplace offering  for QuestDB, set up multiple Droplets to report to QuestDB through  Telegraf, and visualized some metrics on the interactive console.

Thank you for your attention!
