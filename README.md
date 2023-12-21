# Telemetry

## Installation
1. Install Telegraf (Ubuntu) https://docs.influxdata.com/telegraf/v1/install/
   ```
   curl -s https://repos.influxdata.com/influxdata-archive.key > influxdata-archive.key
   echo '943666881a1b8d9b849b74caebf02d3465d6beb716510d86a39f6c8e8dac7515 influxdata-archive.key' | sha256sum -c && cat influxdata-archive.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/influxdata-archive.gpg > /dev/null
   echo 'deb [signed-by=/etc/apt/trusted.gpg.d/influxdata-archive.gpg] https://repos.influxdata.com/debian stable main' | sudo tee /etc/apt/sources.list.d/influxdata.list
   sudo apt-get update && sudo apt-get install telegraf
   ```
2. Install InfluxDB v2 (Ubuntu) https://docs.influxdata.com/influxdb/v2/install/
   ```
   sudo apt-get update
   sudo apt-get install influxdb2
   ```
3. Install Grafana (Ubuntu) https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/
   ```
   sudo apt-get install -y apt-transport-https software-properties-common wget
   sudo mkdir -p /etc/apt/keyrings/
   wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
   echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
   sudo apt-get update
   sudo apt-get install grafana
   ```
   ## Configuration
1. InfluxDB
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
2. Juniper router
   ```
   set system services extension-service request-response grpc clear-text port 32767
   set system services extension-service request-response grpc skip-authentication
   ```
4. Telegraf
   ```
   grep -v '^ *#\|^$' /etc/telegraf/telegraf.conf
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
     addresses = ["acx7509-2:32767","mx204-83:32767"]
     encoding = "proto"
     redial = "10s"
     max_msg_size = "10MB"
     vendor_specific = ["juniper_header"]
     [[inputs.gnmi.subscription]]
          name = "interface"
          path="/interfaces/interface"
          subscription_mode = "sample"
          sample_interval = "60s" 
   ```
   (Optional)
   ```
   [[outputs.file]]
   files = ["stdout", "/tmp/metrics.out"]
   ```
   Start Telegraf
   ```
   sudo systemctl start telegraf
   sudo systemctl enable telegraf
   ```
   Make sure that Telegraf subscribed to telenetry data via gNMI
   ```
   mx204# run show agent sensors
 
   (omitted for bravity)

   Sensor Information :
  
      Name                                    : sensor_1004_3_1
      Resource                                : /interfaces/interface/
      Version                                 : 1.0
      Sensor-id                               : 3143451993
      Subscription-ID                         : 1004
      Parent-Sensor-Name                      : sensor_1004
      Component(s)                            : l2ald
  
      Profile Information :
  
          Name                                : export_1004
          Reporting-interval                  : 60
          Payload-size                        : 5000
          Format                              : GPB

   (omitted for bravity)
   ```

5. Start Granafa https://grafana.com/docs/grafana/latest/setup-grafana/start-restart-grafana/
   ```
   sudo systemctl start grafana-server
   sudo systemctl enable grafana-server.service
   ```
   Add a data source https://grafana.com/docs/grafana/latest/administration/data-source-management/#add-a-data-source
   <img height="789" alt="image" src="https://github.com/agantonov/telemetry/assets/34284048/c1a9dbde-6776-419f-b28e-e9ca46100d63">

   Create a [dashboard](https://github.com/agantonov/telemetry/blob/main/my_dashboard.json) pulling data from InfluxDB
   <img width="1789" alt="image" src="https://github.com/agantonov/telemetry/assets/34284048/78a3028d-dea9-4b3e-96d1-bc7c9992fe59">

   Flux Queries:
   ```
   from(bucket:"poc")
   |> range(start: -1h)
   |> filter(fn: (r) => r.source == "acx7509-2" and r.name =~ /ae*/ and r.index == "0" and r._field =~ /subinterfaces\/subinterface\/state\/counters\/(in|out)_pkts/ )
   |> derivative(nonNegative: true)
   |> aggregateWindow(every: v.windowPeriod, createEmpty: false, fn: mean)
   |> yield (name: "mean")

   from(bucket:"poc")
   |> range(start: -10h)
   |> filter(fn: (r) => r.source == "acx7509-2" and r.name =~ /ae*/ and r.index == "0" and r._field =~ /subinterfaces\/subinterface\/state\/counters\/(in|out)_octets/ )
   |> map(fn: (r) => ({r with _value: r._value * 8}))
   |> derivative(nonNegative: true)
   |> aggregateWindow(every: v.windowPeriod, createEmpty: false, fn: mean)
   |> yield (name: "mean")

   from(bucket:"poc")
   |> range(start: -1h)
   |> filter(fn: (r) => r.source == "mx204-83" and r.index == "0" and r.name =~ /^et-*/ and r._field =~ /subinterfaces\/subinterface\/state\/counters\/(in|out)_pkts/ )
   |> derivative(nonNegative: true)
   |> aggregateWindow(every: v.windowPeriod, createEmpty: false, fn: mean)
   |> yield (name: "mean")

   from(bucket:"poc")
   |> range(start: -10h)
   |> filter(fn: (r) => r.source == "mx204-83" and r.name =~ /et-*/ and r.index == "0" and r._field =~ /subinterfaces\/subinterface\/state\/counters\/(in|out)_octets/ )
   |> map(fn: (r) => ({r with _value: r._value * 8}))
   |> derivative(nonNegative: true)
   |> aggregateWindow(every: v.windowPeriod, createEmpty: false, fn: mean)
   |> yield (name: "mean")
   ```
  

   
