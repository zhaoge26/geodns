# GeoDNS in Go

[English README](./README-EN.md)

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

多数的记录会有分配一个权重值（weight）。如果某一特定类型的任何记录特定名称有权重，则系统会返回`max_hosts`个记录值，默认是2。

如果所有记录的的权重值都为0，所有匹配的记录都将返回。一个标签的权重值可以是任何整数，小于20亿。（The
weight for a label can be any integer as long as the weights for a label and record
type is less than 2 billion.）

一个例子

```
    10.0.0.1, weight 10
    10.0.0.2, weight 20
    10.0.0.3, weight 30
    10.0.0.4, weight 40
```

如果`max_hosts`是2，则返回.4的次数将会是返回.1的4倍。

## 配置文件

`geodns.conf`：可以配置一个特定的目录用来存放GeoIP的数据，还有其他一些选项。可以参考 `geodns.conf.sample`

注意：这个文件的修改不会自动加载，需要重启。(The global configuration file is not reloaded at runtime.)

多数的配置文件是关于zone的配置，更改后会自动reload。

## Zone的格式

zone配置文件里，整个zone是一个大的hash (key-value格式，关联数组？associative array). 在最顶层，可选的设置一些选项，如`keys serial, ttl and max_hosts`。

键`data`里是真正的zone数据（dns记录），也是一个hash。hash里的这些键是`hostname`，每一个`hostname`同时又是另一个hash，他们的键是记录(record)类型（小写），值是数组。

如下面的列子，在zone里设置一个MX记录，对应的A记录里，给欧洲用户设置了不同的值：

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

更新后，该配置会自动重载。一个文件不可读，如不正确的JSON，那么配置将不会被更改，依然是之前的配置。

## Zone选项

* serial：GeoDNS不支持zone传输（AXFR），所以serial只是用于debug和监控。默认是zone文件的最后更改时间，timestamp格式。
* ttl：zone的ttl (默认120s).
* targeting: 
* max_hosts:
* contact: 参见soa 'contact'  (默认是"hostmaster.$domain").

## Zone targeting options

@
country continent
region and regiongroup

## 支持的记录类型

每个标签有一个记录值，hash格式(object/associative array)，键是类型。支持的类型和它们的选项如后所列。

如果你需要增加更多的记录类型，请在issue tracker里提交。

### A

格式如下：数组，第一个值是IP，第二个值是权重。

    [ [ "192.168.0.1", 10], ["192.168.2.1", 5] ]

### AAAA

类似A记录，记录类型是"aaaa"。

### Alias

只在zone内部使用，用来解析cname

    "foo"

### CNAME

    "target.example.com."
    "www"

从v2.2.0开始，如果不是FQDN，则CNAME将会加上当前的zone名。

### MX

类似A记录，MX记录也支持用`weight`指明权重，来表明一个记录的返回频率。

`preference` 是MX记录返回给客户的偏好值。

`weight`和 `preference`是可选的。

    { "mx": "foo.example.com" }
    { "mx": "foo.example.com", "weight": 100 }
    { "mx": "foo.example.com", "weight": 100, "preference": 10 }

### NS

NS记录，在zone配置的顶层，标签是空(`""`)，用来指定该域的域名服务器。

    [ "ns1.example.com", "ns2.example.com" ]

There's an alternate legacy syntax that has space for glue records (IPv4 addresses),
but in GeoDNS the values in the object are ignored so the list syntax above is
recommended.
如下是一个老语法，不推荐。

    { "ns1.example.net.": null, "ns2.example.net.": null }

### TXT

基本语法:

    "Some text"

也可以带权重值：

    { "txt": "Some text", "weight": 10 }

### SPF

SPF记录，语义上同TXT记录。

一个例子，带有权重值：

    { "spf": "v=spf1 ~all]", "weight": 1 }

spf记录是在zone的root里，可以是任意值，数组格式。

      "spf": [ { "spf": "v=spf1 ~all", "weight": 1 } , "spf": "v=spf1 10.0.0.1", "weight": 100]

### SRV

SRV记录由4部分组成：权重，优先级，端口和目标，键名是"srv_weight", "priority", "target", "port"。注意srv_weight和weight的不同。

下面是一个`_sip._tcp`服务的srv记录定义：

    "_sip._tcp": {
        "srv": [ { "port": 5060, "srv_weight": 100, "priority": 10, "target": "sipserver.example.com."} ]
    },

类似MX，SRV记录也可以用多个目标：

    "_http._tcp": {
        "srv": [
            { "port": 80, "srv_weight": 10, "priority": 10, "target": "www.example.com."},
            { "port": 8080, "srv_weight": 10, "priority": 20, "target": "www2.example.com."}
        ]
    },

## 许可和版权

This software is Copyright 2012-2015 Ask Bjørn Hansen. For licensing information
please see the file called LICENSE.
