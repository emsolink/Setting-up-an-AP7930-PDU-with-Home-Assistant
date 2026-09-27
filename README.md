# Setting-up-an-AP7930-PDU-with-Home-Assistant
Setting up an AP7930 PDU with Home Assistant


Setting up an AP7930 PDU with Home Assistant
Jul 8, 2022
Overview
I have a workbench in my office for working on various projects. There are more devices than the current regular power strip has outlets. For this reason, I wanted to mount a power strip to the wall behind the workbench. After looking at various options, I came up with the idea to use a used power distribution unit (PDU) and found a switched one that was a decent price. With it being a switched PDU I can remotely turn outlets on/off and monitor the load across all outlets. This is definitely overkill but a fun project.

Selecting a PDU
There are several things to look at when finding a PDU. The first is the input voltage and whether it is single phase or three phase. If you are in North America and don’t have three phase like most people then you will want something that is 120v and single phase. I found that some listings don’t mention this information so you should double check the model number with the manufacturers data sheet. Another thing to take note of is the type of plug. The AP7930 I found and most other single phase PDUs appear to use a NEMA l5-20P plug. This means that it is rated for 20 amps and uses a locking plug that can’t be accidentally be pulled out. This is not the standard outlet in most houses. You can either replace the receptacle with something like this one and a corresponding wall plate or get an adapter. It’s important to note that it is not code to put a 20 amp receptacle on a 15 amp circuit. Consult an electrician if you are not comfortable. In my case I had already run a new wire as the existing wire was only rated at 15 amps and did not have a ground wire.

Another thing to look at is the supported features. There are two features you will want to look for: metered and switched. Metered will let you monitor the load on the PDU. Note that most of the time this is for all outlets and not monitored per outlet. However, there are options that have metering per outlet. A switched PDU will let you remotely control the outlet and turn it on/off. Each of these features comes with its own power consumption overhead that you may want to take into account. For APC they list out the power consumption for different models here https://www.apc.com/us/en/faqs/FA156173/. In my case the idle power consumption of several devices I wanted to automate turning off was more than the consumption of the PDU itself, so it made sense.

Performing a Factory Reset
The AP7930 has one of the most convoluted reset procedures I have come across. The reset button by itself is not enough to perform a factory reset. You also need a serial cable model 940-0144 to then connect to it and reset it. I tried creating a cable with the listed pinout for this cable with the default baud rate, along with several other rates. However, I was unable to communicate with the device. I found an alternative method using the default SNMP string, which was not changed by the previous owner, as described on this site https://robbat2.livejournal.com/240492.html. There were a couple changes and things not listed so the process I used is as follows.

1 - Look at your DHCP leases and determine if the device was assigned an IP. If not, connect to the same network that the AP7930 is plugged into. Then run Wireshark with a filter for the MAC address listed on the back of the PDU. You should see the preconfigured IP address. In this case it is 10.66.154.244. Manually set your IP address and subnet appropriately to communicate with the device.Wireshark ScreenNote that if you can access the web interface for the device in the browser with the default username and password of apc you can skip the following steps

2 - Install the SNMP MIBs

Terminal window
sudo apt-get install snmp-mibs-downloader
sudo sed -i 's/mibs :/# mibs :/g' /etc/snmp/snmp.conf
sudo wget "https://fossies.org/linux/misc/netxms-4.1.377.tar.gz/netxms-4.1.377/contrib/mibs/PowerNet-MIB.txt?m=b" -O /usr/share/snmp/mibs/PowerNet-MIB.txt

3 - Verify that the default SNMP string is set by running the following command.

Terminal window
snmpget -v 1 -c private $IP SNMPv2-MIB::sysDescr.0

You should get back the version running. If not, you may need to attempt to get the serial reset method working or extract the SNMP string from the I2C chip.

4 - Setup a TFTP server and host the following file as config.ini.

