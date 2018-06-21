# Ingesters

This section contains more detailed instruction for configuring and running Gravwell ingesters.

The Gravwell created ingesters are released open source and can be found on [Github](https://github.com/gravwell/ingesters). Also, the ingest APIs are released open source as well so creating your own ingesters for unique data sources, performing additional normalization or pre-processing, or any other manner of things. The ingest API code [is located here](https://github.com/gravwell/ingest).

In general, when sending data from a different machine to Gravwell, the other machine will need to know the “Ingest Secret” of the Gravwell instance (this is for authentication). This can be found by viewing the `/opt/gravwell/etc/gravwell.conf` file on the Gravwell server and finding the entry for `Ingest-Auth`.

The Gravwell GUI has a Remote Ingesters tag on the System Stats page which can be used to easily identify which remote ingesters are actively connected to which indexers as well as how long they have been connected and how much data they have pushed.

![](remote-ingesters.png)

Attention: The [replication system](#!configuration/replication.md) does not replicate entries larger than 999MB. Larger entries can still be ingested and searched as usual, but they are omitted from replication. This is not a concern for 99.9% of use cases, as all the ingesters detailed in this page tend to create entries no larger than a few kilobytes.

## Global Configuration Parameters

Most of the core ingesters support a set of global configuration parameters which are shared across all the core ingesters.  The shared Global configuration parameters are implemented using the [ingest config](https://godoc.org/github.com/gravwell/ingest/config#IngestConfig) package.  Global configuration parameters should be specified in the Glboal section of each Gravwell ingester INI file.  The following Global ingester paramters are available:

* Ingest-Secret
* Connection-Timeout
* Insecure-Skip-TLS-Verify
* Cleartext-Backend-Target
* Encrypted-Backend-Target
* Pipe-Backend-Target
* Ingest-Cache-Path
* Max-Ingest-Cache
* Log-Level
* Log-File
* Source-Override

### Ingest-Secret

The Ingest-Secret parameter specifies the token to be used for ingest authentication.  The token specified here MUST match the Ingest-Auth parameter for Gravwell indexers.

### Connection-Timeout

The Connection-Timeout parameter specifies how long we want to wait for indexers before failing.  An empty timeout means that the ingester will wait forever to start.  Timeouts should be specified in durations of minutes, seconds, or hours.

#### Examples

Connection-Timeout=30s
Connection-Timeout=5m
Connection-Timeout=1h

### Insecure-Skip-TLS-Verify

The Insecure-Skip-TLS-Verify token tells the ingester to ignore bad certificates when connecting over encrypted TLS tunnels.  As the name suggests, any and all authentication provided by TLS is thrown out the window and attackers can easily Man-in-the-Middle TLS connections.  The ingest connections will still be encrypted, but the connection is by no means secure.  By default TLS certificates are validated and the connections will fail if the certificate validation fails.

#### Examples

Insecure-Skip-TLS-Verify=true
Insecure-Skip-TLS-Verify=false

### Cleartext-Backend-Target

Cleartext-Backend-Target specifies a host and port which an indexer is listening on.  The ingester will connect to the indexer using a cleartext TCP connection.  If no port is specified the default port of 4023 is used.  Cleartext connections support both IPv6 and IPv4 destinations.  Multiple Cleartext-Backend-Targets can be specified to load balance an ingester across multiple indexers.

#### Examples

Cleartext-Backend-Target=192.168.1.1
Cleartext-Backend-Target=192.168.1.1:4023
Cleartext-Backend-Target=DEAD::BEEF
Cleartext-Backend-Target=[DEAD::BEEF]:4023

### Encrypted-Backend-Target

Encrypted-Backend-Target specifies a host and port which an indexer is listening on.  The ingester will connect to the indexer using a TCP connection and perform a full TLS handshake and certificate validation.  If no port is specified the default port of 4024 is used.  Encrypted connections support both IPv6 and IPv4 destinations.  Multiple Encrypted-Backend-Targets can be specified to load balance an ingester across multiple indexers.

#### Examples

Encrypted-Backend-Target=192.168.1.1
Encrypted-Backend-Target=192.168.1.1:4023
Encrypted-Backend-Target=DEAD::BEEF
Encrypted-Backend-Target=[DEAD::BEEF]:4023

### Pipe-Backend-Target

Pip-Backend-Target specifies a unix named socket via a full path.  Unix named sockets are ideal for ingesters that are co-resident with indexers as they are extremely fast and incur little overhead.  Only a single Pipe-Backend-Target is supported per ingester, but pipes can be multiplexed along side cleartext and encrypted connections.

#### Examples

Pipe-Backend-Target=/opt/gravwell/comms/pipe
Pipe-Backend-Target=/tmp/gravwellpipe

### Ingest-Cache-Path

The Ingest-Cache-Path enables a local cache for ingested data.  When enabled ingesters can begin caching locally when they cannot forward entries to indexers.  The ingest cache can help ensure you don't lose data when links go down or if you need to take a Gravwell cluster offline momentarily.  Be sure to specify a Max-Ingest-Cache value so that a long term network failure won't cause an ingester to fill the host disk.  The local ingest cache is not as fast as ingesting directly to indexers, so don't expect the ingest cache to handle 2 million entries per second the way the indexers can.

#### Examples

Ingest-Cache-Path=/opt/gravwell/cache/simplerelay.cache
Ingest-Cache-Path=/mnt/storage/networklog.cache

### Max-Ingest-Cache

Max-Ingest-Cache provides for limiting the amount of storage the an ingester will consume when the cache is engaged.  The maximum cache value is specified in megabytes.  So a value of 1024 means that the ingester can consume 1TB of storage before it will stop accepting new entries.  The Cache system will NOT overwrite old entries when the cache fills up, this is by design so that an attacker can't disrupt a network connection and cause an ingester to overwrite potentially critical data at the point the disruption happened.

#### Examples

Max-Ingest-Cache=32
Max-Ingest-Cache=1024
Max-Ingest-Cache=10240

### Log-Level

The Log-Level parameter controlls the leveled logging system in each ingester for both log files and metadata that is sent to indexers under the "gravwell" tag.  Setting a log level to INFO allows for seeing alot of detail, such as when the file follower follows a new file or simple relay receives a new TCP connection.  The following levels are supported:

* OFF
* INFO
* WARN
* ERROR

#### Examples

Log-Level=Off
Log-Level=INFO
Log-Level=WARN
Log-Level=ERROR

### Log-File

Ingesters can log errors and debug information to log files to assist in debugging installation and configuration problems.  An empty Log-File parameter disables file logging.

#### Examples

Log-File=/top/gravwell/log/ingester.log

### Source-Override

The Source-Override parameter allows for overriding the SRC data item that is attached to each entry.  The SRC item is either an IPv6 or IPv4 address.

#### Examples

Source-Override=10.0.0.1
Source-Override=0.0.0.0
Source-Override=DEAD:BEEF::FEED:FEBE

## Simple Relay

Simple Relay is a text ingester which is capable of listening on multiple TCP or UDP ports.  Each port can be assigned a tag as well as an ingest standard (e.g. parse RFC5424 or simple newline delimited entries).  Simple Relay is the go-to ingester for ingesting remote syslog entries or consuming from any data source that can throw text logs over a network connection.

Using a terminal on the Gravwell server, issue the following command as a superuser (e.g. via the `sudo` command) to install the ingester:

```
root@gravserver ~ # bash gravwell_simple_relay_installer.sh
```

If the Gravwell services are present on the same machine, the installation script will automatically extract and configure the `Ingest-Auth` parameter and set it appropriately.  However, if your ingester is not resident on the same machine as a pre-existing Gravwell backend, it will be necessary to modify the configuration file in `/opt/gravwell/etc/simple_relay.conf` to match the `Ingest-Auth` value set on the Indexers.

An example configuration for the Simple Relay ingester, configured to listen on several ports and apply a unique tag to each is as follows:

```
[Global]
Ingest-Secret = IngestSecrets # the secret used to authenticate with indexers
Connection-Timeout = 0
Insecure-Skip-TLS-Verify = false #Validate the public keys of TLS connections
#Cleartext-Backend-target=127.0.0.1:4023 #example of adding a cleartext connection
#Cleartext-Backend-target=127.1.0.1:4023 #example of adding another cleartext connection
#Encrypted-Backend-target=127.1.1.1:4023 #example of adding an encrypted connection
#Assume-Local-Timezone=true #assume local timezone if none is found in the timestamp
#a named pipe connection, this can be used to avoid network overhead
Pipe-Backend-target=/opt/gravwell/comms/pipe
#basic default logger, all entries will go to the default tag
#no Tag-Name means use the default tag
#no Reader-Type default to newline delimited entries
[Listener "default"]
        Bind-String="0.0.0.0:7777" #we are binding to all interfaces
#syslog logger, all entries are tagged with the syslog tag
[Listener "new hotness syslog "]
       #use reliable syslog, which is syslog over TCP on port 601
       #bind ONLY to localhost with no proto specifier we default to tcp
       Bind-String = 127.0.0.1:601
       Tag-Name = syslog
[Listener "crappy old syslog"]
       #use regular old UDP syslog using the RFC5424 format
       #RFC5424 lexer also eats RFC3164 logs from legacy syslog and BSD-syslog
       Bind-String = udp://127.0.0.1:514 #bind ONLY to localhost on UDP
       Tag-Name = syslog2
       Reader-Type=rfc5424 # parse out syslog entries irrespective of newlines
       Source-Override=DEAD:BEEF::FEED:FEBE
[Listener "strange UDP line reader"]
       #NOTICE! Lines CANNOT span multiple UDP packets, if they do, they will be treated
       #as seperate entries
       Bind-String = udp://127.0.0.1:9999 #bind ONLY to localhost on UDP
       Tag-Name = udpliner
       Reader-Type=line #look for newlines to delimit entries
       Source-Override=192.168.1.1
# generic event handler, entries will be tagged with the "generic" tag
# Notice the Ignore-Timestamps directive, this tells gravwell to not attempt
# To extract a timestamp from the entry, but apply the current time to it
# This can be useful if the source timestamps are not accurate, or non-existent
[Listener "GenericEvents"]
       #example generic event handler, it takes lines, and attaches current timestamp
       Bind-String = 127.0.0.1:8888 #bind only to localhost TCP port 8888
       Tag-Name = generic
       Ignore-Timestamps = true # do not look for a timestamp, use the current time
```

## File Follower

The File Follower ingester is designed to follow files and to capture logs from sources that cannot natively integrate with Gravwell or are incapable of sending logs via a network connection.  The file follower comes in both Linux and Windows flavors and can follow any logging file that is line delimited.  It is compatible with file rotation and employs a powerful pattern matching system so that the file follower can deal with applications that are not consistent with log file names.

### Example Configurations

The file follower configuration is nearly identical for both the Windows and Linux variants.

#### Windows

The windows configuration file is located at `C:\Program Files\gravwel\file_follow.cfg` by default.  The windows file follower runs as a windows service.  Its status can be queried by issuing `sc query GravwellFileFollow` in a command prompt.  An example configuration which tracks the windows CBS log files looks like the following:

```
[Global]
Ingest-Secret = IngestSecrets
Connection-Timeout = 0
Insecure-Skip-TLS-Verify = false
Cleartext-Backend-target=172.20.0.2:4023 #example of adding a cleartext connection
State-Store-Location="c:\\Program Files\\gravwell\\file_follow.state"
Ingest-Cache-Path="c:\\Program Files\\gravwell\\file_follow.cache"
Log-Level=ERROR #options are OFF INFO WARN ERROR
#basic default logger, all entries will go to the default tag
#no Tag-Name means use the default tag
[Follower "cbs"]
        Base-Directory="C:\\Windows\\Logs\\CBS"
        File-Filter="CBS.log" #we are looking for just the CBS log
        Tag-Name=auth
        Assume-Local-Timezone=true
```

#### Linux

The linux configuration file is located at `/opt/gravwell/etc/file_follow.conf` by default.  An example configuration which watches kernel, dmesg, and debian installation logs might look like the following:

```
[Global]
Ingest-Secret = IngestSecrets
Connection-Timeout = 0
Insecure-Skip-TLS-Verify = false
Cleartext-Backend-target=172.20.0.1:4023 #example of adding a cleartext connection
Cleartext-Backend-target=172.20.0.2:4023 #example of adding another cleartext connection
#Encrypted-Backend-target=127.1.1.1:4024 #example of adding an encrypted connection
#Pipe-Backend-target=/opt/gravwell/comms/pipe #a named pipe connection, this should be used when ingester is on the same machine as a backend
State-Store-Location=/opt/gravwell/etc/file_follow.state
Log-Level=ERROR #options are OFF INFO WARN ERROR
#Ingest-Cache-Path=/opt/gravwell/cache/file_follow.cache #allows for ingested entries to be cached when indexer is not available
#basic default logger, all entries will go to the default tag
#no Tag-Name means use the default tag
[Follower "syslog"]
        Base-Directory="/var/log/"
        File-Filter="{syslog,syslog.[0-9]}" #we are looking for all authorization log files
        Tag-Name=syslog
        Assume-Local-Timezone=true #Default for assume localtime is false
[Follower "auth"]
        Base-Directory="/var/log/"
        File-Filter="{auth.log,auth.log.[0-9]}" #we are looking for all authorization log files
        Tag-Name=syslog
        Assume-Local-Timezone=true #Default for assume localtime is false
[Follower "packages"]
        Base-Directory="/var/log"
        File-Filter="{dpkg.log,dpkg.log.[0-9]}" #we are looking for all dpkg files
        Tag-Name=dpkg
        Ignore-Timestamps=true
[Follower "kernel"]
        Base-Directory="/var/log"
        File-Filter="{dmesg,dmesg.[0-9]}" #we are looking for all dpkg files
        Tag-Name=kernel
        Ignore-Timestamps=true
[Follower "kernel2"]
        Base-Directory="/var/log"
        File-Filter="{kern.log,kern.log.[0-9]}" #we are looking for all dpkg files
        Tag-Name=kernel
        Ignore-Timestamps=true
```

## Mass File Ingester

The Mass File ingester is a very powerful but specialized tool for ingesting an archive of many logs from many sources.

### Example use case
Gravwell users have needed this tool when investigating a potential breach. The user had Apache logs from over 50 different servers and needed to search them all. Ingesting them one after another causes poor temporal indexing performance. Thus, this tool was created to ingest the files while preserving the temporal nature of the log entries and ensuring solid performance.  The massfile ingester works best when the ingesting machine has enough space (storage and memory) to optimized the source logs prior to ingesting.  The optimization phase helps relieve pressure on the Gravwell storage system at ingest and during search, ensuring that incident responders can move quickly and get performant access to their log data in short order.

### Notes

The mass file ingester is driven via command line parameters and is not designed to run as a service.  The code is available on [Github](https://github.com/gravwell/ingesters).

```
Usage of ./massFile:
  -clear-conns string
        comma seperated server:port list of cleartext targets
  -ingest-secret string
        Ingest key (default "IngestSecrets")
  -no-ingest
        Optimize logs but do not perform ingest
  -pipe-conn string
        Path to pipe connection
  -s string
        Source directory containing log files
  -skip-op
        Assume working directory already has optimized logs
  -tag-name string
        Tag name for ingested data (default "default")
  -timeout int
        Connection timeout in seconds (default 1)
  -tls-conns string
        comma seperated server:port list of TLS connections
  -tls-private-key string
        Path to TLS private key
  -tls-public-key string
        Path to TLS public key
  -tls-remote-verify string
        Path to remote public key to verify against
  -w string
        Working directory for optimization
```

## Windows Event Service

The Gravwell windows events ingester runs as a service on a windows machine and sends windows events to the Gravwell indexer.

### Installation

Download the Gravwell windows ingester installer. If you do not have a link, contact a Gravwell rep at support@gravwell.io.

Run the .msi installation wizard to install the Gravwell events service.

Future versions of the wizard will allow entering and discovery of Gravwell servers, but for now, the config file located at `C:\Program Files\gravwell\config.cfg` needs to be manually configured.

Change the connection ip address to the IP of your Gravwell server.

```
Encrypted-Backend-target=ip.addr.goes.here:port
```

Once configured, this file can be copied to any other windows system on which you would like to collect events.

### Optional Sysmon Integration

A wildly effective and popular tool for monitoring windows systems is the Sysmon utility, part of the sysinternals suite. There are plenty of resources with examples of good sysmon configuration files. At Gravwell, we like to use the config created by infosec Twitter personality @InfosecTaylorSwift.

Edit the Gravwell windows agent config file located at `C:\Program Files\gravwell\config.cfg` and add the following lines:

```
[EventChannel "Sysmon"]
        Tag-Name=sysmon #Apply a new tag name
        Provider=Microsoft-Windows-Sysmon #Only look for the provider
        Channel=Microsoft-Windows-Sysmon/Operational
```

[Download the excellent sysmon configuration file by SwiftOnSecurity](https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml)

[Download sysmon](https://technet.microsoft.com/en-us/sysinternals/sysmon)

Put sysmon and that config in `C:\Program Files\gravwell`

In an admin powershell run:

```
sysmon.exe -accepteula -i sysmonconfig-export.xml
```

Restart the Gravwell service via standard windows service management.

#### Example Configuration with Sysmon

```
[EventChannel "system"]
        #no Tag-Name means use the default tag
        Tag-Name=windows
        #no Provider means accept from all providers
        #no EventID means accept all event ids
        #no Level means pull all levels
        #no Max-Reachback means look for logs starting from now
        Channel=System #pull from the system channel
[EventChannel "application"]
        #no Tag-Name means use the default tag
        Tag-Name=windows
        #no Provider means accept from all providers
        #no EventID means accept all event ids
        #no Level means pull all levels
        #no Max-Reachback means look for logs starting from now
        Channel=Application #pull from the system channel
[EventChannel "security"]
        #no Tag-Name means use the default tag
        Tag-Name=windows
        #no Provider means accept from all providers
        #no EventID means accept all event ids
        #no Level means pull all levels
        #no Max-Reachback means look for logs starting from now
        Channel=Security #pull from the system channel
[EventChannel "setup"]
        #no Tag-Name means use the default tag
        Tag-Name=windows
        #no Provider means accept from all providers
        #no EventID means accept all event ids
        #no Level means pull all levels
        #no Max-Reachback means look for logs starting from now
        Channel=Setup #pull from the system channel
[EventChannel "sysmon"]
        Tag-Name=windows
        Provider=Microsoft-Windows-Sysmon #Only look for the provider
        Channel=Microsoft-Windows-Sysmon/Operational
```

### Troubleshooting

You can verify the windows ingester connectivity by navigating to the Remote Ingester tag on the System Stats page via the webserver.  If the windows ingester is not present, check the status of the service either via the windows GUI or by issuing sc query GravwellEvents checking the status of the service.

![](querystatus.png)

![](querystatusgui.png)

### Example Windows Searches

Assuming the default windows tag names are used, to see ALL sysmon entries in their entirety run this search:

```
tag=sysmon
```

To see ALL windows events in their entirety run:

```
tag=windows
```

For the following searches, I took the Windows results and threw them in a regex validator [regex101.com](regex101.com) to build the regex. Anything you "name" with a `(<?P<foo>.*)` style regex is something you can chart by adding `| count by foo | chart count by foo`. See documentation about the search modules for more information on those.

To see all network creation by non-standard processes:

```
tag=sysmon regex ".*EventID>3.*'Image'>(?P<exe>\S*)<\/Data>.*SourceHostname'>(?P<src>\S*)<\/Data>.*DestinationIp'>(?P<dst>[0-9]+.[0-9]+.[0-9]+.[0-9]+).*DestinationPort'>(?P<dport>[0-9]*)"
```

To chart network creation by source host:

```
tag=sysmon regex ".*EventID>3.*'Image'>(?P<exe>\S*)<\/Data>.*SourceHostname'>(?P<src>\S*)<\/Data>.*DestinationIp'>(?P<dst>[0-9]+.[0-9]+.[0-9]+.[0-9]+).*DestinationPort'>(?P<dport>[0-9]*)" | count by src | chart count by src limit 10
```

To see suspicious file creation:

```
tag=sysmon regex ".*EventID>11.*Image'>(?P<process>.*)<\/Data>.*TargetFilename'>(?P<file>[\-:\.\ _\a-zA-z]*)<\/Data><Data Name='"
```
```
tag=sysmon regex ".*EventID>11.*Image'>(?P<process>.*)<\/Data>.*TargetFilename'>(?P<file>[\-:\.\ _\a-zA-z]*)<\/Data><Data Name='" | count by file | chart count by file
```

## Netflow Ingester

The Netflow ingester acts as a Netflow collector (see [the wikipedia article](https://en.wikipedia.org/wiki/NetFlow) for a full description of Netflow roles), gathering records created by Netflow exporters and capturing them as Gravwell entries for later analysis. These entries can then be analyzed using the [netflow](#!/search/netflow/netflow.md) search module.

To install the Netflow ingester, simply run the installer as root (the actual file name will typically include a version number):

```
root@gravserver ~ # bash gravwell_netflow_capture_installer.sh
```

If there is no Gravwell indexer on the local machine, the installer will prompt for an Ingest-Secret value and an IP address for an indexer (or a Federator). Otherwise, it will pull the appropriate values from the existing Gravwell configuration. In any case, review the configuration file in `/opt/gravwell/etc/netflow_capture.conf` after installation. A straightforward example which listens on UDP port 2055 might look like this:

```
[Global]
Ingest-Secret = IngestSecrets
Connection-Timeout = 0
Insecure-Skip-TLS-Verify=false
Pipe-Backend-target=/opt/gravwell/comms/pipe #a named pipe connection, this should be used when ingester is on the same machine as a backend
Log-Level=INFO

[Collector "netflow v5"]
	Bind-String="0.0.0.0:2055" #we are binding to all interfaces
	Tag-Name=netflow
```

Note that this configuration sends entries to a local indexer via `/opt/gravwell/comms/pipe`. Entries are tagged 'netflow'.

You can configure any number of `Collector` entries listening on different ports with different tags; this can help organize the data more clearly.

Note: At this time, the ingester only supports Netflow v5; keep this in mind when configuring Netflow exporters.

## Network Ingester

A primary strength of Gravwell is the ability to ingest binary data. The network ingester allows you to capture full packets from the network for later analysis; this provides much better flexibility than simply storing netflow or other condensed traffic information.

To install the network ingester, simply run the installer as root (the file name may differ slightly):

```
root@gravserver ~ # bash gravwell_network_capture_installer.sh
```

It is highly advised to co-locate the network ingester with an indexer and use a `pipe-conn` link to send data, rather than a `clear-conn` or `tls-conn` link.  If the network ingester is capturing from the same link it is using to push entries, a feedback loop is created which will rapidly saturate the link (e.g. capturing from eth0 while also sending entries to the ingester via eth0).

If the ingester is on a machine with a Gravwell backend already installed, the installer should automatically pick up the correct `Ingest-Secrets` value and populate the config file with it. In any case, review the configuration file in `/opt/gravwell/etc/network_capture.conf` before running. An example which captures traffic from eth0 might look like this:

```
[Global]
Ingest-Secret = IngestSecrets
Connection-Timeout = 0
Insecure-Skip-TLS-Verify = false
#Cleartext-Backend-target=127.1.0.1:4023 #example of adding a cleartext connection
#Encrypted-Backend-target=127.1.1.1:4023 #example of adding an encrypted connection
Pipe-Backend-target=/opt/gravwell/comms/pipe #a named pipe connection, this should be used when ingester is on the same machine as a backend
Log-Level=INFO #options are OFF INFO WARN ERROR
Ingest-Cache-Path=/opt/gravwell/cache/network_capture.cache

#basic default logger, all entries will go to the default tag
#no Tag-Name means use the default tag
[Sniffer "spy1"]
	Interface="eth0" #sniffing from interface eth0
	Tag-Name="pcap"  #assigning tag  fo pcap
	Snap-Len=0xffff  #maximum capture size
	#BPF-Filter="not port 4023" #do not sniff any traffic on our backend connection
	#Promisc=true
```

You can configure any number of `Sniffer` entries in order to capture from different interfaces.

If disk space is a concern, you may wish to change the `Snap-Len` parameter to capture only packet metadata. A value of 96 is usually sufficient to capture only headers.  Due to the potential for very high bandwidth links it is also advisable to assign the network capture data to its own Well.

The NetworkCapture ingester also supports native BPF filtering using the `BPF-Filter` parameter, which adheres to the libpcap syntax.  To ignore all traffic on port 22, one could configure a sniffer like this:

```
[Sniffer "no-ssh"]
	Interface="eth0"
	Tag-Name="pcap"
	Snap-Len=0xffff
	BPF-Filter="not port 22"
```

### Example Network Searches

The following search looks for tcp packets with the RST flag set which do not originate from the IP 10.0.0.0/24 class C subnet and then graphs them by IP.  This query can be used to rapidly identify outbound port scans from a network.

```
tag=pcap packet tcp.RST==TRUE ipv4.SrcIP !~ 10.0.0.0/24 | count by SrcIP | chart count by SrcIP limit 10
```

![](portscan.png)

The following search looks for IPv6 traffic and extracts the FlowLabel, which is passed on to a math operation.  This allows for the creation of per-flow traffic accounting by summing the lengths of packets and passing them into the chart renderer.

```
tag=pcap packet ipv6.Length ipv6.FlowLabel | sum Length by FlowLabel | chart sum by FlowLabel limit 10
```

To identify the languages in use in TCP payloads, filter network data and pass it to the langfind module.  This query is looking for outbound HTTP requests and handing the TCP payload data to the langfind module, which passes the identified languages to count and then chart.  This produces a chart of human languages used in outbound HTTP queries.

```
tag=pcap packet ipv4.DstIP != 10.0.0.100 tcp.DstPort == 80 tcp.Payload | langfind -e Payload | count by lang | chart count by lang
```

![](langfind.png)

Traffic accounting can also be performed at layer 2. This is accomplished by extracting the packet length from the ethernet header and summing the length by the destination MAC address and sorting by traffic count.  This allows us to rapidly identify physical devices on an Ethernet network that might be particularly chatty.

```
tag=pcap packet eth.DstMAC eth.Length > 0 | sum Length by DstMAC | sort by sum desc | table DstMAC sum
```

A similar query can identify chatty devices via packet counts as well.  For example, a device may be aggressively broadcasting small Ethernet packets which stress a switch but may not amount to large amounts of traffic.

```
tag=pcap packet eth.DstMAC eth.Length > 0 | count by DstMAC | sort by count desc | table DstMAC count
```

It may be desirable to identify HTTP traffic operating on non-standard HTTP ports.  This can be achieved by exercising the filtering options and passing payloads to other modules.  For example, looking for outbound traffic that is not TCP port 80 and is originating from a specific subnet and then passing the IP payload on to other modules, allows abnormal HTTP traffic to be identified.

```
tag=pcap packet ipv4.SrcIP ipv4.DstIP tcp.DstPort !=80 ipv4.SrcIP ~ 10.0.0.0/24 tcp.Payload | regex -e Payload "(?P<method>[A-Z]+)\s+(?P<url>[^\s]+)\s+HTTP/\d.\d" | table method url SrcIP DstIP DstPort
```

![](nonstandardhttp.png)

## Session Ingester

The session ingester is a specialized tool used to ingest larger, single records. The ingester listens on a given port and upon receiving a connection from a client it will aggregate any data received into a single records.

This enables behavior such as indexing all of your windows executable files:

```
for i in `ls /path/to/windows/exes`; do cat $i | nc 192.168.1.1 7777 ; done
```

The session ingester is driven via command line parameters rather than a persistent configuration file.

```
Usage of ./session:
  -bind string
        Bind string specifying optional IP and port to listen on (default "0.0.0.0:7777")
  -clear-conns string
        comma seperated server:port list of cleartext targets
  -ingest-secret string
        Ingest key (default "IngestSecrets")
  -max-session-mb int
        Maximum MBs a single session will accept (default 8)
  -pipe-conns string
        comma seperated list of paths for named pie connection
  -tag-name string
        Tag name for ingested data (default "default")
  -timeout int
        Connection timeout in seconds (default 1)
  -tls-conns string
        comma seperated server:port list of TLS connections
  -tls-private-key string
        Path to TLS private key
  -tls-public-key string
        Path to TLS public key
  -tls-remote-verify string
        Path to remote public key to verify against
```

### Notes

The session ingester is not formally supported, nor is there an installer available.  The source code for the session ingester is available on [github](https://github.com/gravwell/ingesters).

## Federator Ingester

Much like the Simple Relay ingester, the Federator is designed to build a series of IngestListeners which bind to a network and relay entries.  The Federator can act as a trust boundary, securely relaying entries across network segments without exposing ingest secrets or allowing untrusted nodes to send data for disallowed tags.  The Federator upstream connections are configured like any other ingester, allowing for multiplexing, local caching, encryption, etc.

![](federatorDiagram.png)

### Use Cases

 * Ingesting data across geographically diverse regions when there may not be robust connectivity
 * Providing an authentication barrier between network segments
 * Reducing the number of connections to an indexer
 * Controlling the tags an data source group can provide

### Example Configuration

The following example configuration connects to two upstream indexers in a protected network segment and provides ingest servers on two untrusted network segments.  Each untrusted ingest point has a unique Ingest-Secret, with one providing a TLS connect with a specific certificate and key pair.  The configuration file also enables a local cache, enabling the Federator to act as a fault tolerate buffer between the Gravwell indexers and the untrusted network segments.   By default the Federator ingester expects the configuration file to be located at /opt/gravwell/etc/federator.conf, but the location can be overriden using the -config flag.

```
[Global]
	Ingest-Secret = SuperSecretUpstreamIndexerSecret
	Connection-Timeout = 0
	Insecure-Skip-TLS-Verify = false
	Encrypted-Backend-target=172.20.232.105:4024
	Encrypted-Backend-target=172.20.232.106:4024
	Ingest-Cache-Path=/opt/gravwell/cache/federator.cache
	Max-Ingest-Cache=1024 #1GB
	Log-Level=INFO

[IngestListener "BusinessOps"]
        Ingest-Secret = CustomBusinessSecret
        Cleartext-Bind = 10.0.0.121:4023
        Tags=windows
        Tags=syslog

[IngestListener "DMZ"]
       Ingest-Secret = OtherRandomSecret
       TLS-Bind = 192.168.220.105:4024
       TLS-Certfile = /opt/gravwell/etc/cert.pem
       TLS-Keyfile = /opt/gravwell/etc/key.pem
       Tags=apache
       Tags=nginx
```

### Troubleshooting

Common configuration errors for the Federator include:

* Incorrect Ingest-Secret in the Global configuration
* Incorrect Backen-Target specification
* Invalid or already taken bind specifications
* Enforcing certification validation when upstream indexer or federators do not have certificates signed by a trusted certificate authority
* Mismatched Ingest-Secret for downstream ingesters

## Kinesis Ingester

Gravwell provides an ingester capable of fetching entries from Amazon's [Kinesis Data Streams](https://aws.amazon.com/kinesis/data-streams/) service. The ingester can process multiple Kinesis streams at a time, with each stream composed of many individual shards. The process of setting up a Kinesis stream is outside the scope of this document, but in order to configure the Kinesis ingester for an existing stream you will need:

* An AWS access key (ID number & secret key)
* The region in which your stream resides
* The name of the stream itself

Once the stream is configured, each record in the Kinesis stream will be stored as a single entry in Gravwell.

### Installation and configuration

First, install the ingester:

```
root@gravserver ~# bash gravwell_kinesis_ingest_installer.sh
```

If the Gravwell services are present on the same machine, the installation script should automatically extract and configure the `Ingest-Auth` parameter and set it appropriately. You will now need to open the `/opt/gravwell/etc/kinesis_ingest.conf` configuration file and set it up for your Kinesis stream. Once you have modified the configuration as described below, start the service with the command `systemctl start gravwell_kinesis_ingest.service`

The example below shows a sample configuration which connects to an indexer on the local machine (note the `Pipe-Backend-target` setting) and feeds it from a single Kinesis stream named "MyKinesisStreamName" in the us-west-1 region.

```
[Global]
Ingest-Secret = IngestSecrets
Connection-Timeout = 0
Insecure-Skip-TLS-Verify = false
Pipe-Backend-target=/opt/gravwell/comms/pipe #a named pipe connection, this should be used when ingester is on the same machine as a backend
Log-Level=ERROR #options are OFF INFO WARN ERROR

# This is the access key *ID* to access the AWS account
AWS-Access-Key-ID=REPLACEMEWITHYOURKEYID
# This is the secret key which is only displayed once, when the key is created
AWS-Secret-Access-Key=REPLACEMEWITHYOURKEY

[KinesisStream "stream1"]
	Region="us-west-1"
	Tag-Name=kinesis
	Stream-Name=MyKinesisStreamName	# should be the stream name as AWS knows it
	Iterator-Type=LATEST
	Parse-Time=false
	Assume-Localtime=true
```

You will need to set at least the following fields before starting the ingester:

* `AWS-Access-Key-ID` - this is the ID of the AWS access key you wish to use
* `AWS-Secret-Access-Key` - this is the secret access key itself
* `Region` - the region in which the kinesis stream resides
* `Stream-Name` - the name of the kinesis stream

You can configure multiple `KinesisStream` sections to support multiple different Kinesis streams.

You can test the config by running `/opt/gravwell/bin/gravwell_kinesis_ingester -v` by hand; if it does not print out errors, the configuration is probably acceptable.

Most of the fields are self-explanatory, but the `Iterator-Type` setting deserves a note. This setting selects where the ingester starts reading data. By setting it to TRIM_HORIZON, the ingester will start reading records from the oldest available. If it is set to LATEST, the ingester will ignore all existing records and only read records created after the ingester starts. In most situations, to avoid duplicating data it should be set to LATEST; set it TRIM_HORIZON if you have existing data you want to ingest, then shut down the ingester and change the value to LATEST before restarting.

## GCP PubSub Ingester

Gravwell provides an ingester capable of fetching entries from Google Compute Platform's [PubSub stream](https://cloud.google.com/pubsub/) service. The ingester can process multiple PubSub streams within a single GCP project. The process of setting up a PubSub stream is outside the scope of this document, but in order to configure the PubSub ingester for an existing stream you will need:

* The Google Project ID
* A file containing GCP service account credentials (see the [Creating a service account](https://cloud.google.com/docs/authentication/getting-started) documentation)
* The name of a PubSub topic

Once the stream is configured, each record in the PubSub stream topic will be stored as a single entry in Gravwell.

### Installation and configuration

First, install the ingester:

```
root@gravserver ~# bash gravwell_pubsub_ingest_installer.sh
```

If the Gravwell services are present on the same machine, the installation script should automatically extract and configure the `Ingest-Auth` parameter and set it appropriately. You will now need to open the `/opt/gravwell/etc/pubsub_ingest.conf` configuration file and set it up for your PubSub topic. Once you have modified the configuration as described below, start the service with the command `systemctl start gravwell_pubsub_ingest.service`

The example below shows a sample configuration which connects to an indexer on the local machine (note the `Pipe-Backend-target` setting) and feeds it from a single PubSub topic named "mytopic", which is part of the "myproject-127400" GCP project.

```
[Global]
Ingest-Secret = IngestSecrets
Connection-Timeout = 0
Insecure-Skip-TLS-Verify = false
Pipe-Backend-target=/opt/gravwell/comms/pipe #a named pipe connection, this should be used when ingester is on the same machine as a backend
Log-Level=ERROR #options are OFF INFO WARN ERROR

# The GCP project ID to use
Project-ID="myproject-127400"
Google-Credentials-Path=/opt/gravwell/etc/google-compute-credentials.json

[PubSub "gravwell"]
	Topic-Name=mytopic	# the pubsub topic you want to ingest
	Tag-Name=gcp
	Parse-Time=false
	Assume-Localtime=true
```

Note the following essential fields:

* `Project-ID` - the Project ID string for a GCP project
* `Google-Credentials-Path` - the path to a file containing GCP service account credentials in JSON format
* `Topic-Name` - the name of a PubSub topic within the specified GCP project

You can configure multiple `PubSub` sections to support multiple different PubSub topics within a single GCP project.

You can test the config by running `/opt/gravwell/bin/gravwell_pubsub_ingester -v` by hand; if it does not print out errors, the configuration is probably acceptable.

## Disk Monitor

The diskmonitor ingester is designed to take periodic samples of disk activity and ship the samples to gravwell.  The disk monitor is extremely useful in identifying storage latency issues, looming disk failures, and potential storage failures.  We at Gravwell actively monitor our own storage infrastructure with the disk monitor to identify how queries are operating and to identify when the storage infrastructure is behaving badly.  We were able to identify a RAID array that transitioned to write-through mode via a latency plot even when the RAID controller failed to mention it in the diagnostic logs.

The disk monitor ingeser is available on [github](https://github.com/gravwell/ingesters)

![diskmonitor](diskmonitor.png)

## Ingest API

The Gravwell ingest API and core ingesters are fully open source under the BSD 2-Clause license.  This means that you can write your own ingesters and integrate Gravwell entry generation into your own products and services.  The core ingest API is written in Go, but the list of available API languages is under active expansion.

[API code](https://github.com/gravwell/ingest)

[API documentation](https://godoc.org/github.com/gravwell/ingest)

A very basic ingester example (less than 100 lines of code) that tails a file and transports entries to a Gravwell cluster [can be seen here](https://www.godoc.org/github.com/gravwell/ingest#example-package)

Keep checking back with the Gravwell Github page, as the team is continually improving the ingest API and porting it to additional languages. Community development is fully supported, so if you have a merge request, language port, or a great new ingester that you have open sourced, let Gravwell know!  The Gravwell team would love to feature your hard work in the ingestor highlight series.