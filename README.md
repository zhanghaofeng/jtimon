# jtimon
Junos Telemetry Interface client

# Setup
<pre>
1. Install golang
2. Set GOROOT/GOPATH as needed
3. git clone https://github.com/nileshsimaria/jtimon.git
4. Fetch following dependent packages

    $ go get github.com/golang/protobuf/proto
    $ go get github.com/gorilla/mux
    $ go get github.com/influxdata/influxdb/client/v2
    $ go get github.com/prometheus/client_golang/prometheus/promhttp
    $ go get github.com/spf13/pflag
    $ go get golang.org/x/net/context
    $ go get google.golang.org/grpc
    $ go get github.com/nileshsimaria/jtimon/telemetry
    $ go get github.com/nileshsimaria/jtimon/authentication

5. cd jtimon
6. go build
7. ./jtimon --help
</pre>

# CLI Options

<pre>
$ ./jtimon --help
Usage of ./jtimon:
      --cert string                   CA certificate file
      --compression string            Enable HTTP/2 compression (gzip, deflate)
      --config string                 Config file name
      --drop-check                    Check for packet drops
      --gtrace                        Collect GRPC traces
      --latency-check                 Check for latency
      --log string                    Log file name
      --max-kv int                    Max kv
      --max-run int                   Max run time in seconds
      --pdt                           PDT style influx DB schema
      --prefix-check                  Report missing __prefix__ in telemetry packet
      --print                         Print Telemetry data
      --prometheus                    Stats for prometheus monitoring system
      --server-host-override string   ServerName used to verify the hostname
      --sleep int                     Sleep after each read (ms)
      --stats int                     Collect and Print statistics periodically
      --time-diff                     Time Diff for sensor analysis using InfluxDB
      --tls                           Connection uses TLS
</pre>      

# Config
<pre>
Sample JSON config file to subscribe /interfaces @2s, /bpg @10s and /components @10s.

{
    "host": "Junos-Device-IP",
    "port": 50051, ## Junos gRPC port
    "user": "uname", 
    "password": "pwd",
    "cid": "cid-2", ## unique client ID
    "grpc" : {
        "ws" : 524288 ## advertise HTTP2 window size (512K) (default 64K)
    },
    "api": {
        "port" : 7878 ## send pause / unpause command to this port (optional)
    },
    "influx" : { ## influx DB config (optional)
        "server" : "127.0.0.1",
        "port" : 8086,
        "dbname" : "vptx-db",
        "measurement" : "vptx",
        "recreate" : true, ## Recreate the said DB (nuke old one)
        "user" : "influx",
        "password" : "influxdb"
    },    
    "paths": [{
        "path": "/interfaces",
        "freq": 2000
	}, {
        "path": "/bgp",
        "freq": 10000
	}, {
        "path": "/components",
        "freq": 10000
    }]
}

Sample run which collect the stats too. Print stats every nth second.
$ ./jtimon --config json-file-name --stats n 
</pre>

# Example run with output

1. Run and collect stats (print stats on screen every 10 seconds). Print Summary in the end.

<pre>
$ ./jtimon --config sample-config/nsimaria-vptx.json --stats 10
2017/06/10 13:02:13
+------------------------------+--------------------+--------------------+--------------------+--------------------+
|         Timestamp            |        KV          |      Packets       |       Bytes        |     Bytes(wire)    |
+------------------------------+--------------------+--------------------+--------------------+--------------------+
| Sat Jun 10 13:02:21 PDT 2017 |               1992 |                 28 |             146416 |             146416 |
| Sat Jun 10 13:02:31 PDT 2017 |               4314 |                 62 |             318344 |             318344 |
| Sat Jun 10 13:02:41 PDT 2017 |               6804 |                 97 |             501354 |             501354 |
| Sat Jun 10 13:02:51 PDT 2017 |               9294 |                132 |             684364 |             684364 |
| Sat Jun 10 13:03:01 PDT 2017 |              11784 |                167 |             867374 |             867374 |
| Sat Jun 10 13:03:11 PDT 2017 |              14274 |                202 |            1050384 |            1050384 |
| Sat Jun 10 13:03:21 PDT 2017 |              16764 |                237 |            1233402 |            1233402 |
| Sat Jun 10 13:03:31 PDT 2017 |              19254 |                272 |            1416432 |            1416432 |
| Sat Jun 10 13:03:41 PDT 2017 |              21744 |                307 |            1599462 |            1599462 |
^C

Collector Stats (Run time : 1m30.862554177s)
310          : in-packets
22113        : data points (KV pairs)
321          : in-header wirelength (bytes)
1624296      : in-payload length (bytes)
1624296      : in-payload wirelength (bytes)
18047        : throughput (bytes per seconds)
</pre>

2. Same as above - with drop-check

<pre>
$ ./jtimon --config sample-config/nsimaria-vptx.json --stats 10 --drop-check
2017/06/10 13:07:16
+------------------------------+--------------------+--------------------+--------------------+--------------------+
|         Timestamp            |        KV          |      Packets       |       Bytes        |     Bytes(wire)    |
+------------------------------+--------------------+--------------------+--------------------+--------------------+
| Sat Jun 10 13:07:24 PDT 2017 |               1640 |                 26 |             122482 |             122482 |
| Sat Jun 10 13:07:34 PDT 2017 |               4130 |                 61 |             305493 |             305493 |
| Sat Jun 10 13:07:44 PDT 2017 |               6620 |                 96 |             488503 |             488503 |
| Sat Jun 10 13:07:54 PDT 2017 |               9110 |                131 |             671513 |             671513 |
| Sat Jun 10 13:08:04 PDT 2017 |              11600 |                166 |             854523 |             854523 |
^C
 Drops Distribution
