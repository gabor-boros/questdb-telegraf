![img](https://github.com/gabor-boros/questdb-telegraf/blob/657baca8b96c08ceb76d58e4c2a0faf9afc21dbd/images/banner.png)

# Using Telegraf and QuestDB to store metrics in a time series database

Telegraf is a plugin-driven server agent for collecting, processing, aggregating, and writing metrics. With [more than 200 plugins](https://docs.influxdata.com/telegraf/v1.19/plugins/), it can collect almost any kind of data about the server it is running on, application data or even filesystem changes.

Although Telegraf can collect an exceptional amount and variety of data,  we need to store and visualize this information at some point. Considering that we collect the metrics over time, a convenient way to store time series data is using a time series database. We'll use [QuestDB](https://questdb.io/) for ingestion and perform some basic visualization for this tutorial.

## Multiple telegraf clients and out-of-order data

When you use multiple clients, it can happen that data coming from various sources simultaneously can arrive out-of-order by time. QuestDB used to have the downside of dropping this kind of out-of-order data. The QuestDB team solved this as of [the 6.0 release](https://questdb.io/blog/2021/04/20/questdb-release-6-0-alpha), meaning there is no need to apply any workarounds like sorting data ourselves before inserting.

This tutorial will set up multiple virtual machines, install Telegraf, QuestDB and experiment with how we can visualize the incoming data about server status (load, CPU, swap, and memory usage) over time.

## Prerequisites

Celebrating the recent public market debut of DigitalOcean and QuestDB's marketplace offering, we are going to join the celebration. Therefore, we will need the following resources for the tutorial:

- A DigitalOcean account (get $100 credit for free by signing up using [this link](https://m.do.co/c/50d6b551562b))
- Basic `shell` knowledge
- Basic knowledge of `vi`/`vim`/`nano` or any terminal-based text editors

Enough talking, let's jump right in and create our Droplets!

## Setting up DigitalOcean droplets

The resources we will create are:

- 1 x QuestDB Droplet for storing and visualizing metrics.
- 2 x Droplets running the Telegraf agent collecting system metrics.


### Create a QuestDB Droplet

Let's get started with the database Droplet. DigitalOcean has an excellent marketplace offering preinstalled, so-called, 1-Click Apps reviewed by its staff. QuestDB is available on the marketplace; therefore its setup is less than 30 seconds:

1. Navigate to the [marketplace listing](https://marketplace.digitalocean.com/apps/questdb?refcode=50d6b551562b)
2. Click on "Create QuestDB Droplet"
3. Select the basic plan for your Droplet and the desired resources (use at least 4GB RAM to avoid slow queries)
4. Choose the region of your choice that is the closest to you
5. At the "Authentication" section, select your SSH key or set a password for the Droplet's root account
6. Set the hostname to `telegraf-questdb-tutorial`
7. Leave all other settings with their defaults, and click "Create Droplet" at the bottom of the page

![img](https://github.com/gabor-boros/questdb-telegraf/blob/657baca8b96c08ceb76d58e4c2a0faf9afc21dbd/images/do-droplet-plan.png)

In about 30  seconds, QuestDB is ready to use. To validate that we set everything up successfully, copy the Droplet's IP address by clicking on it and navigate to `http://<IP ADDRESS>:9000/` where `<IP ADDRESS>` is the IP address you just copied. The interactive console should load and we can start querying the database and inserting data!

### Create DigitalOcean Droplets with Telegraf agent

We don't have any data in the database to query yet, so the next steps are to send some metrics to QuestDB for inspection. Scripts that create dummy data are always good to get started, but in this case, let's use some actual data collected on demo machines, so we have proper metrics to play with instead of synthetic data.

> QuestDB exposes a reader for InfluxDB line protocol which allows using QuestDB as a drop-in replacement for InfluxDB and other systems which implement this protocol. – [questdb.io](https://questdb.io/docs/guides/influxdb-line-protocol/)

We will utilize InfluxDB line protocol to send data via Telegraf to QuestDB directly. The next step is to create some Droplets, start Telegraf agents, and point them to QuestDB. Create the Droplets following these steps:

1. Navigate to the [Droplets dashboard](https://cloud.digitalocean.com/droplets)
2. In the top-right section of the page, click "Create" and select "Droplets"
3. At the "Choose an image" section, select `Ubuntu` (`20.04 LTS x64` at the time of writing)
4. Select the **basic** plan for your Droplet and the minimum resource type
5. Choose a region of your choice that is the closest to you
6. At the "Authentication" section, select your SSH key or set a password for the Droplet's root account
7. Set the number of Droplets to 2
8. Set the hostname to `telegraf-agent-1` and `telegraf-agent-2`
9. Leave all other settings with their defaults, and click "Create Droplet" at the bottom of the page

Compared to the previous Droplet creation, DigitalOcean will create two Droplets instead of one. In a few seconds, the Droplets are ready to start up the Telegraf agent.

![img](https://github.com/gabor-boros/questdb-telegraf/blob/657baca8b96c08ceb76d58e4c2a0faf9afc21dbd/images/do-droplet-list.png)         

## Configuring Telegraf to send metrics to QuestDB

In this section, we will install the Telegraf agent on **all three Droplets**. To install Telegraf, we will follow the official installation method.

### Installing Telegraf on the QuestDB Droplet

First of all, login to `telegraf-questdb-tutorial` Droplet by executing `ssh root@<IP ADDRESS>` where `<IP ADDRESS>` is the Droplet's IP address. Then, on the server, run the following to make the Telegraf client available for installation.

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

We are running the above commands to add the APT repository to our list of available repositories. Now, we can install the agent as we would do with any packages by executing `apt-get install -y telegraf`.

The agent is installed but not configured yet. To configure it, let's create a new configuration file at `/etc/telegraf/telegraf.d/questdb.conf` with the following content:

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

After saving the configuration file, we have one thing left to do: restart Telegraf by running `systemctl restart telegraf`. In 5 seconds, the agent will start reporting to QuestDB.

### Installing Telegraf on demo Droplets

Lastly, install Telegraf on the remaining Droplets. As you may expect, we have to perform the same process as the QuestDB droplet. SSH into both Droplets, `telegraf-agent-1` and `telegraf-agent-2`.

Add the necessary signing keys and prepare the local APT repository list:

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

Note that below, we set the `socket_writer` address in the configuration to `<QUESTDB IP ADDRESS>`, which is the IP address of the QuestDB Droplet. 

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

Restart the Telegraf agents with `systemctl restart telegraf` just like the QuestDB Droplet; in a few seconds, the agents will start reporting to our database.

## Run SQL queries on Telegraf metrics

At this point, we have every component set up and running to visualize some incoming data. Navigate to `http://<QUESTDB IP ADDRESS>:9000` where the `<QUESTDB IP ADDRESS>` is the IP address of your QuestDB droplet, and write the following SQL statement in the SQL editor:

```SQL
SELECT * FROM cpu
```

This will return all data in the table for CPU metrics sent by Telegraf. We can easily create some aggregates like the average CPU usage per machine:

```SQL
SELECT 
    host,
    avg(usage_system) cpu_average,
    timestamp
FROM cpu
```

If we want to perform some more complex queries, we can perform JOINs across the three tables:

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

And to visualize this data, query memory usage using the following SQL:

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

The basic in-built charting functionality that QuestDB has can be used like so:

1. Click the **Chart** tab
2. Set **Chart type** to `line`
3. Set **Labels** to `timestamp`
4. Plot `usage_average` as a **Series** and click **Draw**

## Summary

We've installed QuestDB on DigitalOcean using the new Marketplace offering for QuestDB, set up multiple Droplets to report actual metrics to QuestDB via Telegraf, and visualized these metrics on the interactive console. This tutorial shows how easy it is to send real system metrics as time series data to a database like QuestDB for reports and visualization. 

For next steps, we can experiment with some of the additional integrations that Telegraf supports to grab insights from other applications and [use the Grafana integration that QuestDB offers](https://questdb.io/tutorial/2020/10/19/grafana/) to set up more detailed dashboards for visualization or even alerting and notifications.

Thank you for your attention!

