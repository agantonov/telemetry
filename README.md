## OpenConfig Telemetry

The [OpenConfig project](https://www.openconfig.net/) defines and implements common, vendor-independent software not only for the management of network devices but also provides models for efficient and accurate real-time monitoring of the network based on streaming telemetry. The key component is the [gNMI protocol](https://www.openconfig.net/docs/gnmi/gnmi-specification/) which defines the _Subscribe_ RPC for subscribing to telemetry data. The telemetry collector uses this RPC to request updates from the network device for state and configuration data.

Below, I will provide an example of how we can use open-source tools to build an infrastructure and monitor Juniper routers using OpenConfig data models. To accomplish this task we need three components:
* An agent that interacts with the router and subscribes to telemetry data
* A database to store the telemetry data
* A data visualization tool

I have chosen one of the most common tools which are well-intergrated with each other:
- [Telegraf](https://github.com/influxdata/telegraf) as a gRPC agent
- [InfluxDB](https://github.com/influxdata/influxdb/tree/master) as a database
- [Grafana](https://github.com/grafana/grafana) as a visualization tool

### Installation
The installation of the software on Ubuntu is quite straightforward and well-documented on the website of each project.

1\. [Install Telegraf](https://docs.influxdata.com/telegraf/v1/install/) (Ubuntu)
   ```
   curl -s https://repos.influxdata.com/influxdata-archive.key > influxdata-archive.key
   echo '943666881a1b8d9b849b74caebf02d3465d6beb716510d86a39f6c8e8dac7515 influxdata-archive.key' | sha256sum -c && cat influxdata-archive.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/influxdata-archive.gpg > /dev/null
   echo 'deb [signed-by=/etc/apt/trusted.gpg.d/influxdata-archive.gpg] https://repos.influxdata.com/debian stable main' | sudo tee /etc/apt/sources.list.d/influxdata.list
   sudo apt-get update && sudo apt-get install telegraf
   ```
2\. [Install InfluxDB v2](https://docs.influxdata.com/influxdb/v2/install/) (Ubuntu)
   ```
   sudo apt-get update
   sudo apt-get install influxdb2
   ```
3\. [Install Grafana](https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/) (Ubuntu)
   ```
   sudo apt-get install -y apt-transport-https software-properties-common wget
   sudo mkdir -p /etc/apt/keyrings/
   wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
   echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
   sudo apt-get update
   sudo apt-get install grafana
   ```
   ### Configuration
1\. **InfluxDB**

   Start the InfluxDB service using the following command:
   ```
   sudo systemctl start influxdb
   ```
   Enable InfluxDB as a service
   ```
   sudo systemctl enable influxdb
   ```
   - Set up InfluxDB through the UI
   - With InfluxDB running, visit http://localhost:8086.
   - Click Get Started
   - Set up your initial user
   - Enter a Username for your initial user.
   - Enter a Password and Confirm Password for your user.
   - Enter your initial Organization Name (Juniper).
   - Enter your initial Bucket Name (poc).
   - Click Continue.
   - Copy the provided operator API token and paste to:
     ```
     echo 'INFLUX_TOKEN="<TOKEN>"' >> cat /etc/default/telegraf
     ```
2\. **Juniper router**

   We configure Juniper router to listen to gRPC requests on port 32767 without authentication.
   ```
   set system services extension-service request-response grpc clear-text port 32767
   set system services extension-service request-response grpc skip-authentication
   ```
3\. **Telegraf**
   
   Here is the most interesting part as we need to specify the data we wish to receive. To begin, let's delve into the OpenConfig telemetry data models and proceed by cloning the 
   OpenConfig repository to obtain the corresponding YANG models.
   ```
   $ git clone https://github.com/openconfig/public.git
   ```
   Let's focus on the interface counters in `openconfig-interfaces.yang`
   ```
   $ cd public/release/models/interfaces
   $ pyang -f tree openconfig-interfaces.yang

   module: openconfig-interfaces
     +--rw interfaces
        +--rw interface* [name]
           +--rw name                  -> ../config/name
           +--rw config
           |  +--rw name?            string
           |  +--rw type             identityref
           |  +--rw mtu?             uint16
           |  +--rw loopback-mode?   oc-opt-types:loopback-mode-type
           |  +--rw description?     string
           |  +--rw enabled?         boolean
           +--ro state
           |  +--ro name?            string
           |  +--ro type             identityref
           |  +--ro mtu?             uint16
           |  +--ro loopback-mode?   oc-opt-types:loopback-mode-type
           |  +--ro description?     string
           |  +--ro enabled?         boolean
           |  +--ro ifindex?         uint32
           |  +--ro admin-status     enumeration
           |  +--ro oper-status      enumeration
           |  +--ro last-change?     oc-types:timeticks64
           |  +--ro logical?         boolean
           |  +--ro management?      boolean
           |  +--ro cpu?             boolean
           |  +--ro counters
           |     +--ro in-octets?             oc-yang:counter64
           |     +--ro in-pkts?               oc-yang:counter64
           |     +--ro in-unicast-pkts?       oc-yang:counter64
           |     +--ro in-broadcast-pkts?     oc-yang:counter64
           |     +--ro in-multicast-pkts?     oc-yang:counter64
           |     +--ro in-errors?             oc-yang:counter64
           |     +--ro in-discards?           oc-yang:counter64
           |     +--ro out-octets?            oc-yang:counter64
           |     +--ro out-pkts?              oc-yang:counter64
           |     +--ro out-unicast-pkts?      oc-yang:counter64
           |     +--ro out-broadcast-pkts?    oc-yang:counter64
           |     +--ro out-multicast-pkts?    oc-yang:counter64
           |     +--ro out-discards?          oc-yang:counter64
           |     +--ro out-errors?            oc-yang:counter64
           |     +--ro last-clear?            oc-types:timeticks64
           |     +--ro in-unknown-protos?     oc-yang:counter64
           |     +--ro in-fcs-errors?         oc-yang:counter64
           |     +--ro carrier-transitions?   oc-yang:counter64
           |     +--ro resets?                oc-yang:counter64
           +--rw hold-time
           |  +--rw config
           |  |  +--rw up?     uint32
           |  |  +--rw down?   uint32
           |  +--ro state
           |     +--ro up?     uint32
           |     +--ro down?   uint32
           +--rw penalty-based-aied
           |  +--rw config
           |  |  +--rw max-suppress-time?    uint32
           |  |  +--rw decay-half-life?      uint32
           |  |  +--rw suppress-threshold?   uint32
           |  |  +--rw reuse-threshold?      uint32
           |  |  +--rw flap-penalty?         uint32
           |  +--ro state
           |     +--ro max-suppress-time?    uint32
           |     +--ro decay-half-life?      uint32
           |     +--ro suppress-threshold?   uint32
           |     +--ro reuse-threshold?      uint32
           |     +--ro flap-penalty?         uint32
           +--rw subinterfaces
              +--rw subinterface* [index]
                 +--rw index     -> ../config/index
                 +--rw config
                 |  +--rw index?         uint32
                 |  +--rw description?   string
                 |  +--rw enabled?       boolean
                 +--ro state
                    +--ro index?          uint32
                    +--ro description?    string
                    +--ro enabled?        boolean
                    +--ro name?           string
                    +--ro ifindex?        uint32
                    +--ro admin-status    enumeration
                    +--ro oper-status     enumeration
                    +--ro last-change?    oc-types:timeticks64
                    +--ro logical?        boolean
                    +--ro management?     boolean
                    +--ro cpu?            boolean
                    +--ro counters
                       +--ro in-octets?             oc-yang:counter64
                       +--ro in-pkts?               oc-yang:counter64
                       +--ro in-unicast-pkts?       oc-yang:counter64
                       +--ro in-broadcast-pkts?     oc-yang:counter64
                       +--ro in-multicast-pkts?     oc-yang:counter64
                       +--ro in-errors?             oc-yang:counter64
                       +--ro in-discards?           oc-yang:counter64
                       +--ro out-octets?            oc-yang:counter64
                       +--ro out-pkts?              oc-yang:counter64
                       +--ro out-unicast-pkts?      oc-yang:counter64
                       +--ro out-broadcast-pkts?    oc-yang:counter64
                       +--ro out-multicast-pkts?    oc-yang:counter64
                       +--ro out-discards?          oc-yang:counter64
                       +--ro out-errors?            oc-yang:counter64
                       +--ro last-clear?            oc-types:timeticks64
                       x--ro in-unknown-protos?     oc-yang:counter64
                       x--ro in-fcs-errors?         oc-yang:counter64
                       x--ro carrier-transitions?   oc-yang:counter64
   ```
   We want to subscribe to the following subtree `/interfaces/interface[name=<interface_name>]/state/counters`.
   The configuration for Telegraf will be as follows:
   ```
   $ grep -v '^ *#\|^$' /etc/telegraf/telegraf.conf
   [global_tags]
   [agent]
     interval = "10s"
     round_interval = true
     metric_batch_size = 1000
     metric_buffer_limit = 10000
     collection_jitter = "0s"
     flush_interval = "10s"
     flush_jitter = "0s"
     precision = "0s"
      logtarget = "file"
      logfile = "/var/log/telegraf/telegraf"
      logfile_rotation_max_size = "10MB"
      logfile_rotation_max_archives = 5
     hostname = ""
     omit_hostname = false
    [[outputs.influxdb_v2]]
      urls = ["http://localhost:8086"]
      token = "$INFLUX_TOKEN"
      organization = "Juniper"
      bucket = "poc"
    [[inputs.gnmi]]
      addresses = ["acx7509-2:32767"]
      encoding = "proto"
      redial = "10s"
      max_msg_size = "10MB"
      [[inputs.gnmi.subscription]]
           name = "interface-counters"
           origin = "openconfig-interfaces"
           path="/interfaces/interface[name=et-1/0/0]/state/counters"
           subscription_mode = "sample"
           sample_interval = "60s"
      [[inputs.gnmi.subscription]]
           name = "interface-counters"
           origin = "openconfig-interfaces"
           path="/interfaces/interface[name=et-1/0/1]/state/counters"
           subscription_mode = "sample"
           sample_interval = "60s"
      [[inputs.gnmi.subscription]]
           name = "interface-counters"
           origin = "openconfig-interfaces"
           path="/interfaces/interface[name=et-6/0/0]/state/counters"
           subscription_mode = "sample"
           sample_interval = "60s"
      [[inputs.gnmi.subscription]]
           name = "interface-counters"
           origin = "openconfig-interfaces"
           path="/interfaces/interface[name=et-6/0/1]/state/counters"
           subscription_mode = "sample"
           sample_interval = "60s"
    [[inputs.gnmi]]
      addresses = ["mx204-83:32767"]
      encoding = "proto"
      redial = "10s"
      max_msg_size = "10MB"
      [[inputs.gnmi.subscription]]
           name = "interface-counters"
           origin = "openconfig-interfaces"
           path="/interfaces/interface[name=et-0/0/0]/state/counters"
           subscription_mode = "sample"
           sample_interval = "60s"
      [[inputs.gnmi.subscription]]
           name = "interface-counters"
           origin = "openconfig-interfaces"
           path="/interfaces/interface[name=et-0/0/1]/state/counters"
           subscription_mode = "sample"
           sample_interval = "60s"
   ```
   Optionally, we can configure Telegraf to write metrics to a file: 
   ```
   [[outputs.file]]
   files = ["stdout", "/tmp/metrics.out"]
   ```
   Start Telegraf and configure it to start automatically after a reboot:
   ```
   sudo systemctl start telegraf
   sudo systemctl enable telegraf
   ```
   
   Make sure that Telegraf is subscribed to telemetry data on the routers via gNMI:
   
   ```
   acx7509-2#run show agent sensors
   
   Sensor Information :
   
       Name                                    : sensor_1000
       Resource                                : /interfaces/interface[name='et-1/0/0']/state/counters/
       Version                                 : 1.0
       Sensor-id                               : 562949953421323
       Subscription-ID                         : 1000
       Component(s)                            : re0/evoaft-jvisiond-brcm,re0/mgmt-ethd,re0/mib2d,re1/evoaft-jvisiond-brcm,re1/mgmt-ethd
   
       Sensor Information :
   
       Name                                    : sensor_1005
       Resource                                : /interfaces/interface[name='et-1/0/1']/state/counters/
       Version                                 : 1.0
       Sensor-id                               : 562949953421324
       Subscription-ID                         : 1005
       Component(s)                            : re0/evoaft-jvisiond-brcm,re0/mgmt-ethd,re0/mib2d,re1/evoaft-jvisiond-brcm,re1/mgmt-ethd
   
       Profile Information :
   
           Name                                : export_1005
           Reporting-interval                  : 60
           Payload-size                        : 5000
           Address                             : 0.0.0.0
           Port                                : 1000
           Timestamp                           : ntp
           Format                              : GPB
   
   Sensor Information :
   
       Name                                    : sensor_1006
       Resource                                : /interfaces/interface[name='et-6/0/0']/state/counters/
       Version                                 : 1.0
       Sensor-id                               : 562949953421325
       Subscription-ID                         : 1006
       Component(s)                            : re0/evoaft-jvisiond-brcm,re0/mgmt-ethd,re0/mib2d,re1/evoaft-jvisiond-brcm,re1/mgmt-ethd
   
       Profile Information :
   
           Name                                : export_1006
           Reporting-interval                  : 60
           Payload-size                        : 5000
           Address                             : 0.0.0.0
           Port                                : 1000
           Timestamp                           : ntp
           Format                              : GPB
   
   Sensor Information :
   
       Name                                    : sensor_1007
       Resource                                : /interfaces/interface[name='et-6/0/1']/state/counters/
       Version                                 : 1.0
       Sensor-id                               : 562949953421326
       Subscription-ID                         : 1007
       Component(s)                            : re0/evoaft-jvisiond-brcm,re0/mgmt-ethd,re0/mib2d,re1/evoaft-jvisiond-brcm,re1/mgmt-ethd
   
       Profile Information :
   
           Name                                : export_1007
           Reporting-interval                  : 60
           Payload-size                        : 5000
           Address                             : 0.0.0.0
           Port                                : 1000
           Timestamp                           : ntp
           Format                              : GPB
   
   mx204-83# run show agent sensors
   
   Sensor Information :
   
       Name                                    : sensor_1004
       Resource                                : /interfaces/interface[name='et-0/0/0']/state/counters/
       Version                                 : 1.0
       Sensor-id                               : 539528119
       Subscription-ID                         : 1004
       Parent-Sensor-Name                      : Not applicable
       Component(s)                            : PFE,PFE,mib2d,xmlproxyd
   
       Profile Information :
   
           Name                                : export_1004
           Reporting-interval                  : 60
           Payload-size                        : 5000
           Format                              : GPB
   
   Sensor Information :
   
       Name                                    : sensor_1004_2_1
       Resource                                : /interfaces/interface[name='et-0/0/0']/state/counters/
       Version                                 : 1.0
       Sensor-id                               : 3143450969
       Subscription-ID                         : 1004
       Parent-Sensor-Name                      : sensor_1004
       Component(s)                            : mib2d
   
       Profile Information :
   
           Name                                : export_1004
           Reporting-interval                  : 60
           Payload-size                        : 5000
           Format                              : GPB
   
   Sensor Information :
   
       Name                                    : sensor_1004_3_1
       Resource                                : /interfaces/interface[name='et-0/0/0']/state/counters/
       Version                                 : 1.0
       Sensor-id                               : 3143451993
       Subscription-ID                         : 1004
       Parent-Sensor-Name                      : sensor_1004
       Component(s)                            : xmlproxyd
   
       Profile Information :
   
           Name                                : export_1004
           Reporting-interval                  : 60
           Payload-size                        : 5000
           Format                              : GPB
   
   Sensor Information :
   
       Name                                    : sensor_1005
       Resource                                : /interfaces/interface[name='et-0/0/1']/state/counters/
       Version                                 : 1.0
       Sensor-id                               : 539528118
       Subscription-ID                         : 1005
       Parent-Sensor-Name                      : Not applicable
       Component(s)                            : PFE,PFE,mib2d,xmlproxyd
   
       Profile Information :
   
           Name                                : export_1005
           Reporting-interval                  : 60
           Payload-size                        : 5000
           Format                              : GPB
   Sensor Information :
   
       Name                                    : sensor_1005_2_1
       Resource                                : /interfaces/interface[name='et-0/0/1']/state/counters/
       Version                                 : 1.0
       Sensor-id                               : 3142402393
       Subscription-ID                         : 1005
       Parent-Sensor-Name                      : sensor_1005
       Component(s)                            : mib2d
   
       Profile Information :
   
           Name                                : export_1005
           Reporting-interval                  : 60
           Payload-size                        : 5000
           Format                              : GPB
   
   Sensor Information :
   
       Name                                    : sensor_1005_3_1
       Resource                                : /interfaces/interface[name='et-0/0/1']/state/counters/
       Version                                 : 1.0
       Sensor-id                               : 3142403417
       Subscription-ID                         : 1005
       Parent-Sensor-Name                      : sensor_1005
       Component(s)                            : xmlproxyd
   
       Profile Information :
   
           Name                                : export_1005
           Reporting-interval                  : 60
           Payload-size                        : 5000
           Format                              : GPB
   
   ```
   3\. **Grafana**

   All that remains is data visualization. 
   * [Start Grafana](https://grafana.com/docs/grafana/latest/setup-grafana/start-restart-grafana/) and configure it to start automatically after a reboot:
      ```
      sudo systemctl start grafana-server
      sudo systemctl enable grafana-server.service
      ```
   * Add a [data source](https://grafana.com/docs/grafana/latest/administration/data-source-management/#add-a-data-source):
     
      <img height="789" alt="image" src="https://github.com/agantonov/telemetry/assets/34284048/c1a9dbde-6776-419f-b28e-e9ca46100d63">
   
   The most challenging part here is creating FLEX requests to InfluxDB for the necessary metrics on our dashboard (especially if you are not familiar with the FLEX query language). I    will focus on visualizing only the IN/OUT packet and bit counters, excluding all metrics under the `/interfaces/interface/state/counters` tree. To accomplish this, I've designed a [dashboard](https://github.com/agantonov/telemetry/blob/main/openconfig_demo_dashboard.json) using the following FLEX queries to the database:
   
   
   ACX7509-2: PPS
   ```
   from(bucket:"poc")
    |> range(start: -3h)
    |> filter(fn: (r) => r.source == "acx7509-2" and r.name =~ /et-*/ and r._field =~ /(in|out)_pkts/ )
    |> derivative(nonNegative: true)
    |> aggregateWindow(every: v.windowPeriod, createEmpty: false, fn: mean)
    |> yield (name: "mean")
   ```
   MX204-83: PPS
   ```
   from(bucket:"poc")
    |> range(start: -3h)
    |> filter(fn: (r) => r.source == "mx204-83" and r.name =~ /et-*/ and r._field =~ /(in|out)_pkts/ )
    |> derivative(nonNegative: true)
    |> aggregateWindow(every: v.windowPeriod, createEmpty: false, fn: mean)
    |> yield (name: "mean")
   ```
   ACX7509-2: BPS
   ```
   from(bucket:"poc")
    |> range(start: -3h)
    |> filter(fn: (r) => r.source == "acx7509-2" and r.name =~ /et-*/ and r._field =~ /(in|out)_octets/ )
    |> map(fn: (r) => ({r with _value: r._value * 8}))
    |> derivative(nonNegative: true)
    |> aggregateWindow(every: v.windowPeriod, createEmpty: false, fn: mean)
    |> yield (name: "mean")
   ```
   MX204-83: BPS
   ```
   from(bucket:"poc")
    |> range(start: -3h)
    |> filter(fn: (r) => r.source == "mx204-83" and r.name =~ /et-*/ and r._field =~ /(in|out)_octets/ )
    |> map(fn: (r) => ({r with _value: r._value * 8}))
    |> derivative(nonNegative: true)
    |> aggregateWindow(every: v.windowPeriod, createEmpty: false, fn: mean)
    |> yield (name: "mean")
   ```
   This is what it looks like in a browser window:
   <img width="1791" alt="image" src="https://github.com/agantonov/telemetry/assets/34284048/752053df-2b28-41ac-8b9e-ca8899a3b9ee">


Thus, we have illustrated how OpenConfig telemetry can be easily implemented in the network, providing a flexible vendor-agnostic approach to real-time monitoring. All tools are ready to use.
