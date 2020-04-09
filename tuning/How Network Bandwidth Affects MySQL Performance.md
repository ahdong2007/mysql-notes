Network is a major part of a database infrastructure. However, often performance benchmarks are done on a local machine, where a client and a server are collocated – I am guilty myself. This is done to simplify the setup and to exclude one more variable (the networking part), but with this we also miss looking at how network affects performance.

The network is even more important for clustering products like [Percona XtraDB Cluster](https://www.percona.com/software/mysql-database/percona-xtradb-cluster) and [MySQL Group Replication](https://dev.mysql.com/doc/refman/8.0/en/group-replication.html). Also, we are working on our [Percona XtraDB Cluster Operator](https://www.percona.com/blog/2019/01/18/percona-xtradb-cluster-operator-early-access-0-2-0-release-is-now-available/) for Kubernetes and OpenShift, where network performance is critical for overall performance.

In this post, I will look into networking setups. These are simple and trivial, but are a building block towards understanding networking effects for more complex setups.

## Setup

I will use two bare-metal servers, connected via a dedicated 10Gb network. I will emulate a 1Gb network by changing the network interface speed with ethtool -s eth1 speed 1000 duplex full autoneg off  command.

![network test topology](https://www.percona.com/blog/wp-content/uploads/2019/02/network-test-topology.png)

I will run a simple benchmark:

Shell

| 1   | sysbench oltp_read_only --mysql-ssl=on --mysql-host=172.16.0.1 --tables=20 --table-size=10000000 --mysql-user=sbtest --mysql-password=sbtest --threads=$i --time=300 --report-interval=1 --rand-type=pareto |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

This is run with the number of threads varied from 1 to 2048. All data fits into memory – innodb_buffer_pool_size is big enough – so the workload is CPU-intensive in memory: there is no IO overhead.

Operating System: Ubuntu 16.04

### Benchmark N1. Network bandwidth

In the first experiment I will compare 1Gb network vs 10Gb network.

![1gb vs 10gb network](https://www.percona.com/blog/wp-content/uploads/2019/02/1gb-vs-10gb-network.png)

| **threads/throughput** | **1Gb network** | **10Gb network** |
| ---------------------- | --------------- | ---------------- |
| 1                      | 326.13          | 394.4            |
| 4                      | 1143.36         | 1544.73          |
| 16                     | 2400.19         | 5647.73          |
| 32                     | 2665.61         | 10256.11         |
| 64                     | 2838.47         | 15762.59         |
| 96                     | 2865.22         | 17626.77         |
| 128                    | 2867.46         | 18525.91         |
| 256                    | 2867.47         | 18529.4          |
| 512                    | 2867.27         | 17901.67         |
| 1024                   | 2865.4          | 16953.76         |
| 2048                   | 2761.78         | 16393.84         |

Obviously the 1Gb network performance is a bottleneck here, and we can improve our results significantly if we move to the 10Gb network.

To see that 1Gb network is bottleneck we can check the network traffic chart in PMM:

![network traffic in PMM](https://www.percona.com/blog/wp-content/uploads/2019/02/network-traffic-in-PMM.png)

We can see we achieved 116MiB/sec (or 928Mb/sec)  in throughput, which is very close to the network bandwidth.

But what we can do if the our network infrastructure is limited to 1Gb?

### Benchmark N2. Protocol compression

There is a feature in MySQL protocol whereby you can see the compression for the network exchange between client and server: --mysql-compression=on  for sysbench.

Let’s see how it will affect our results.

![1gb network with compression protocol](https://www.percona.com/blog/wp-content/uploads/2019/02/1gb-network-with-compression-protocol.png)

| threads/throughput | 1Gb network | 1Gb with compression protocol |
| ------------------ | ----------- | ----------------------------- |
| 1                  | 326.13      | 198.33                        |
| 4                  | 1143.36     | 771.59                        |
| 16                 | 2400.19     | 2714                          |
| 32                 | 2665.61     | 3939.73                       |
| 64                 | 2838.47     | 4454.87                       |
| 96                 | 2865.22     | 4770.83                       |
| 128                | 2867.46     | 5030.78                       |
| 256                | 2867.47     | 5134.57                       |
| 512                | 2867.27     | 5133.94                       |
| 1024               | 2865.4      | 5129.24                       |
| 2048               | 2761.78     | 5100.46                       |

Here is an interesting result. When we use all available network bandwidth, the protocol compression actually helps to improve the result.![10g network with compression protocol](https://www.percona.com/blog/wp-content/uploads/2019/02/10g-network-with-compression-protocol.png)

| threads/throughput | 10Gb     | 10Gb with compression |
| ------------------ | -------- | --------------------- |
| 1                  | 394.4    | 216.25                |
| 4                  | 1544.73  | 857.93                |
| 16                 | 5647.73  | 3202.2                |
| 32                 | 10256.11 | 5855.03               |
| 64                 | 15762.59 | 8973.23               |
| 96                 | 17626.77 | 9682.44               |
| 128                | 18525.91 | 10006.91              |
| 256                | 18529.4  | 9899.97               |
| 512                | 17901.67 | 9612.34               |
| 1024               | 16953.76 | 9270.27               |
| 2048               | 16393.84 | 9123.84               |

But this is not the case with the 10Gb network. The CPU resources needed for compression/decompression are a limiting factor, and with compression the throughput actually only reach half of what we have without compression.

Now let’s talk about protocol encryption, and how using SSL affects our results.

### Benchmark N3. Network encryption

![1gb network and 1gb with SSL](https://www.percona.com/blog/wp-content/uploads/2019/02/1gb-network-and-1gb-with-SSL.png)

| threads/throughput | 1Gb network | 1Gb SSL |
| ------------------ | ----------- | ------- |
| 1                  | 326.13      | 295.19  |
| 4                  | 1143.36     | 1070    |
| 16                 | 2400.19     | 2351.81 |
| 32                 | 2665.61     | 2630.53 |
| 64                 | 2838.47     | 2822.34 |
| 96                 | 2865.22     | 2837.04 |
| 128                | 2867.46     | 2837.21 |
| 256                | 2867.47     | 2837.12 |
| 512                | 2867.27     | 2836.28 |
| 1024               | 2865.4      | 1830.11 |
| 2048               | 2761.78     | 1019.23 |

![10gb network and 10gb with SSL](https://www.percona.com/blog/wp-content/uploads/2019/02/10gb-network-and-10gb-with-SSL.png)

| **threads/throughput** | **10Gb** | **10Gb SSL** |
| ---------------------- | -------- | ------------ |
| 1                      | 394.4    | 359.8        |
| 4                      | 1544.73  | 1417.93      |
| 16                     | 5647.73  | 5235.1       |
| 32                     | 10256.11 | 9131.34      |
| 64                     | 15762.59 | 8248.6       |
| 96                     | 17626.77 | 7801.6       |
| 128                    | 18525.91 | 7107.31      |
| 256                    | 18529.4  | 4726.5       |
| 512                    | 17901.67 | 3067.55      |
| 1024                   | 16953.76 | 1812.83      |
| 2048                   | 16393.84 | 1013.22      |

For the 1Gb network, SSL encryption shows some penalty – about 10% for the single thread – but otherwise we hit the bandwidth limit again. We also see some scalability hit on a high amount of threads, which is more visible in the 10Gb network case.

With 10Gb, the SSL protocol does not scale after 32 threads. Actually, it appears to be a scalability problem in OpenSSL 1.0, which MySQL currently uses.

In our experiments, we saw that OpenSSL 1.1.1 provides much better scalability, but you need to have a special build of MySQL from source code linked to OpenSSL 1.1.1 to achieve this. I don’t show them here, as we do not have production binaries.

## Conclusions

1. Network performance and utilization will affect the general application throughput.
2. Check if you are hitting network bandwidth limits
3. Protocol compression can improve the results if you are limited by network bandwidth, but also can make things worse if you are not
4. SSL encryption has some penalty (~10%) with a low amount of threads, but it does not scale for high concurrency workloads.
