NetLytics
=========
NetLytics is a Hadoop-powered framework for performing advanced analytics on various kinds of networks logs.

It is able to parse log files generated by popular network softwares implementing HTTP proxy, DNS server and passive sniffers; 
it can also parse raw PCAP files.

It assumes that log files are stored on HDFS in a Hadoop based cluster.

NetLytics uses log files to perform a wide range of advanced network analytics for traffic monitoring and security purposes. All code is written in Python and uses Apache Spark.

For information about this Readme file and this tool please write to
[martino.trevisan@polito.it](mailto:martino.trevisan@polito.it)

# Table of Content
<!---
Done with https://github.com/ekalinin/github-markdown-toc )
-->

   * [NetLytics](#netlytics)
   * [Table of Content](#table-of-content)
   * [1. Prerequisites](#1-prerequisites)
   * [2. Architecture](#2-architecture)
      * [2.1 Tools generating Network Logs](#21-tools-generating-network-logs)
         * [Bro](#bro)
         * [Squid](#squid)
         * [Tstat](#tstat)
         * [Bind](#bind)
         * [Pcap Files](#pcap-files)
      * [2.2 Data storage](#22-data-storage)
      * [2.3 Connectors](#23-connectors)
      * [2.4 Data Tables](#24-data-tables)
      * [2.5 Algorithms](#25-algorithms)
      * [2.6 Running SQL queries](#26-running-sql-queries)
   * [3. Running jobs:](#3-running-jobs)
   * [4. Running SQL queries:](#4-running-sql-queries)
   * [5. Examples](#5-examples)
      * [5.1 Running an algorithm](#51-running-an-algorithm)


# 1. Prerequisites
Netlytics is designed to work in the Hadoop ecosystem. As such, it needs HDFS and Apache Spark (>= 2.0.0) to respectively store and process log files. It uses python 2.7 and few python packages: you can install them with pip
```
sudo pip install zipfile scapy numpy pandas pyasn
```
If you are using a cluster, these packages must be installed on each worker node.

You can clone the GitHub repository with the command:
```
git clone https://github.com/marty90/netlytics
```

# 2. Architecture
NetLytics is composed on many building blocks illustrated in the figure below.
![alt text](https://github.com/marty90/netlytics/raw/master/images/netlytics_arch.png)

## 2.1 Tools generating Network Logs
NetLytics handles log files generated by a wide number of softwares.
From raw log files, NetLytics can create Data Tables for **DNS**, **HTTP** and **Named Flows**.
Not all tables can be created from all tools.

|**Tool** | **DNS**  |  **HTTP**   |   **Named Flows** |
|-------|-----|------|-------------|
|  **Bro**   | **Y**   | **Y**    | **Y**          |
|  **Squid** | N   | **Y**    | **Y**           |
|  **Tstat**  | **Y**   | **Y**    | **Y**           |
| **Bind** | **Y**   | N    | N           |
|  **PCAP** | **Y**   | N    | N           |

It not differently specified, NetLytics assumes that the folder structure and the file names are the default ones.
Currently Netlytics can parse log files generated by 5 tools.
### Bro
[Bro](https://www.bro.org/) is a network security monitor that passively analyzes traffic and produces several log files.
In particular, log files for TCP, HTTP and DNS traffic are generated.
Bro typically rotate log files every hour, and puts all log files generated in one day in a separate directory. The typical directory tree is the following:
```
2017-11-20/
\------------ dns.22.00.00-23.00.00.log.gz
\------------ dns.23.00.00-00.00.00.log.gz
\------------ http.13.00.00-14.00.00.log.gz
\------------ http.14.00.00-15.00.00.log.gz
\------------ ...
2017-11-21/
\------------ ...
```
NetLyitics assumes that this directory structure is replicated in HDFS.

NetLytics Data Tables: **DNS**, **HTTP** and **Named Flows**.

### Squid
[Squid](http://www.squid-cache.org/) is the most popular software HTTP proxy. It generates a log file where all HTTP transactions are recorded. It is typically stored in `/var/log/squid/access.log`; it can assume various formats, but NetLytics assumes the default one is used (called `squid format` in Squid documentation).

Squid does not handle log file rotation and storage, but users typically employ the [logrotate](https://linux.die.net/man/8/logrotate) utility to handle this.
`logrotate` periodically rotates log files and stores old ones with the name `access.log-YYYYMMDD` where `YYYY`, `MM` and `DD` are the current year, month and day respectively.
NetLytics assumes this name format is used to store log files on HDFS.

NetLytics Data Tables: **HTTP** and **Named Flows**.


### Tstat
[Tstat](http://tstat.polito.it/) is a network meter that passively analyzes traffic and produces rich log files.
In particular, log files for TCP, HTTP and DNS traffic are generated.

Tstat typically rotate log files every hour, and puts all log files generated in one day in a separate directory. The typical directory tree is the following:
```
2016
\------------ 04-Apr
              \------------ 2017_04_01_00_30.out
              \------------ 2017_04_01_01_30.out
              \------------ 2017_04_01_02_30.out
              \------------ ...
\------------ 05-May
              \------------ 2017_05_01_00_30.out
              \------------ 2017_05_01_01_30.out
              \------------ ...
\------------ ...
2017
\------------ 01-Jan
...
```
NetLyitics assumes that this directory structure is replicated in HDFS.

NetLytics Data Tables: **DNS**, **HTTP** and **Named Flows**.


### Bind
[Bind](https://www.isc.org/downloads/bind/) is the most popular software DNS server.
It can be configured to create log files changing the `/etc/bind/named.conf` files, adding:
```
logging {
  channel bind_log {
    file "/var/log/bind/bind.log" versions 3 size 5m;
    severity info;
    print-category yes;
    print-severity yes;
    print-time yes;
  };
  category queries { bind_log; };
};

```
The log file is created in `/var/log/bind/bind.log`.
Bind does not handle log file rotation and storage, but users typically employ the [logrotate](https://linux.die.net/man/8/logrotate) utility to handle this.
`logrotate` periodically rotates log files and stores old ones with the name `bind.log-YYYYMMDD` where `YYYY`, `MM` and `DD` are the current year, month and day respectively.
NetLytics assumes this name format is used to store log files on HDFS.

NetLytics Data Tables: **DNS**.

### Pcap Files
NetLytics can parse raw Pcap files to extract DNS data.
We reccommend to use the utility [dnscap](https://github.com/DNS-OARC/dnscap) to generate such Pcap files.
It can rotate log files after a configurable period of time.
Default Pcap file names have the format: `<base_name>.YYYYMMDD.HHmmSS.uSec`.
NetLytics assumes this format is used in HDFS.

NetLytics Data Tables: **DNS**.

## 2.2 Data storage
NetLytics assumes that network log files are stored on HDFS and accessible by the current Spark user.
Each tool producing

## 2.3 Connectors
The connectors are the software modules to parse log files and create Data Tables.
NetLytics parses log files on the original tool format, and creates on-the-fly Data Tables to be used by algorithms.

Each connector is identified by a *class name*, and is suited for generating a given *Data Table* from log file of a *tool*.

The following table illustrates the available connectors:

| **Tool**|**Data Table**| **Class Name**                                  |
|-------|------------|------------------------------------------------------|
| Tstat | DNS        | `connectors.tstat_to_DNS.Tstat_To_DNS`                 |
| Tstat | NamedFlows | `connectors.tstat_to_named_flows.Tstat_To_Named_Flows` |
| Tstat | HTTP       | `connectors.tstat_to_HTTP.Tstat_To_HTTP`               |
| Bro   | DNS        | `connectors.bro_to_DNS.Bro_To_DNS`                     |
| Bro   | HTTP       | `connectors.bro_to_HTTP.Bro_To_HTTP`                   |
| Bro   | NamedFlows | `connectors.bro_to_named_flows.Bro_To_Named_Flows`     |
| Squid | HTTP       | `connectors.squid_to_HTTP.Squid_To_HTTP`               |
| Squid | NamedFlows | `connectors.squid_to_named_flows.Squid_To_Named_Flows` |
| Bind  | DNS        | `connectors.bind_to_DNS.Bind_To_DNS`                   |
| PCAP  | DNS        | `connectors.PCAP_to_DNS.PCAP_To_DNS`                   |


## 2.4 Data Tables
A Data Table represents a dataset of network measurements under a certain period of time. It is implemented using Spark Dataframes. Three types of Data Tables are handled by NetLytics:

* **DNS**: contains information about DNS traffic, such as queried domains, contacted resolvers, etc. It is generated by passive sniffers and DNS servers.

* **HTTP**: contains information HTTP transactions. It reports queried URLs, contacted servers, etc. It is generated by passive sniffers and HTTP servers. Limited information is available when encryption (SSL, TLS, HTTPS) is used.

* **Named Flows**: contains flow level measurements enriched with hostname of the server being contacted. Beside typical flow level statistics (number of packets and bytes), there is an explicit indication of the hostname of the server. This indication is typically generated using DPI on HTTP/TLS fields or with DNS traffic analysis. These logs are generated by passive sniffers and can be derived from HTTP proxy logs.

## 2.5 Algorithms
Using Data Tables, NetLyitics can run algorithms on data. Several algorithms are available, and users are encouraged to share their own to enrich this software.
Available algorithms are:
* **REMeDy: DNS manipulation detection**: REMeDy is a system that assists operators to identify the use of rogue DNS resolvers in their networks. It is a completely automatic and parameter-free system that evaluates the consistency of responses across the resolvers active in the network. The paper is available [here](https://www.dropbox.com/s/8s9azpvnqsle78t/rogue_DNS_ds4n.pdf?dl=1).
    * Input Data Table: DNS
    * Class Name: `algos.remedy.Remedy_DNS_manipulations.RemedyDNSManipulations`
    * Parameters: 
        * min_resolutions : Minimum number of observation to consider a domain. Default 50.
        * ASN_VIEW : Path to an updated ASN view compatible with `pyasn` python module.
     * Output: A single CSV file reporting general per-resolver statistics, along with discovered manipulations.
* **WHAT: Cloud service meter**: WHAT is a system to uncover the overall traffic produced by specific web services. WHAT combines big data and machine learning approaches to process large volumes of network flow measurements and learn how to group traffic due to pre-defined services of interest. The paper is available [here](http://www.tlc-networks.polito.it/mellia/papers/BMLIT_web_meter.pdf).
    * Input Data Table: Named Flows
    * Class Name: `algos.WHAT.WHAT`
    * Parameters: 
        * CORES : Core Domains to measure.",
        * OW : Observation Windows for training, in milliseconds. Default 10000.
        * GAP : Silent time for considering OW, in milliseconds. Default 10000.
        * EW : Evaluation Window for classification, in milliseconds. Default 5000.
        * N_TOP : Truncate output list after N_TOP entries. Default 50.
     * Output: A single CSV file reporting the amount of traffic due to each core domain.
* **Contacted domains**: account the traffic generated on the network on a per domain fashion.
    * Input Data Table: Named Flows
    * Class Name: `algos.domain_traffic.DomainTraffic`
    * Parameters: 
        * N_TOP :Truncate output list after N_TOP entries. Default 50.
     * Output: A single CSV file reporting the amount of traffic due to each domain.
* **Save Dataset**: Simply save the dataframe in local.
    * Input Data Table: * (all)
    * Class Name: `algos.save_Dataframe.SaveDataFrame`
    * Parameters: 
        * N : Truncate output after N entries. Default: No limit.
     * Output: The dataframe in CSV format.
     
## 2.6 Running SQL queries
NetLyitics allows to run SQL queries on Data Tables to perform simple analytics. Instruction to this are reported later.


# 3. Running jobs:
To run a NetLytics job, you shoud use the `run_job.py` script, which has the following syntax:
```
spark2-submit run_job.py [-h] [--input_path input_path] [--output_path output_path]
                  [--connector connector] [--algo algo] [--params params]
                  [--start_day start_day] [--end_day end_day]
                  [--temp_dir_local temp_dir_local]
                  [--temp_dir_HDFS temp_dir_HDFS]
                  [--persistent_dir_local persistent_dir_local]
                  [--persistent_dir_HDFS persistent_dir_HDFS]

optional arguments:
  -h, --help            show this help message and exit
  --input_path input_path
                        Base Log Files Input Path
  --output_path output_path
                        Directory where the output of the algo is stored
  --connector connector
                        Connector class name
  --algo algo           Algorithm to run
  --params params       Parameters to be given to the Algo, in Json
  --start_day start_day
                        Start day for analysis, format YYYY_MM_DD
  --end_day end_day     End day for analysis, format YYYY_MM_DD
  --temp_dir_local temp_dir_local
                        Directory where to store intermediate files
  --temp_dir_HDFS temp_dir_HDFS
                        Directory on HDFS where to store intermediate files
  --persistent_dir_local persistent_dir_local
                        Directory where to store persistent algorithm data
                        (local)
  --persistent_dir_HDFS persistent_dir_HDFS
                        Directory where to store persistent algorithm data
                        (HDFS)
```
Recall that the script must be submitted to spark, and, thus, executed with the `spark-submit` utility.


# 4. Running SQL queries:
NetLytics can run SQL queries on Data Table, using the `run_query.py` script, which has the following syntax:  
```
spark2-submit run_query.py [-h] [--input_path input_path]
                    [--output_file_local output_file_local]
                    [--output_file_HDFS output_file_HDFS] [--query query]
                    [--connector connector] [--start_day start_day]
                    [--end_day end_day]

optional arguments:
  -h, --help            show this help message and exit
  --input_path input_path
                        Base Log Files Input Path
  --output_file_local output_file_local
                        File where the resulting table is saved (locally).
                        Cannot be specified together with output_file_HDFS
  --output_file_HDFS output_file_HDFS
                        File where the resulting table is saved (HDFS). Cannot
                        be specified together with output_file_local
  --query query         SQL Query to exectute. Use "netlytics" as SQL table
                        name
  --connector connector
                        Connector class name
  --start_day start_day
                        Start day for analysis, format YYYY_MM_DD
  --end_day end_day     End day for analysis, format YYYY_MM_DD

```

To get the list of the available columns, see the `json` schemas in the `schema` directories.
The name of the table to be used in the query is `netlytics`.

# 5. Examples
## 5.1 Running an algorithm
In this example we suppose to have SQUID log files in `logs/squid` for the whole month of March 2017.
Now we run the command to account the account the traffic to the corresponding domain for the whole month.

```
spark-submit run_job.py \
      \
      --connector "connectors.squid_to_named_flows.Squid_To_Named_Flows" \
      --input_path "logs/squid" \
      --start_day "2017_03_01" \
      --end_day "2017_03_31" \
      \
      --algo "algos.domain_traffic.DomainTraffic" \
      --params '{"N":40}' \
      --output_path "results_DOMAIN_TRAFFIC" \
```
## 5.2 Running a SQL query
Now, we perform the same job of above using a SQL query, and the `run_query.py` script.
```
spark-submit run_query.py \
      \
      --connector "connectors.squid_to_named_flows.Squid_To_Named_Flows" \
      --input_path "logs/squid" \
      --start_day "2017_03_01" \
      --end_day "2017_03_31" \
      \
      --query "SELECT name, SUM(s_bytes) AS traffic FROM netlytics GROUP BY name ORDER BY SUM(s_bytes)" \
      --output_file_local "traffic_name.csv"

```


