# GeoDNS in Go

[English README](./READM-EN.md)

[NTP Pool](http://www.pool.ntp.org/)系统和一些类似的服务都用架构在这个DNS基础上. 它是[pgeodns](http://github.com/abh/pgeodns) 的替代者。
[![Build Status](https://drone.io/github.com/abh/geodns/status.png)](https://drone.io/github.com/abh/geodns/latest)

据我所知道的这个DNS的作用：根据地域来解析，可以给不同的解析地址加权重。

## 安装

1. 一些前提

> 以下命令行是在Amazon Linux AMI release 2017.09上执行的。

* [go](http://code.google.com/p/go/downloads/list)
* gcc: `yum install gcc`
* geoip： `yum install geoip geoip-devel`

2. 安装编译

```sh
export PATH=$PATH:/usr/local/go/bin
export GOPATH=~/go
go get github.com/abh/geodns
cd ~/go/src/github.com/abh/geodns
go test
go build
./geodns -h
```

3. 当时遇到两个问题：

```
# pkg-config --cflags geoip
Package geoip was not found in the pkg-config search path.
Perhaps you should add the directory containing `geoip.pc'
to the PKG_CONFIG_PATH environment variable
No package 'geoip' found
```

做了如下操作，添加了一个文件：

```
cd /usr/lib64/pkgconfig/
cat geoip.pc 

prefix=/usr
libdir=/usr/lib64
includedir=/usr/include
datadir=${prefix}/share

Name: geoip
Description: A non-DNS IP-to-country resolver library.
Version: 1.4.8
Libs: -L${libdir} -lGeoIP
Cflags: -I${includedir}/
databasedir=${datadir}/GeoIP
```

还有就是关于geo ip的数据不存在的情况，去[官网](http://dev.maxmind.com/geoip/legacy/geolite/)上下载即可，下载后文件放的位置在`dns/geodns.conf`里指定。

## 简单配置

配置示例： `dns/example.com.json`. 用于单元测试，不是最佳实践的示例。

如下是一个更大的测试文件：

```sh
mkdir -p dns
curl -o dns/test.ntppool.org.json http://tmp.askask.com/2012/08/dns/ntppool.org.json.big
```

## 运行DNS server

构建好后，照如下方式运行，127.0.0.1是网络接口地址：

`./geodns -log -interface 127.0.0.1 -port 5053`

在另一个窗口测试：

`dig -t a test.example.com @127.0.0.1 -p 5053`

或方向解析：

`dig -t ptr 2.1.168.192.IN-ADDR.ARPA. @127.0.0.1 -p 5053`

或简单的：

`dig -x 192.168.1.2 @127.0.0.1 -p 5053`


### 命令选项说明

注意：`=`后面是默认配置，可以不写

* `-config="./dns/"`: 指定zone files的位置，默认配置文件`geodns.conf`
* `-checkconfig=false`: 检查配置，包括zone file，检查完退出，默认`false`
* `-interface="*"`： DNS服务监听的IP，默认`*`，可以用逗号分隔，多个IP
* `-port="53"`：监听端口 (UDP和TCP)
* `-http=":8053"`：HTTP接口监听的端口. `127.0.0.1:8053`就是只监听`localhost`
* `-identifier=""`： 实例的标识符，可以是`hostname, pop name`或者类似的。也可以是逗号分隔的一个列表，"server id"开头，然后是"group names", 例如服务器的地区, 服务器所在的anycast集群的名称等。这是将来`reporting/statistics`特征将会用到的。
* `-log=false`: 日志打印，默认false。测试和debug的时候打开。生产上不推荐，除非请求量很小(<1-200/s).
* `-cpus=1`: 使用的最大CPU数量. 设为0就用所有的。默认1是经过检验的，推荐。

## WebSocket接口

`8053`端口上有接口`/monitor`： There's a "companion program" that can
use this across a cluster to show aggregate statistics, email for more information.

## 运行状态接口

 `:8053/status`: queries per second, queries and
most frequently requested labels per zone, etc

## StatHat集成

GeoDNS可post数据到 [StatHat](http://www.stathat.com/).详见
([文档](https://github.com/abh/geodns/wiki/StatHat))

## 国家和州的查找

见后文。zone targeting options.

## 记录值的权重

Most records can have a 'weight' assigned. If any records of a particular type
for a particular name have a weight, the system will return `max_hosts` records
(default 2).

If the weight for all records is 0, all matching records will be returned. The
weight for a label can be any integer as long as the weights for a label and record
type is less than 2 billion.

As an example, if you configure

    10.0.0.1, weight 10
    10.0.0.2, weight 20
    10.0.0.3, weight 30
    10.0.0.4, weight 40

with `max_hosts` 2 then .4 will be returned about 4 times more often than .1.

## 配置文件

The geodns.conf file allows you to specify a specific directory for the GeoIP
data files and other options. See the `geodns.conf.sample` file for example
configuration.

The global configuration file is not reloaded at runtime.

Most of the configuration is "per zone" and done in the zone .json files.
The zone configuration files are automatically reloaded when they change.

## Zone的格式

In the zone configuration file the whole zone is a big hash (associative array).
At the top level you can (optionally) set some options with the keys serial,
ttl and max_hosts.

The actual zone data (dns records) is in a hash under the key "data". The keys
in the hash are hostnames and the value for each hostname is yet another hash
where the keys are record types (lowercase) and the values an array of records.

For example to setup an MX record at the zone apex and then have a different
A record for users in Europe than anywhere else, use:

    {
        "serial": 1,
        "data": {
            "": {
                "ns": [ "ns.example.net", "ns2.example.net" ],
                "txt": "Example zone",
                "spf": [ { "spf": "v=spf1 ~all", "weight": 1 } ],
                "mx": { "mx": "mail.example.com", "preference": 10 }
            },
            "mail": { "a": [ ["192.168.0.1", 100], ["192.168.10.1", 50] ] },
            "mail.europe": { "a": [ ["192.168.255.1", 0] ] },
            "smtp": { "alias": "mail" }
        }
    }

The configuration files are automatically reloaded when they're updated. If a file
can't be read (invalid JSON, for example) the previous configuration for that zone
will be kept.

## Zone选项

* serial

GeoDNS doesn't support zone transfers (AXFR), so the serial number is only used
for debugging and monitoring. The default is the 'last modified' timestamp of
the zone file.

* ttl

Set the default TTL for the zone (default 120).

* targeting

* max_hosts



* contact

Set the soa 'contact' field (default is "hostmaster.$domain").

## Zone targeting options

@

country
continent

region and regiongroup

## Supported record types

Each label has a hash (object/associative array) of record data, the keys are the type.
The supported types and their options are listed below.

Adding support for more record types is relatively straight forward, please open a
ticket in the issue tracker with what you are missing.

### A

Each record has the format of a short array with the first element being the
IP address and the second the weight.

    [ [ "192.168.0.1", 10], ["192.168.2.1", 5] ]

See above for how the weights work.

### AAAA

Same format as A records (except the record type is "aaaa").

### Alias

Internally resolved cname, of sorts. Only works internally in a zone.

    "foo"

### CNAME

    "target.example.com."
    "www"

The target will have the current zone name appended if it's not a FQDN (since v2.2.0).

### MX

MX records support a `weight` similar to A records to indicate how often the particular
record should be returned.

The `preference` is the MX record preference returned to the client.

    { "mx": "foo.example.com" }
    { "mx": "foo.example.com", "weight": 100 }
    { "mx": "foo.example.com", "weight": 100, "preference": 10 }

`weight` and `preference` are optional.

### NS

NS records for the label, use it on the top level empty label (`""`) to specify
the nameservers for the domain.

    [ "ns1.example.com", "ns2.example.com" ]

There's an alternate legacy syntax that has space for glue records (IPv4 addresses),
but in GeoDNS the values in the object are ignored so the list syntax above is
recommended.

    { "ns1.example.net.": null, "ns2.example.net.": null }

### TXT

Simple syntax

    "Some text"

Or with weights

    { "txt": "Some text", "weight": 10 }

### SPF

An SPF record is semantically identical to a TXT record with the exception that the label is set to 'spf'. An example of an spf record with weights:


    { "spf": "v=spf1 ~all]", "weight": 1 }

An spf record is typically at the root of a zone, and a label can have an array of SPF records, e.g

      "spf": [ { "spf": "v=spf1 ~all", "weight": 1 } , "spf": "v=spf1 10.0.0.1", "weight": 100]

### SRV

An SRV record has four components: the weight, priority, port and target. The keys for these are "srv_weight", "priority", "target" and "port". Note the difference between srv_weight (the weight key for the SRV qtype) and "weight".

An example srv record definition for the _sip._tcp service:

    "_sip._tcp": {
        "srv": [ { "port": 5060, "srv_weight": 100, "priority": 10, "target": "sipserver.example.com."} ]
    },

Much like MX records, SRV records can have multiple targets, eg:

    "_http._tcp": {
        "srv": [
            { "port": 80, "srv_weight": 10, "priority": 10, "target": "www.example.com."},
            { "port": 8080, "srv_weight": 10, "priority": 20, "target": "www2.example.com."}
        ]
    },

## 许可和版权

This software is Copyright 2012-2015 Ask Bjørn Hansen. For licensing information
please see the file called LICENSE.
