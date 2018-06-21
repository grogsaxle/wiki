# Downloads

## Ingesters

The core suite of ingesters are available for download as an installable packages.  Ingesters designed to operate on Linux machines are typically self contained, statically linked executables that are agnostic to the hosts package management system (with the exception of the NetworkCapture ingester).  Windows based ingesters are distributed as executable MSI packages.  Source code for many ingesters can be found at [Github](https://github.com/gravwell/ingesters).


### Version 2.0.7 Ingester Releases
| Ingester | Description | MD5 | More Info |
|:--------:|-------------|:---:|----------:|
| [Simple Relay](#!ingesters/ingesters.md#Simple_Relay) | An ingester capable of accepting syslog or line brokend data sent over the network. |0e245d040e4cba86030aabe0e0e27dad| [Download](https://update.gravwell.io/files/gravwell_simple_relay_installer_2.0.7.tar.bz2)|
| [File Follower](#!ingesters/ingesters.md#File_Follower) | The standard file following ingester designed to look for line broken log entries in files.  Useful for ingesting logs from systems that can only log to files. |d7b354e212e47c38847eb689dae7a0b5| [Download](https://update.gravwell.io/files/gravwell_file_follow_installer_2.0.7.tar.bz2) |
| [Netflow Capture](#!ingesters/ingesters.md#Netflow_Ingester) | The Netflow Capture ingester acts as a Netflow v5 collector, ingesting Netflow records as Gravwell entries. |2fd9af8f848588b8285b7dee534f693f| [Download](http://update.gravwell.io/files/gravwell_netflow_capture_installer_2.0.7.tar.bz2) |
| [Network Capture](#!ingesters/ingesters.md#Network_Ingester) | The Network Capture ingester is a passive network sniffing ingester which can bind to multiple network taps and send raw network traffic to Gravwell. |4bf9f42ac064b49f7010f8ca55d8226f| [Download](https://update.gravwell.io/files/gravwell_network_capture_installer_2.0.7.tar.bz2) |
| [Ingest Federator](#!ingesters/ingesters.md#Federator_Ingester) | The Federator ingester is designed to aggregate multiple downstream ingesters and relay entries to upstream ingestion points.  The Federator is useful for crossing trust boundaries, aggregating entry flows, and insulating Gravwell indexers from potentially untrusted downstream entry generators. |ec6dc45a97f4d8aa7d324a76262f601b| [Download](https://update.gravwell.io/files/gravwell_federator_installer_2.0.7.tar.bz2) |
| [Windows Events](#!ingesters/ingesters.md#Windows_Event_Service) | The Winevent uses the Windows events subsystem to acquire windows events and ship them to gravwell.  The Winevent ingester can be placed on a single Windows machine acting as a log collector, or on multiple endpoints. | a23ef46fe93855e480b44feb4f9fec57 | [Download](https://update.gravwell.io/files/gravwell_win_events_2.0.7.msi) |
| [Windows File Follower](#!ingesters/ingesters.md#File_Follower) | The Windows file follower is identical to the File Follower ingester, but for Windows. | 29343609b96dda3552ab010c83dea663 | [Download](https://update.gravwell.io/files/gravwell_file_follow_2.0.7.msi) |
| [Amazon Kinesis](#!ingesters/ingesters.md#Kinesis_Ingester) | The Amazon Web Services Kinesis ingester can attach to the Kinesis stream and dramatically simplify logging a cloud deployment |aec4f6ee047f0e5e7207dfe671768355| [Download](https://update.gravwell.io/files/gravwell_kinesis_ingest_installer_2.0.7.tar.bz2)|
| [Google PubSub](#!ingesters/ingesters.md#GCP_PubSub) | The Google Cloud Platform PubSub Ingester can subscribe to exhausts on the GCP PubSub system, easing integration with GCP. |5a6c65d88e095229c0f0ae8a98365fbe| [Download](https://update.gravwell.io/files/gravwell_pubsub_ingest_installer_2.0.7.tar.bz2)|

[//]: <> (| [](#!ingesters/ingesters.md#) | | | [Download](https://update.gravwell.io/files/) |)
[//]: <> (| [](#!ingesters/ingesters.md#) | | | [Download](https://update.gravwell.io/files/) |)

## Other downloads

Some Gravwell components are distributed as optional additional installers, such as the search agent and the datastore.

| Component | Description | MD5 | More Info |
|:---------:|-------------|:---:|----------:|
| [Datastore](#!distributed/frontend.md) | The datastore keeps multiple Gravwell webservers in sync, enabling load balancing |e2c5dec008623e1359ae57678baf1136| [Download](https://update.gravwell.io/files/gravwell_datastore_installer_2.0.7.tar.bz2) |
| [Search Agent](#!scripting/scheduledsearch.md) | The search agent runs user-defined searches and scripts on a specified schedule. |ea6d45614a61c8eac215bb78fe15ab2b| [Download](https://update.gravwell.io/files/gravwell_searchagent_installer_2.0.7.tar.bz2) |