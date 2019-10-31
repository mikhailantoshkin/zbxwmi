# zbxwmi
Zabbix WMI connector - discover and retrieve info from MS Windows hosts without installing agent.

Originaly developed by [13hakta](https://github.com/13hakta/zbxwmi).

Original code had several rarely used features and overcomplicated flags, so some refactorisation was done. Also support for several credentials configuration was added



Depends on python library [impacket](https://github.com/CoreSecurity/impacket).

Operates in 4 modes:
* Discover objects
* Get multiple values in JSON

## Command line parameters

```
usage: zbxwmi [-h] [-action {get,json,discover}] [-namespace NAMESPACE]
              -fields FIELDS [-filter FILTER] [-server SERVER]
              [-sender SENDER] [-host HOST] [-dc-ip ip address]
              [-rpc-auth-level [{integrity,privacy,default}]]
              cls cred target

Zabbix WMI connector v0.1.1

positional arguments:
  cls                   <WMI Class>
  cred                  <Credential file>
  target                <targetName or address>

optional arguments:
  -h, --help            show this help message and exit
  -action {get,json,discover}
                        The action to take
  -namespace NAMESPACE  namespace name (default //./root/cimv2)
  -fields FIELDS        Field list delimited by comma
  -filter FILTER        Filter
  -server SERVER        Zabbix server
  -sender SENDER        Zabbix sender
  -host HOST            Hostname specified in cred config

authentication:
  -dc-ip ip address     IP Address of the domain controller. If ommited it use
                        the domain part (FQDN) specified in the target
                        parameter
  -rpc-auth-level [{integrity,privacy,default}]
                        default, integrity (RPC_C_AUTHN_LEVEL_PKT_INTEGRITY)
                        or privacy (RPC_C_AUTHN_LEVEL_PKT_PRIVACY). For
                        example CIM path "root/MSCluster" would require
                        privacy level by default)
```

Credential file structure:
```
[<zabbix_hostname>]
username = <username>
password = <password>
domain = <domain>
```
were `<zabbix_hostname>` is the hostname on Zabbix server returned by `{HOST.HOST}` macro.

Scrip supports use of several credentials configurations. If `<zabbix_hostname>` given in `-host` is not found in credentials file,
then values in `[DEFAULT]` are used.

## Installation

Assume you installed Zabbix Appliance with Ubuntu onboard. Access root shell and install appropriate dependencies.

Put `zbxwmi` script to `/usr/lib/zabbix/externalscripts` and set permissions:

```sh
# cd /usr/lib/zabbix/externalscripts
# chmod 755 zbxwmi
# chown root.root zbxwmi
```

Install required python modules:

```sh
# apt install python3-six python3-pycryptodome python3-pyasn1
```

Install [impacket](https://github.com/CoreSecurity/impacket) library.

Download from github or [stripped down](https://13hakta.ru/assets/components/fileattach/connector.php?action=web/download&ctx=web&fid=MDK5dMZwyEHoTNkHGkamjLSs7fIpRXTh) version sufficient to perform WMI calls.
Unpack contents to any directory and run `pip3 install` on it: 
```sh
# pip3 install /tmp/impacket-master
```

Create file /etc/zabbix/wmi.pw with structure, described above. Set file access:

```sh
# chmod 640 /etc/zabbix/wmi.pw
# chown zabbix.zabbix /etc/zabbix/wmi.pw
```

## Examples

Receive disc I/O performance

```sh
$ zbxwmi \
-a json \
-host "DEFAULT" \
-fields "DiskWritesPersec,DiskWriteBytesPersec,DiskReadsPersec,DiskReadBytesPersec" \
-filter "Name='_Total'" \
"Win32_PerfRawData_PerfDisk_LogicalDisk" \
"wmi.pw" \
"127.0.0.1"
```

Outputs:
`[{"DiskReadBytesPersec": 9845850112, "DiskReadsPersec": 485564, "DiskWriteBytesPersec": 7573131264, "DiskWritesPersec": 334575}]`

### Discover objects:

Discover local drive partitions

```sh
$ zbxwmi \
 -action discover \
 -host "DEFAULT" \
 -fields "DeviceID" \
 -filter MediaType=12 \
 "Win32_LogicalDisk" \
 "wmi.pw" \
 "127.0.0.1"
```
Outputs kind of:
`{"data": [{"{#WMI.DEVICEID}": "C:"}]}`

### Usage in zabbix items

Get processor load:

`zbxwmi["-host","{HOST.HOST}","-action","json","-fields","PercentProcessorTime","-filter","Name<>'_Total'","Win32_PerfFormattedData_PerfOS_Processor","{$WMI_AUTHFILE}",{HOST.CONN}]`

Get disk I/O load:

`zbxwmi["-host","{HOST.HOST}","-action","json","-fields","DiskWritesPersec,DiskWriteBytesPersec,DiskReadsPersec,DiskReadBytesPersec","-filter","Name='_Total'","Win32_PerfRawData_PerfDisk_LogicalDisk","{$WMI_AUTHFILE}",{HOST.CONN}]`


## Zabbix usage

* Create template
* If your credential file located not in /etc/zabbix/wmi.pw, then set template macro `{$WMI_AUTHFILE}` = `/path/to/wmi.pw`
* Create discovery rule with external check script
* Create discrovery item prototypes
  * Create main item to receive multiple values
  * Create dependent items with JSON preprocessing
* Create graph prototype
* Optionally create trigger
* Assign template to MS Windows hosts
