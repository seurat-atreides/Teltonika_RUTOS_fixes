# Introduction
While debugging the Data-To-Server functionality on a RUT360 I required tcpdump to debug the traffic fron the router to my Ubidots account since I found no other means of looking at the CURL message payload.
I installed the tcpdump package and proceeded to configure the tcpdump trace under: System -> Maintenance -> Troubleshoot.
I set the traffic filter as follows:
![image](https://github.com/seurat-atreides/Teltonika_RUTOS_fixes/assets/30745827/f69256be-0241-4f48-a01b-d1d6214522fb)

After clicking Save & Aply I noticed that the tcpdump process was not running.
If I instead configure the tcpdump service without ptocol filter the service script works fine:
```
# ps -w | grep tcpdump
29487 root      5092 S    /usr/sbin/tcpdump -i any -Q inout -C 20 -W 1 host 3.19.87.203 and port 80 -w /tmp/tcpdebug.pcap
```
Configuring tcpdump with protocl filter but no host or port filter also works:
```shell
# ps -w | grep tcpdump
31032 root      5092 S    /usr/sbin/tcpdump -i any -Q inout tcp -C 20 -W 1 -w /tmp/tcpdebug.pcap
```
But the composition of the tcpdump command is not what I expected. IMHO, the host and port parameters should be part of the filter and not the options.

# Analysis of the root cause for the problem
Analysing the code in the tcpdebubg script it looks like the following code block is the cause for the script not producing the expected result when host and port parameters are included:
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
I.e., if filter is not empty, then append it to the command. After this append the options.
This is destined to fail.
The filter should be appended to the command after the options and the syntax should include the required logical oprator "and". E.g., "tcp and host <IPaddr> and port <port_nr>".
The root cause beeing thjat you can't add the protocol filter and then the options containing the host and/or port parameters.
This leads to the tcpdump command failing execution!
I see no point in using cryptic && operators where If-then-elif-fi statements would help readability.

# Script refactoring proposal
Following is my proposal for a replacement of the [test] && statements in a more traditional readable If-elif-fi form:
```shell
if [ "$storage" = "/tmp" ]
then
		OPTIONS="-C 1 -W 1"
fi
# Added a new filter creation logic to make it work and more readable
if [ "$host" != "" ] && [ "$port" != "" ]
then
		if [ "$filter" != "" ]
		then
				filter="$filter and host $host and port $port"
		else
				filter=" host $host and port $port"
		fi
elif [ "$host" != "" ] && [ "$port" = "" ]
then
		if [ "$filter" != "" ]
		then
				filter="$filter and host $host"
		else
				filter=" host $host"
		fi
elif [ "$host" = "" ] && [ "$port" != "" ]
then
		if [ "$filter" != "" ]
		then
				filter="$filter and port $port"
		else
				filter=" port $port"
		fi
fi
```
After this code block I propose to append the configuration parameters in a more traditional manner:
```shell
procd_append_param command $OPTIONS
[ -n "$interface" ] && procd_append_param command -i "$interface"
[ -n "$direction" ] && procd_append_param command -Q "$direction"
[ -n "$filter" ] && procd_append_param command "$filter"
procd_append_param command -w "$storage"/tcpdebug.pcap
```
Which will yield the following tcpdump command for the same configuration parameters shown in the introduction:
```
ps -w | grep tcpdump
 3507 root      5092 S    /usr/sbin/tcpdump -C 1 -W 1 -i any -Q inout tcp and host 3.139.145.194 and port 80 -w /tmp/tcpdebug.pcap
 3711 root      2036 S    grep tcpdump
```
This is the command composition I was expecting to see!<br>
BTW, I reduced the PCAP file size to 1MB (-C 1) and no rotation (-W 1) to keep the RAM usage lower.

> [!NOTE]
> The script was refactored using VS-Code. AKAIK it's ScriptCheck conform.
