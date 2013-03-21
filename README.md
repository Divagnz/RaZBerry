ZWave Serial API Sniffing Journal
========================

These are my notes About ZWave mostly compiled with RaZerry. 

RaZberry
========

A RaZberry hardware solution is a combination of the [Raspberry Pi] [2] motherboard and the [RaZberry] [2] Z-Wave transceiver daughter board. The daughter board is connected to the mother-board using the General IO Pin header connector of Raspberry PI. This GPIO interface offers Serial TX and RX signals, ground and 3.3 V VCC to power the Z-Wave transceiver board. 

The RaZberry uses the [ZM3102][1] Z-Wave transceiver from [SIGMA DESIGNS] [4]. This module combines a "System on Chip" (SOC) with a 8051 micro controller, the Z-Wave transceiver and some IO interfaces the systems crystal and the SAW antenna filter.


The micro controller of the SOC contains control code that operates the wireless transceiver and handles certain network level operations of Z-Wave. The communication with this code runs over the serial interface. There is a protocol specification for this interface that is issued by the
Manufacturer of the Z-Wave chip Sigma Designs that most of the [Z-Wave transceivers] [5] on the market (e.g., USB Sticks) use. This interface specification — called Sigma Designs Serial API - is not a public document but available under Non Disclosure Agreement only as part of the Sigma Designs Systems Development Kit (SDK). The firmware of RaZberry is based on the SDK Version 4.54 but has enhanced the Sigma Designs Serial API in several ways.


ZWave
=======

