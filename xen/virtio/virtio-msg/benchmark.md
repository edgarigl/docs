# Benchmarks

## 8 Feb 2025

With slow byte memcpy.
```console
root@versal-generic:~# iperf3 -c 10.0.6.238 -bidir
Connecting to host 10.0.6.238, port 5201
[  5] local 10.0.6.136 port 60268 connected to 10.0.6.238 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  68.6 MBytes   576 Mbits/sec  652   29.7 KBytes       
[  5]   1.00-2.00   sec  63.1 MBytes   529 Mbits/sec  506   36.8 KBytes       
[  5]   2.00-3.00   sec  59.9 MBytes   502 Mbits/sec  509   24.0 KBytes       
[  5]   3.00-4.00   sec  61.9 MBytes   520 Mbits/sec  552   24.0 KBytes       
[  5]   4.00-5.00   sec  65.1 MBytes   546 Mbits/sec  645   26.9 KBytes       
[  5]   5.00-6.00   sec  63.1 MBytes   530 Mbits/sec  519   82.0 KBytes       
[  5]   6.00-7.00   sec  58.0 MBytes   487 Mbits/sec  427   31.1 KBytes       
[  5]   7.00-8.00   sec  62.2 MBytes   522 Mbits/sec  499   31.1 KBytes       
[  5]   8.00-9.00   sec  64.4 MBytes   540 Mbits/sec  520   35.4 KBytes       
[  5]   9.00-10.00  sec  58.9 MBytes   493 Mbits/sec  509   48.1 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec   625 MBytes   524 Mbits/sec  5338             sender
[  5]   0.00-10.01  sec   623 MBytes   523 Mbits/sec                  receiver

iperf Done.
```

Generic 32bit word memcpy from TBM:
```console
# iperf3 -c 10.0.6.238 -bidir
Connecting to host 10.0.6.238, port 5201
[  5] local 10.0.6.122 port 37840 connected to 10.0.6.238 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec   114 MBytes   954 Mbits/sec    0    387 KBytes       
[  5]   1.00-2.00   sec   112 MBytes   935 Mbits/sec    0    387 KBytes       
[  5]   2.00-3.00   sec   112 MBytes   938 Mbits/sec    0    407 KBytes       
[  5]   3.00-4.00   sec   111 MBytes   934 Mbits/sec    0    407 KBytes       
[  5]   4.00-5.00   sec   112 MBytes   936 Mbits/sec    0    407 KBytes       
[  5]   5.00-6.00   sec   112 MBytes   940 Mbits/sec    0    407 KBytes       
[  5]   6.00-7.00   sec   111 MBytes   934 Mbits/sec    0    407 KBytes       
[  5]   7.00-8.00   sec   111 MBytes   934 Mbits/sec    0    407 KBytes       
[  5]   8.00-9.00   sec   112 MBytes   938 Mbits/sec    0    407 KBytes       
[  5]   9.00-10.00  sec   112 MBytes   935 Mbits/sec    0    407 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  1.09 GBytes   938 Mbits/sec    0             sender
[  5]   0.00-10.01  sec  1.09 GBytes   936 Mbits/sec                  receiver

iperf Done.
```