[NetworkTCP/IP]
SystemIP = 0.0.0.0
SubnetMask = 0.0.0.0
DefaultGateway = 0.0.0.0
Bootp = enabled
[NetworkTFTPClient]
RemoteIP = 0.0.0.0
[NetworkFTPClient]
RemoteIP = 0.0.0.0
RemoteUserName = apc
RemotePassword = apc
[NetworkFTPServer]
Access = enabled
Port = 21
[NetworkTelnet]
Access = enabled
Port = 23
[NetworkWeb]
Access = enabled
Port = 80
[NetworkSNMP]
Access = enabled
AccessControl1Community = public
AccessControl1NMSIP = 0.0.0.0
AccessControl1AccessType = read
AccessControl2Community = private
AccessControl2NMSIP = 0.0.0.0
AccessControl2AccessType = write
[NetworkDNS]
DNSServerIP = 0.0.0.0
[SystemID]
Name = Unknown
Contact = Unknown
Location = Unknown
[SystemDate/Time]
Date = 01/01/2014
Time = 12:00:00
[SystemUserManager]
Authentication = Basic
AutoLogout = 10
AdminUserName = apc
AdminPassword = apc
AdminAuthPhrase = admin user phrase
DeviceUserName = device
DevicePassword = apc
DeviceAuthPhrase = device user phrase

5 - Run the following commands to have the device download and apply the config. Note that it may take a while and need to be run a couple times.

Terminal window
snmpset -v 1 -c private $DEVICEIP PowerNet-MIB::mfiletransferConfigTFTPServerAddress.0 s $SERVERIP
snmpset -v 1 -c private $DEVICEIP PowerNet-MIB::mfiletransferConfigSettingsFilename.0 s /config.ini
snmpset -v 1 -c private $DEVICEIP PowerNet-MIB::mfiletransferControlInitiateFileTransfer.0 i initiateFileTransferDownloadViaTFTP

6 - If it worked correctly, it should reset and you can access it by DHCP with the default username and password apc. Reset all the settings by going under Administration - General - Reset/Reboot, click Reset All, and Apply.

Updating the Firmware
The device will likely not be running the latest firmware. It took a little while to find but the latest version can be downloaded from here https://www.apc.com/us/en/product/SFRPDU374_390/ap7xxx-switched-metered-rack-power-distribution-unit-firmware-revision-3-7-4-aos-3-9-0/. Once it is downloaded, extract it and run the executable. It will ask for the IP address of the device along with the username and password. The rest of the update process will be handled by the program. If there are any issues you can also update the device by logging in over ftp and uploading the rpdu and aos bin files after switching to binary mode.

Configuring the Device
Note that the security support on these devices is lacking. You can enable HTTPS, SSH, SNMPv3 but the ciphers it uses are outdated and not supported by modern clients. The easiest option is to change the default passwords and then segment it off from any untrusted devices. Under Administration - Security - Local Users, change the password for the administrator user and then uncheck access enabled for the device and read-only account. For Home Assistant to control the device we will use SNMPv1, SNMPv3 would be preferred but I couldn’t get it working with the default SNMP integration. Make sure SNMPv1 is enabled by going to Administration - Network - SNMPv1 - access and checking Enable SNMPv1 access. Set the SNMPv1 community strings by going to access control right under the current page link. Click the first community name and change it to a secure string and change Access Type to write. Then go and change the other community names access type to disabled.

Configuring Home Assistant
Home Assistant has an integration for SNMP built in. You can find more configuration information here https://www.home-assistant.io/integrations/snmp/. You will need to modify your configuration.yaml and add the following sections. If you have either an existing switch or sensor section you can just append it to the existing section. To add other outlets just copy the switch entry and modify the name and the last number in the baseoid, which is the outlet number. The community string should be changed to the SNMPv1 write string you set and the host is the IP of the PDU.

switch:
  - platform: snmp
    name: PDU Outlet 1
    host: your_pdu_IP
    community: your_community_string
    baseoid: 1.3.6.1.4.1.318.1.1.4.4.2.1.3.1
    payload_on: 1
    payload_off: 2
  - platform: snmp
    name: PDU Outlet 2
    host: your_pdu_IP
    community: your_community_string
    baseoid: 1.3.6.1.4.1.318.1.1.4.4.2.1.3.2
    payload_on: 1
    payload_off: 2

sensor:
  - platform: snmp
    name: PDU Load
    host: your_pdu_IP
    community: your_community_string
    baseoid: 1.3.6.1.4.1.318.1.1.12.2.3.1.1.2.1
    unit_of_measurement: A
    value_template: "{{((value | float) / 10) | float}}"

Once the configuration is done, restart Home Assistant and everything hopefully should work. You now can create automations to control the PDU, manually switch outlets on/off, etc.Home Assistant Screen