Thanks to the folks over at the OpenZwave project I found out that the basis of ZWave is an ITU standard. A little Googling and it appears the standard of interest is ITU-T G.9959 (http://www.itu.int/rec/T-REC-G.9959/en).

Have I mentioned that I love the project Open-ZWave?


Thoughts on Application Design
==============================

I have written my fair share of Wireshark dissectors but I have always had Wireshark to do the display of those captures. 

Creating a ZWave sniffer will require the dissector but it will also require some user interface. This may be best split into to project. Similar to TShark (or tcpdump) that does the capture and the user interface application. Initially the UI will just dump to the console or to a file.

or

Maybe I can sort out someway to save the capture in PCAP format and still use Wireshark to decode it?


Future Goals
===============

I would like to make a ZWave decoder that will monitor the serial line and output human readable information about what is being sent.


ZWave Frames
========

_Basic ZWave Frame_

|Byte Position: |0|1|3|4|5|6|7|8|9|10|
|---|---|---|---|---|---|---|---|---|---|---|
| **Field:** |SOF|Length|Request/Response|ZWave Function (see ZWave Functions below)|


ZWave Sample Decodes
=====================

The following data is sent/receive on the RaZberry board.

This is some scratch data to try and manually decode what the binary packets actually mean.

_Table: First frame sent from controller_

|index|Direction|Value|Decode Information|
|---|---|---|---|
|1 |TX|0x01| SOF (see Preambles below)|
|2 |TX|0x03| Length |
|3 |TX|0x00| 0x00-Request|
|4 |TX|0x07| SerialGetCapabilities |
|5 |TX|0xfb| Checksum - see _Generating a checksum_ below|

_Table: Response_

|index|Direction|Value|Decode Information|
|---|---|---|---|---|
| |RX|0x06| ACK (see Preambles below)| |
| |RX|0x01| SOF (see Preambles below)| |
| |RX|0x2b| Length 43 Bytes | |
| |RX|0x01| 0x01-Response | |
| |RX|0x07| SerialGetCapabilities| |
| |RX|0x04| Version | |
| |RX|0x02| Revision | |
| |RX|0x01| Manufacture ID1 | |
| |RX|0x47| Manufacture ID1 | |
| |RX|0x00| Product Type 1| |
| |RX|0x02| Product Type 2| |
| |RX|0x00| Product ID 1| |
| |RX|0x03| Product ID 2| |
| |RX|0xfe| | |
| |RX|0x00| | |
| |RX|0x16| | |
| |RX|0x80| | |
| |RX|0x0c| | |
| |RX|0x00| | |
| |RX|0x00| | |
| |RX|0x00| | |
| |RX|0xe3| | |
| |RX|0x97| | |
| |RX|0x7d| | |
| |RX|0x80| | |
| |RX|0x07| | |
| |RX|0x00| | |
| |RX|0x00| | |
| |RX|0x80| | |
| |RX|0x00| | |
| |RX|0x00| | |
| |RX|0x00| | |
| |RX|0x00| | |
| |RX|0x00| | |
| |RX|0x00| | |
| |RX|0x00| | |
| |RX|0x00| | |
| |RX|0x00| | |
| |RX|0x00| | |
| |RX|0x02| | |
| |RX|0x00| | |
| |RX|0x00| | |
| |RX|0x80| | |
| |RX|0x07| | |
| |RX|0x00| | |
| |RX|0x7c| Checksum| | 


_Table: (Switch Binary Set)_

|index|Direction|Value|Decode Information|
|---|---|---|---|
|1  |TX|0x01| SOF (see Preambles below)|
|2  |TX|0x0a| Length |
|3  |TX|0x00| 0x00-Request|
|4  |TX|0x13| SendData |
|5  |TX|0x09| Node ID|
|6  |TX|0x03| 3-BinarySet (2-BinaryGet) |
|7  |TX|0x25| BINARY_SWITCH|
|8  |TX|0x01| Set value? (is this [SET, GET, REPORT]?)|
|9  |TX|0x00| 0x00-Switch off (0xff=ON) |
|10 |TX|0x25| |
|11 |TX|0x03| |
|12 |TX|0xee| Checksum - see _Generating a checksum_ below|

_Table: Random decodes_

|nodeId|  |  | |  |Description|
|---|---|---|---|---|---|
|18|2|30|2|5| Sensor Binary Get|
|9|3|25|1|ff 25| BINARY_SWITCH Set|
|9|2|25|2|25| BINARY_SWITCH Get|
|9|3|25|1|0 25| BINARY_SWITCH Set|
|9|2|25|2|25| BINARY_SWITCH Get|
|9|3|25|1|0 25| BINARY_SWITCH Set|
|9|2|25|2|25| BINARY_SWITCH Get|



*Generating a checksum*

```JAVA
    private static byte generateChecksum(byte[] dataFrame) {
        int offset = 0;
        byte ret = data[offset];
        for (int i = offset; i < data.length; i++) {
            // Xor bytes
            ret ^= data[i];
        }
        ret = (byte) (~ret);
        return ret;
    }
    

Usage:
    byte[] zwaveFrame = new byte[] {0x01, 0x03, 0x20, /* (byte) 0xdc, */};
    System.out.println("==============> Checksum: 0x" + Integer.toHexString(EIMApplication.generateChecksum(zwaveFrame)));


```

Device Classes
===========

To allow inter-operability between different Z-Wave devices from different manufacturers, each device must include certain well-defined functions above and beyond the ‘Basic’ command class.

These requirements are called ‘Device Classes’. A device class refers to a typical device and defines which command classes that are mandatory for it to support.

Device classes are organized into a three-layer hierarchy:

Every device must belong to a basic device class
Devices can be further specified by assigning them to a generic device class
Further functionality can be defined by assigning the device to a specific device class
Basic Device Class
The ‘Basic’ device class simply defines a device as a Controller, Slave or Routing Slave. Therefore every device belongs to one basic device class.

Generic Device Class
The ‘Generic’ device class defines the basic functionality that the devices will support as a controller or slave. Current ‘Generic’ device classes are:

    General controller (GENERIC_CONTROLLER)
    Static controller (STATIC_CONTROLLER)
    Binary switch (BINARY_SWITCH)
    Multi level switch (MULTI_LEVEL_SWITCH)
    Binary sensor (BINARY_SENSOR)
    Multilevel-Sensor (MULTILEVEL_SENSOR)
    Meter (METER)
    Input controller (ENTRY_CONTROL)
    Thermostat (THERMOSTAT)
    Window Blind controller (WINDOW_COVERING)


_Specific Device Class_

Assigning a ‘Specific’ device class to a Z-Wave device allows it to further specify its functionality. Each ‘Generic’ device class refers to a number of specific device classes. You can decide to assign a specific device class, however, it only makes sense if the device really supports all functions of a ‘Specific’ device class.

_Table: **ZWave Command Classes**_

|Name|Hex|Dec|
|---|---|---|
|NO OPERATION|0x00|0|
|BASIC|0x20|32|
|CONTROLLER REPLICATION|0x21|33|
|APPLICATION STATUS|0x22|34|
|ZIP SERVICES|0x23|35|
|ZIP SERVER|0x24|36|
|SWITCH BINARY|0x25|37|
|SWITCH MULTILEVEL|0x26|38|
|SWITCH MULTILEVEL V2|0x26|38|
|SWITCH ALL|0x27|39|
|SWITCH TOGGLE BINARY|0x28|40|
|SWITCH TOGGLE MULTILEVEL|0x29|41|
|CHIMNEY FAN|0x2A|42|
|SCENE ACTIVATION|0x2B|43|
|SCENE ACTUATOR CONF|0x2C|44|
|SCENE CONTROLLER CONF|0x2D|45|
|ZIP CLIENT|0x2E|46|
|ZIP ADV SERVICES|0x2F|47|
|SENSOR BINARY|0x30|48|
|SENSOR MULTILEVEL|0x31|49|
|SENSOR MULTILEVEL V2|0x31|49|
|METER|0x32|50|
|ZIP ADV SERVER|0x33|51|
|ZIP ADV CLIENT|0x34|52|
|METER PULSE|0x35|53|
|METER TBL CONFIG|0x3C|60|
|METER TBL MONITOR|0x3D|61|
|METER TBL PUSH|0x3E|62|
|THERMOSTAT HEATING|0x38|56|
|THERMOSTAT MODE|0x40|64|
|THERMOSTAT OPERATING STATE|0x42|66|
|THERMOSTAT SETPOINT|0x43|67|
|THERMOSTAT FAN MODE|0x44|68|
|THERMOSTAT FAN STATE|0x45|69|
|CLIMATE CONTROL SCHEDULE|0x46|70|
|THERMOSTAT SETBACK|0x47|71|
|COMMAND CLASS DOOR LOCK LOGGING|0x4C|76|
|SCHEDULE ENTRY LOCK|0x4E|78|
|BASIC WINDOW COVERING|0x50|80|
|MTP WINDOW COVERING|0x51|81|
|MULTI CHANNEL V2|0x60|96|
|MULTI INSTANCE|0x60|96|
|DOOR LOCK|0x62|98|
|USER CODE|0x63|99|
|CONFIGURATION|0x70|112|
|CONFIGURATION V2|0x70|112|
|ALARM|0x71|113|
|MANUFACTURER SPECIFIC|0x72|114|
|POWERLEVEL|0x73|115|
|PROTECTION|0x75|117|
|PROTECTION V2|0x75|117|
|LOCK|0x76|118|
|NODE NAMING|0x77|119|
|FIRMWARE UPDATE MD|0x7A|122|
|GROUPING NAME|0x7B|123|
|REMOTE ASSOCIATION ACTIVATE|0x7C|124|
|REMOTE ASSOCIATION|0x7D|125|
|BATTERY|0x80|128|
|CLOCK|0x81|129|
|HAIL|0x82|130|
|WAKE UP|0x84|132|
|WAKE UP V2|0x84|132|
|ASSOCIATION|0x85|133|
|ASSOCIATION V2|0x85|133|
|VERSION|0x86|134|
|INDICATOR|0x87|135|
|PROPRIETARY|0x88|136|
|LANGUAGE|0x89|137|
|TIME|0x8A|138|
|TIME PARAMETERS|0x8B|139|
|GEOGRAPHIC LOCATION|0x8C|140|
|COMPOSITE|0x8D|141|
|MULTI CHANNEL ASSOCIATION V2|0x8E|142|
|MULTI INSTANCE ASSOCIATION|0x8E|142|
|MULTI CMD|0x8F|143|
|ENERGY PRODUCTION|0x90|144|
|MANUFACTURER PROPRIETARY|0x91|145|
|SCREEN MD|0x92|146|
|SCREEN MD V2|0x92|146|
|SCREEN ATTRIBUTES|0x93|147|
|SCREEN ATTRIBUTES V2|0x93|147|
|SIMPLE AV CONTROL|0x94|148|
|AV CONTENT DIRECTORY MD|0x95|149|
|AV RENDERER STATUS|0x96|150|
|AV CONTENT SEARCH MD|0x97|151|
|SECURITY|0x98|152|
|AV TAGGING MD|0x99|153|
|IP CONFIGURATION|0x9A|154|
|ASSOCIATION COMMAND CONFIGURATION|0x9B|155|
|SENSOR ALARM|0x9C|156|
|SILENCE ALARM|0x9D|157|
|SENSOR CONFIGURATION|0x9E|158|
|MARK|0xEF|239|
|NON INTEROPERABLE|0xF0|240



Node Definitions
=================

```XML
<?xml version="1.0" encoding="utf-8"?>
<DeviceClasses>
  <Basic key="0x01" label="Controller" />
  <Basic key="0x02" label="Static Controller" />
  <Basic key="0x03" label="Slave" />
  <Basic key="0x04" label="Routing Slave" />
  <Generic key="0x01" label="Remote Controller" command_classes="0xef,0x20">
    <Specific key="0x01" label="Portable Remote Controller" />
    <Specific key="0x02" label="Portable Scene Controller" command_classes="0x2d,0x72,0x85,0xef,0x2b" />
    <Specific key="0x03" label="Portable Installer Tool" command_classes="0x21,0x72,0x86,0x8f,0xef,0x21,0x60,0x70,0x72,0x84,0x85,0x86,0x8e" />
  </Generic>
  <Generic key="0x02" label="Static Controller" command_classes="0xef,0x20">
    <Specific key="0x01" label="Static PC Controller" />
    <Specific key="0x02" label="Static Scene Controller" command_classes="0x2d,0x72,0x85,0xef,0x2b" />
    <Specific key="0x03" label="Static Installer Tool" command_classes="0x21,0x72,0x86,0x8f,0xef,0x21,0x60,0x70,0x72,0x84,0x85,0x86,0x8e" />
  </Generic>
  <Generic key="0x03" label="AV Control Point" command_classes="0x20">
    <Specific key="0x04" label="Satellite Receiver" command_classes="0x72,0x86,0x94" />
    <Specific key="0x11" label="Satellite Receiver V2" command_classes="0x72,0x86,0x94" basic="0x94" />
    <Specific key="0x12" label="Doorbell" command_classes="0x30,0x72,0x85,0x86" basic="0x30"/>
  </Generic>
  <Generic key="0x04" label="Display" command_classes="0x20">
    <Specific key="0x01" label="Simple Display" command_classes="0x72,0x86,0x92,0x93" />
  </Generic>
  <Generic key="0x08" label="Thermostat" command_classes="0x20">
    <Specific key="0x01" label="Heating Thermostat" />
    <Specific key="0x02" label="General Thermostat" command_classes="0x40,0x43,0x72" basic="0x40" />
    <Specific key="0x03" label="Setback Schedule Thermostat" command_classes="0x46,0x72,0x86,0x8f,0xef,0x46,0x81,0x8f" basic="0x46" />
    <Specific key="0x04" label="Setpoint Thermostat" command_classes="0x43,0x72,0x86,0x8f,0xef,0x43,0x8f" basic="0x43" />
    <Specific key="0x05" label="Setback Thermostat" command_classes="0x40,0x43,0x47,0x72,0x86" basic="0x40" />
    <Specific key="0x06" label="General Thermostat V2" command_classes="0x40,0x43,0x72,0x86" basic="0x40" />
  </Generic>
  <Generic key="0x09" label="Window Covering" command_classes="0x20">
    <Specific key="0x01" label="Simple Window Covering" command_classes="0x50" basic="0x50" />
  </Generic>
  <Generic key="0x0f" label="Repeater Slave" command_classes="0x20">
    <Specific key="0x01" label="Basic Repeater Slave" />
  </Generic>
  <Generic key="0x10" label="Binary Switch" command_classes="0x20,0x25" basic="0x25">
    <Specific key="0x01" label="Binary Power Switch" command_classes="0x27" />
    <Specific key="0x03" label="Binary Scene Switch" command_classes="0x27,0x2b,0x2c,0x72" />
  </Generic>
  <Generic key="0x11" label="Multilevel Switch" command_classes="0x20,0x26" basic="0x26">
    <Specific key="0x01" label="Multilevel Power Switch" command_classes="0x27" />
    <Specific key="0x03" label="Multiposition Motor" command_classes="0x72,0x86" />
    <Specific key="0x04" label="Multilevel Scene Switch" command_classes="0x27,0x2b,0x2c,0x72" />
    <Specific key="0x05" label="Motor Control Class A" command_classes="0x25,0x72,0x86" />
    <Specific key="0x06" label="Motor Control Class B" command_classes="0x25,0x72,0x86" />
    <Specific key="0x07" label="Motor Control Class C" command_classes="0x25,0x72,0x86" />
  </Generic>
  <Generic key="0x12" label="Remote Switch" command_classes="0xef,0x20">
    <Specific key="0x01" label="Binary Remote Switch" command_classes="0xef,0x25" basic="0x25"/>
    <Specific key="0x02" label="Multilevel Remote Switch" command_classes="0xef,0x26" basic="0x26"/>
    <Specific key="0x03" label="Binary Toggle Remote Switch" command_classes="0xef,0x28" basic="0x28"/>
    <Specific key="0x04" label="Multilevel Toggle Remote Switch" command_classes="0xef,0x29" basic="0x29"/>
  </Generic>
  <Generic key="0x13" label="Toggle Switch" command_classes="0x20" >
    <Specific key="0x01" label="Binary Toggle Switch" command_classes="0x25,0x28" basic="0x28" />
    <Specific key="0x02" label="Multilevel Toggle Switch" command_classes="0x26,0x29" basic="0x29" />
  </Generic>
  <Generic key="0x14" label="Z/IP Gateway" command_classes="0x20">
    <Specific key="0x01" label="Z/IP Tunneling Gateway" command_classes="0x23,0x24,0x72,0x86"/>
    <Specific key="0x02" label="Z/IP Advanced Gateway" command_classes="0x23,0x24,0x2f,0x33,0x72,0x86"/>
  </Generic>
  <Generic key="0x15" label="Z/IP Node">
    <Specific key="0x01" label="Z/IP Tunneling Node" command_classes="0x23,0x2e,0x72,0x86" />
    <Specific key="0x02" label="Z/IP Advanced Node" command_classes="0x23,0x2e,0x2f,0x34,0x72,0x86" />
  </Generic>
  <Generic key="0x16" label="Ventilation" command_classes="0x20">
    <Specific key="0x01" label="Residential Heat Recovery Ventilation" command_classes="0x37,0x39,0x72,0x86" basic="0x39"/>
  </Generic>
  <Generic key="0x20" label="Binary Sensor" command_classes="0x30,0xef,0x20" basic="0x30">
    <Specific key="0x01" label="Routing Binary Sensor" />
  </Generic>
  <Generic key="0x21" label="Multilevel Sensor" command_classes="0x31,0xef,0x20" basic="0x31">
    <Specific key="0x01" label="Routing Multilevel Sensor" />
  </Generic>
  <Generic key="0x30" label="Pulse Meter" command_classes="0x35,0xef,0x20" basic="0x35"/>
  <Generic key="0x31" label="Meter" command_classes="0xef,0x20">
    <Specific key="0x01" label="Simple Meter" command_classes="0x32,0x72,0x86" basic="0x32" />
  </Generic>
  <Generic key="0x40" label="Entry Control" command_classes="0x20">
    <Specific key="0x01" label="Door Lock" command_classes="0x62" basic="0x62"/>
    <Specific key="0x02" label="Advanced Door Lock" command_classes="0x62,0x72,0x86" basic="0x62"/>
    <Specific key="0x03" label="Secure Keypad Door Lock" command_classes="0x62,0x63,0x72,0x86,0x98" basic="0x62"/>
  </Generic>
  <Generic key="0x50" label="Semi Interoperable" command_classes="0x20,0x72,0x86,0x88">
    <Specific key="0x01" label="Energy Production" command_classes="0x90" />
  </Generic>
  <Generic key="0xa1" label="Alarm Sensor" command_classes="0xef,0x20" basic="0x71">
    <Specific key="0x01" label="Basic Routing Alarm Sensor" command_classes="0x71,0x72,0x85,0x86,0xef,0x71" />
    <Specific key="0x02" label="Routing Alarm Sensor" command_classes="0x71,0x72,0x80,0x85,0x86,0xef,0x71" />
    <Specific key="0x03" label="Basic Zensor Alarm Sensor" command_classes="0x71,0x72,0x86,0xef,0x71" />
    <Specific key="0x04" label="Zensor Alarm Sensor" command_classes="0x71,0x72,0x80,0x86,0xef,0x71" />
    <Specific key="0x05" label="Advanced Zensor Alarm Sensor" command_classes="0x71,0x72,0x80,0x85,0x86,0xef,0x71" />
    <Specific key="0x06" label="Basic Routing Smoke Sensor" command_classes="0x71,0x72,0x85,0x86,0xef,0x71" />
    <Specific key="0x07" label="Routing Smoke Sensor" command_classes="0x71,0x72,0x80,0x85,0x86,0xef,0x71" />
    <Specific key="0x08" label="Basic Zensor Smoke Sensor" command_classes="0x71,0x72,0x86,0xef,0x71" />
    <Specific key="0x09" label="Zensor Smoke Sensor" command_classes="0x71,0x72,0x80,0x86,0xef,0x71" />
    <Specific key="0x0a" label="Advanced Zensor Smoke Sensor" command_classes="0x71,0x72,0x80,0x85,0x86,0xef,0x71" />
  </Generic>
  <Generic key="0xff" label="Non Interoperable" />
</DeviceClasses>
```

Device Class


ZWave Functions
===============

|Name|Value|
|---|---|
|None|0x00|
|DiscoveryNodes|0x02|
|SerialApiApplNodeInformation|0x03|
|ApplicationCommandHandler|0x04|
|GetControllerCapabilities|0x05|
|SerialApiSetTimeouts|0x06|
|SerialGetCapabilities|0x07|
|SerialApiSoftReset|0x08|
|SetRFReceiveMode|0x10|
|SetSleepMode|0x11|
|SendNodeInformation|0x12|
|SendData|0x13|
|SendDataMulti|0x14|
|GetVersion|0x15|
|SendDataAbort|0x16|
|RFPowerLevelSet|0x17|
|SendDataMeta|0x18|
|MemoryGetId|0x20|
|MemoryGetByte|0x21|
|MemoryPutByte|0x22|
|MemoryGetBuffer|0x23|
|MemoryPutBuffer|0x24|
|ReadMemory|0x23|
|ClockSet|0x30|
|ClockGet|0x31|
|ClockCompare|0x32|
|RtcTimerCreate|0x33|
|RtcTimerRead|0x34|
|RtcTimerDelete|0x35|
|RtcTimerCall|0x36|
|GetNodeProtocolInfo|0x41|
|SetDefault|0x42|
|ReplicationCommandComplete|0x44|
|ReplicationSendData|0x45|
|AssignReturnRoute|0x46|
|DeleteReturnRoute|0x47|
|RequestNodeNeighborUpdate|0x48|
|ApplicationUpdate|0x49|
|AddNodeToNetwork|0x4a|
|RemoveNodeFromNetwork|0x4b|
|CreateNewPrimary|0x4c|
|ControllerChange|0x4d|
|SetLearnMode|0x50|
|AssignSucReturnRoute|0x51|
|EnableSuc|0x52|
|RequestNetworkUpdate|0x53|
|SetSucNodeId|0x54|
|DeleteSucReturnRoute|0x55|
|GetSucNodeId|0x56|
|SendSucId|0x57|
|RediscoveryNeeded|0x59|
|RequestNodeInfo|0x60|
|RemoveFailedNodeId|0x61|
|IsFailedNode|0x62|
|ReplaceFailedNode|0x63|
|TimerStart|0x70|
|TimerRestart|0x71|
|TimerCancel|0x72|
|TimerCall|0x73|
|GetRoutingTableLine|0x80|
|GetTXCounter|0x81|
|ResetTXCounter|0x82|
|StoreNodeInfo|0x83|
|StoreHomeId|0x84|
|LockRouteResponse|0x90|
|SendDataRouteDemo|0x91|
|SerialApiTest|0x95|
|SerialApiSlaveNodeInfo|0xa0|
|ApplicationSlaveCommandHandler|0xa1|
|SendSlaveNodeInfo|0xa2|
|SendSlaveData|0xa3|
|SetSlaveLearnMode|0xa4|
|GetVirtualNodes|0xa5|
|IsVirtualNode|0xa6|
|SetPromiscuousMode|0xd0


_Table: Preambles used see "man ascii"_

|Name|Value|Description|
|---|---|---|
|SOF|0x01|Start Of Frame |
|ACK|0x06|Message Ack|
|NAK|0x15|Message NAK|
|CAN|0x18|Cancel - Resend request|


Capturing serial port data
=========

Install "interceptty".

* http://www.suspectclass.com/sgifford/interceptty/
* unpack
* build it and install the tool

```BASH

# Usage:
# Stop Z-Way process "sudo /etc/init.d/Z-Way stop"
#  > sudo vi /opt/z-way-server/config.xml
# change "/dev/ttyAMA0" to "/tmp/ttyAMA0"
# Run this script
# Start Z-Way "sudo /etc/init.d/Z-Way start"

interceptty -s 'ispeed 115200 ospeed 115200' /dev/ttyAMA0 /tmp/ttyAMA0

```

Once you are done sniffing the serial port you will need to stop the Z-Way server and change "config.xml" back to it's original form.



JSON Server
========

Z-Way makes a server for Raspberry PI and RaZberry. It seems nice but I don't really have much interest in it. I am a little anti Node.js which is what so many ZWave things seem to be based on?


Terms
========

Term | Description
-----|--------------------
FLIRS | Frequently Listening Devices
NIF | Node Information Frame
SIS | Static ID-Server
SOF | Start Of Frame
SUC | Static Update Controller



More ZWave References
==========
* linuxmce ZWave API - http://wiki.linuxmce.org/index.php/ZWave_API
* Catching the Z-Wave - http://www.drdobbs.com/embedded-systems/catching-the-z-wave/193104353
* Open ZWave - https://code.google.com/p/open-zwave/
* An introduction to Z-Wave programming in C# - http://www.digiwave.dk/en/programming/an-introduction-to-z-wave-programming-in-c/
* http://www.vesternet.com/resources/technology-indepth/how-z-wave-controllers-work
* http://www.codecoretechnologies.com/community/index.php?topic=946.20
* ViziaRF - https://code.google.com/p/zwave-driver-for-premise/source/browse/trunk/ViziaRF/ViziaRF.xdo
* "razberry.pdf" - http://razberry.z-wave.me/docs/razberry.pdf
* http://wiki.micasaverde.com/index.php/ZWave_Command_Classes
* ZWave protocol version - http://wiki.micasaverde.com/index.php/ZWave_Protocol_Version
* Aeonz Stick Driver - https://bitbucket.org/bradsjm/aeonzstickdriver/src/befa5117e290?at=default
* ZWave device DB (far from complete) - http://www.pepper1.net/zwavedb/

[1]: http://www.digikey.com/us/en/ph/SigmaDesigns/z-wave_zm3102.html        "ZM3102"
[2]: http://www.raspberrypi.org/ "Raspberry Pi"
[3]: http://razberry.zwave.me/ "RaZberry"
[4]: http://www.sigmadesigns.com/ "SIGMA DESIGNS"
[5]: http://www.itu.int/rec/T-REC-G.9959/en "Z-Wave transceivers"