+----+-----+-------+----------+-------------------------------------------------------------------------------------------------------------------------+
| CID |SCID| Drops | Received | Sensor Path                                                                                                             |
+----+-----+-------+----------+-------------------------------------------------------------------------------------------------------------------------+
|65535|   0|      0|       24 | sensor_1001:/bgp:/bgp:rpd                                                                                               |
|65535|   0|      0|       96 | sensor_1002:/components:/components:chassisd                                                                            |
|65535|   0|      0|       24 | sensor_1000_3_1:/interfaces:/interfaces:mib2d                                                                           |
|65535|   0|      0|       24 | sensor_1000_5_1:/interfaces:/interfaces:xmlproxyd                                                                       |
+----+-----+-------+----------+-------------------------------------------------------------------------------------------------------------------------+

Collector Stats (Run time : 50.62200827s)
168          : in-packets
11952        : data points (KV pairs)
321          : in-header wirelength (bytes)
878458       : in-payload length (bytes)
878458       : in-payload wirelength (bytes)
17569        : throughput (bytes per seconds)
0            : total packet drops
</pre>

3. Same as above - with drop-check and latency-check and some sleep in between packets to simulate latencies.

<pre>

$ ./jtimon --config sample-config/nsimaria-vptx.json --stats 2 --drop-check --latency-check --sleep 1000
2017/06/10 13:15:36
+------------------------------+--------------------+--------------------+--------------------+--------------------+-----------------+
|         Timestamp            |        KV          |      Packets       |       Bytes        |     Bytes(wire)    | Average Latency |
+------------------------------+--------------------+--------------------+--------------------+--------------------+-----------------+
| Sat Jun 10 13:15:38 PDT 2017 |                 17 |                  1 |                901 |                901 |
| Sat Jun 10 13:15:40 PDT 2017 |                 96 |                  3 |               8501 |               8501 |             701 |
| Sat Jun 10 13:15:42 PDT 2017 |                146 |                  5 |              12669 |              12669 |            1592 |
| Sat Jun 10 13:15:44 PDT 2017 |                498 |                  7 |              36604 |              36604 |            2388 |
| Sat Jun 10 13:15:46 PDT 2017 |                548 |                  9 |              40220 |              40220 |            3067 |
| Sat Jun 10 13:15:48 PDT 2017 |                620 |                 11 |              47338 |              47338 |            3776 |
| Sat Jun 10 13:15:50 PDT 2017 |                812 |                 13 |              60355 |              60355 |            4522 |
| Sat Jun 10 13:15:52 PDT 2017 |               1013 |                 15 |              74107 |              74107 |            5261 |
| Sat Jun 10 13:15:54 PDT 2017 |               1092 |                 17 |              81709 |              81709 |            5944 |
| Sat Jun 10 13:15:56 PDT 2017 |               1142 |                 19 |              85877 |              85877 |            6672 |
| Sat Jun 10 13:15:59 PDT 2017 |               1310 |                 20 |              96959 |              96959 |            7035 |
| Sat Jun 10 13:16:01 PDT 2017 |               1511 |                 22 |             110711 |             110711 |            7857 |
| Sat Jun 10 13:16:03 PDT 2017 |               1590 |                 24 |             118311 |             118311 |            8625 |
| Sat Jun 10 13:16:05 PDT 2017 |               1640 |                 26 |             122479 |             122479 |            9413 |
| Sat Jun 10 13:16:07 PDT 2017 |               1992 |                 28 |             146414 |             146414 |           10193 |
| Sat Jun 10 13:16:09 PDT 2017 |               2042 |                 30 |             150029 |             150029 |           10945 |
| Sat Jun 10 13:16:11 PDT 2017 |               2114 |                 32 |             157146 |             157146 |           11697 |
| Sat Jun 10 13:16:13 PDT 2017 |               2306 |                 34 |             170163 |             170163 |           12456 |
| Sat Jun 10 13:16:15 PDT 2017 |               2507 |                 36 |             183915 |             183915 |           13212 |
| Sat Jun 10 13:16:17 PDT 2017 |               2586 |                 38 |             191515 |             191515 |           13941 |
| Sat Jun 10 13:16:19 PDT 2017 |               2636 |                 40 |             195683 |             195683 |           14687 |
| Sat Jun 10 13:16:21 PDT 2017 |               2988 |                 42 |             219618 |             219618 |           15433 |
^C
 Drops Distribution
+----+-----+-------+----------+-------------------------------------------------------------------------------------------------------------------------+
| CID |SCID| Drops | Received | Sensor Path                                                                                                             |
+----+-----+-------+----------+-------------------------------------------------------------------------------------------------------------------------+
|65535|   0|      0|        7 | sensor_1001:/bgp:/bgp:rpd                                                                                               |
|65535|   0|      0|       24 | sensor_1002:/components:/components:chassisd                                                                            |
|65535|   0|      0|        6 | sensor_1000_3_1:/interfaces:/interfaces:mib2d                                                                           |
|65535|   0|      0|        6 | sensor_1000_5_1:/interfaces:/interfaces:xmlproxyd                                                                       |
+----+-----+-------+----------+-------------------------------------------------------------------------------------------------------------------------+

Collector Stats (Run time : 46.425333948s)
43           : in-packets
3005         : data points (KV pairs)
321          : in-header wirelength (bytes)
220517       : in-payload length (bytes)
220517       : in-payload wirelength (bytes)
4793         : throughput (bytes per seconds)
42           : latency sample packets
663740       : latency (ms)
15803        : average latency (ms)
0            : total packet drops
</pre>
