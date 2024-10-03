# Introduction
While debugging the Data-To-Server functionality on a RUT360 I required tcpdump to debug the traffic fron the router to my Ubidots account since I found no other means of looking at the CURL message payload.
I installed the tcpdump package and proceeded to configure the tcpdump trace under: System -> Maintenance -> Troubleshoot.
I set the traffic filter as follows:
![image](https://github.com/seurat-atreides/Teltonika_RUTOS_fixes/assets/30745827/f69256be-0241-4f48-a01b-d1d6214522fb)

After clicking Save & Apply I noticed that the tcpdump process was not running.

If I instead configure the tcpdump service without protocoll filter the service script works fine:
```shell
# ps -w | grep tcpdump
29487 root      5092 S    /usr/sbin/tcpdump -i any -Q inout -C 20 -W 1 host 3.19.87.203 and port 80 -w /tmp/tcpdebug.pcap
```
Configuring tcpdump with protocoll filter but no host or port filter does works:
```shell
# ps -w | grep tcpdump
31032 root      5092 S    /usr/sbin/tcpdump -i any -Q inout tcp -C 20 -W 1 -w /tmp/tcpdebug.pcap
```
But the composition of the tcpdump command is not what I expected. IMHO, the host and port parameters should be part of the filter and not the options.

# Analysis of the root cause for the problem
Analysing the code in the tcpdebubg script it looks like the following code block is the cause for the script not producing the expected result when host and port parameters are included together with the protocoll filter:
```shell
[ "$storage" = "/tmp" ] && options="-C 20 -W 1"
[ "$host" != "" -a "$port" != "" ] && options="$options host $host and port $port"
[ "$host" != "" -a "$port" = "" ] && options="$options host $host"
[ "$host" = "" -a "$port" != "" ] && options="$options port $port"
```
The author decided to make the host and port filter parameters part of the options that later are appended to the tcpdump command after the prococol type filter is appended.
```shell
[ -n "$filter" ] && procd_append_param command "$filter"
procd_append_param command $options
```
I.e., if $filter is not empty, then append it to the command. After this append the options.
This is destined to fail.
The filter should be appended to the command after the options and the syntax should include the required logical oprator "and". E.g., "tcp and host <IPaddr> and port <port_nr>".
The root cause beeing thjat you can't add the protocol filter and then the options containing the host and/or port parameters.
This leads to the tcpdump command failing execution!

BTW, I see no point in using the cryptic && oneliners where If-then-elif-fi statements would help readability.

# Script refactoring proposal
Following is the diff patch I propose to make tcpdebug work as expected:
```shell
--- tcpdebug.org        2024-10-01 11:38:30.000000000 +0200
+++ tcpdebug    2024-10-03 10:30:37.551164803 +0200
@@ -23,12 +23,12 @@
        [ "$host" != "" -a "$port" != "" ] && options="$options host $host and port $port"
        [ "$host" != "" -a "$port" = "" ] && options="$options host $host"
        [ "$host" = "" -a "$port" != "" ] && options="$options port $port"
+       [ -n "$filter" ] && [ "$port" != "" -o "$host" != "" ] && options="$options and $filter" || options="$options $filter"

        procd_open_instance
        procd_set_param command /usr/sbin/tcpdump
        [ -n "$interface" ] && procd_append_param command -i "$interface"
        [ -n "$direction" ] && procd_append_param command -Q "$direction"
-       [ -n "$filter" ] && procd_append_param command "$filter"
        procd_append_param command $options
        procd_append_param command -w "$storage"/tcpdebug.pcap
        procd_set_param respawn
```
A
Which will yield the following tcpdump command for the same configuration parameters shown in the introduction:
```shell
ps -w | grep tcpdump
 3507 root      5092 S    /usr/sbin/tcpdump -C 20 -W 1 -i any -Q inout host 3.139.145.194 and port 80 and tcp -w /tmp/tcpdebug.pcap
 3711 root      2036 S    grep tcpdump
```
This is the command composition I was expecting to see!<br>
BTW, I would reduce the PCAP file size to 1MB (-C 1) and no rotation (-W 1) to keep the RAM usage lower.

> **_NOTE:_**<br>
> To debug PROCD scripts you can use INIT_TRACE=1 as prefix to the script execution:
>```shell
># INIT_TRACE=1 /etc/init.d/tcpdebug start
>```
