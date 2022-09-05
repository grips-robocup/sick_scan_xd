# MultiScan136/sick_scan_segment_xd

The MultiScan136 Beta is a new lidar from Sick with a total of 16 lidar units rotating around a vertical axis. 
The rotation speed is 20 rounds per second.
Scan data are transmitted in msgpack format over UDP.

MultiScan136 / sick_scan_segment_xd lidars are supported by sick_scan_xd. See [README](../README.md) for build and run instructions.

The following describes the configuration, validation and test in more detail.


## Configuration

MultiScan136/sick_scan_segment_xd is configured by launch file [sick_scansegment_xd.launch](../launch/sick_scansegment_xd.launch).

Modify file [sick_scansegment_xd.launch](../launch/sick_scansegment_xd.launch) to change configuration. Note that the ip address of the udp receiver __must__ be configured on each system. This is the ip address of the computer running sick_scan_xd.

The ip address of the lidar and the udp receiver can be configured in the launch file by e.g.
```
<arg name="hostname" default="192.168.0.1"/>
<arg name="udp_receiver_ip" default="192.168.0.100"/>
```
or by command line by e.g.
```
hostname:=192.168.0.1 udp_receiver_ip:=192.168.0.100
```

## SOPAS support

On ROS-1 and ROS-2, service `ColaMsg` is provided to send CoLa commands to the lidar. Using this service, filters can be applied during runtime. 

See [sick_scansegment_xd_sopas_examples.md](sick_scansegment_xd_sopas_examples.md) for examples.

See the manual for further information of filter settings and parameter. 


The driver sends the following SOPAS start and stop sequence at program start resp. exit (example with default ip address 192.168.0.1):
```
// Prerequirement: measurement is active, but no UDP data is sent
// Start sending scan data output
sMN SetAccessMode   3 F4724744   // set authorization level for writing settings
sWN ScanDataEthSettings 1 +192 +168 +0 +1 +2115   // configure destination scan data output destination to 192.168.0.52 port 2115
sWN ScanDataFormatSettings 1 1   // set scan data output format to MSGPACK
sWN ScanDataEnable 1   // enable scan data ouput
sMN LMCstartmeas // start measurement
sMN Run   // apply the settings and logout
// ...
// UDP data is sent
// ...
// Stop sending scan data output
sMN SetAccessMode   3 F4724744   // set authorization level for writing settings
sWN ScanDataEnable 0   // disable scan data output
sMN Run   // apply the settings and logout
// No UDP data is sent anymore      
```

## Visualization

The Multiscan136 scans can be visualized by rviz. The following screenshots show two examples of a Multiscan136 pointcloud:

![msgpacks-emulator-rviz](20210929-tokenized-msgpacks-emulator-rviz.png)
![msgpacks-emulator-rviz](20210929-tokenized-msgpacks-mrs100-rviz.png)

Note that sick_scan_xd publishes 2 pointclouds:
* The pointcloud on topic `/cloud` is published for each scan segment.
* The pointcloud on topic `/cloud_360` collects all segments for a complete 360 degree full scan.

## Msgpack validation

A msgpack validation can be activated. This validation checks
1. each incoming msgpack for scan data out of the expected values, and
2. missing scandata after collecting the msgpack data for a full scan (360 degree)

If a msgpack contains scan data out of expected values, the msgpack is discarded and an error message is printed. This should not happen in normal operation mode. If scan data are missing after a full 360 degree scan, an error message is printed. This might happen in case of udp packet drops.

By default, the full range of scan data is expected, i.e. all echos, all segments, all layers and azimuth values covering -180 up to +180 degree. If filters are activated (echo-, layer- or angle-range-filter to reduce network traffic), the msgpack validation should currently be deactivated or configured thoroughly to avoid error messages. In the next release, the filter configuration is queried from  MultiScan136 Beta and validation settings are adopted to the MultiScan136 Beta filter settings.

The msgpack validation is configured in file [sick_scansegment_xd.launch](../launch/sick_scansegment_xd.launch). To activate or deactivate msgpack validation, set `msgpack_validator_enabled` to True (activated) resp. False (deactivated). 

Msgpack validation leads to error messages in case of udp packet drops. Increase the value `msgpack_validator_check_missing_scandata_interval` to tolerate udp packet drops. Higher values increase the number of msgpacks collected for verification.

## Firewall configuration

By default, UDP communication is allowed on localhosts. To enable udp communication between 2 different machines, firewalls have to be configured.

On Windows: Setup the windows firewall to allow sick_scan to receive udp packages on port 2115.
To pass udp packages from a remote sender, the default rule for incoming udp packages has to be configured in the windows firewall:
1. Run "wf.msc" as admin,
2. Click Inbound Rules and locate the rule(s) for lidar3d_msr100_recv (resp. python to allow python test scripts), and
3. Deactivate the UDP-rule for this process(es) or configure exceptions for remote computers.
4. Alternatively, you can create a new rule allowing udp communication on port 2115.

On Linux: Run the following commands to allow any udp communication on port 2115:
```
sudo iptables -A INPUT -p udp -m udp --dport 2115 -j ACCEPT
sudo iptables -A OUTPUT -p udp -m udp --sport 2115 -j ACCEPT
sudo iptables-save
```
Alternatively, you can also use
```
sudo ufw allow from any to any port 2115 proto udp
```
to allow all udp traffic on port 2115.

Note: If Linux or Windows is running in a virtual machine, make sure UDP port 2115 is forwarded. With VMware Workstation Pro, you can configure port forwarding 
using the Virtual Network Editor. Udp echos, delays, drops and other unexpected errors might occure when more than one network card is configured in VMware. 
Make sure you have only one network adapter activated with custom NAT:
![vmware_network_settings](vmware_network_settings.png)

## FAQ

### Visual Studio: Breakpoints in Debug Mode disabled

:question: In Windows debug version the compiler does not stop at breakpoints.

:white_check_mark: Check, that you are using the Debug Version. At '/Zi' to compiler settings. Disable optimization.
(see `https://stackoverflow.com/questions/865546/generating-symbols-in-release-binaries-with-visual-studio` for details).

### Packages lost in benchmark 

:question: sick_scan_xd seems to drop packages, when sending 10000 msgpacks with polarscan_sender_test.py from another computer

:white_check_mark: There can be a number of reasons for dropped messages (udp or msgpacks). Besides slow network connection, there can be other pitfalls depending on the system:

- If Linux or Windows is running in a virtual machine, make sure UDP port 2115 is forwarded. See [Firewall configuration](#firewall__configuration).

- Depending on ROS2 system settings, log messages might be buffered. To really see all log messages of sick_generic_caller, terminate sick_scan_xd/sick_generic_caller (Ctrl-C or kill) and view the ros logfile by `cat ~/.ros/log/sick_scan_*.log`