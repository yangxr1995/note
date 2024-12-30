# 模组拨号上网
## 复位模组

```shell
# 关机
at+cfun=0
# 开机
at+cfun=0
```

## 检查SIM卡
```shell
at+cpin?
```
apn：5glan
频点小区：152650，123

## 确定能搜索到运营商
```shell
# 设置显示格式为
at+cops=3,2
# 扫描运营商
at+cops=?
# 自动选择运营商，并注册
at+cops=0
# 查看当前已注册的运营商
at+cops?
```

## 查看注册状态
```shell
at+cgreg?

+CREG: 0,1

OK
```
根据返回的结果 +CREG: 0,1，可以解读如下：

第一个参数 0 表示注册模式（禁止主动上报）。
第二个参数 1 表示注册状态，1 表示已注册到本地网络。
因此，+CREG: 0,1 表示注册成功。你的设备已经成功注册到网络。

## 设置apn等

```shell
at-mngr 'at+cgdcont=1,"Ethernet","data1"'
或
at-mngr  AT+CGDCONT=1,\"IP\",\"cmnet\";
at-mngr at+cgauth=1,0,\"any\",\"any\" 2>/dev/null
```

## 拨号
```shell
AT+GTRNDIS=1,1
```

## 请求ip
```shell
busybox udhcpc -b -t 5 -i pcie0 -p /var/mobile/dhcpd
```
# 接入技术和锁频
## 接入技术名称
- 3g
  - UMTS
  - WCDMA
- 4g
  - LTE
- 5g
  - NR-RAN

## at指令

`AT+GTACT=[<rat>[,[<Preferred Act1>],[<PreferredAct2>][,<band_1>[,<band_2>[,…….[,<band_n>]]]]]]`

- rat 频段接入类型
  - 1 UMTS
  - 2 LTE
  - 4 LTE/UMTS
  - 10 Auto
  - 14 NR-RAN
  - 16 NR-RAN/WCDMA 
  - 17 NR-RAN/LTE
  - 20 NR-RAN/WCDMA/LTE
- PreferredAct 选择优先接入类型
  - 2 WCDMA is preferred
  - 3 LTE is preferred
  - 6 NR-RAN is preferred
- Band
  - 0 支持当前接入技术所有频段
  - UMTS 
    - 1 BAND_UMTS_1
    - 2 BAND_UMTS_2
    - 3 BAND_UMTS_3
    - 4 BAND_UMTS_4
    - ...
    - 25 BAND_UMTS_25
  - LTE
    - 101 BAND_LTE_1
    - 102 BAND_LTE__2
    - ...
    - 171 BAND_LTE_71
    - 111 BAND_LTE_11
    - 112 BAND_LTE_12
    - ...
    - 120 BAND_LTE_20
  - NR-RAN
    - 501 BAND_NR_1
    - 502 BAND_NR_2
    - ...
    - 5010 BAND_NR_10
    - ...
    - 50512 BAND_NR_512

如果仅需要配置频段，则前三个参数必须为空。 因此，命令如下所示：`AT+GTACT=,,,160, 155` (e 例如：配置 LTE 频段 60 和 LTE 频段 55)。

### 示例

```bash
# 接入技术4g,5g，5g优先，锁B3,B8,n28,n41,n78
at+gtact=17,6,,103,108,5028,5041,5078

# 查询支持的锁频和接入技术
at+gtact=?
+GTACT: (1,2,4,10,14,16,17,20),(2,3,6),(2,3,6),(),(1,5,8),(101,103,105,107,108,120,128,132,138,140,141,142,143),(),(),(501,503,505,507,508,5020,5028,5038,5040,5041,5075,5076,5077,5078)

OK

# 查询当前接入技术和锁频
at+gtact?

+GTACT: 17,6,,103,108,5028,5041,5078

OK
```

## 补充
### 接入技术5GNSA和5G的区别
5G NSA（Non-Standalone）和5G SA（Standalone）的主要差别在于网络架构和依赖程度：

网络架构：

5G NSA：结合已有的4G LTE基础设施。5G NR（新无线接入技术）依赖于4G核心网来进行控制和数据管理。这种方式可以更快地部署5G服务。
5G SA：独立于4G网络，使用全新的5G核心网架构，支持所有5G功能（如网络切片、边缘计算等）。
部署和过渡：

5G NSA：允许运营商利用现有的4G基础设施，减少初期投资，快速提供5G服务。
5G SA：需要全新部署核心网，虽然投资和时间成本较高，但提供更全面的5G功能和性能。
性能和功能：

5G NSA：支持基本的5G功能和低延迟，但在性能上受限于4G核心网。
5G SA：提供更高的速度、更低的延迟，以及更多的创新应用，如网络切片和更高级别的QoS（服务质量）。
总结而言，5G NSA是一种过渡方案，主要依赖于现有的4G基础设施，而5G SA则是完全基于5G技术的独立网络架构。


# PCI
PCI代表“Physical Cell ID”（物理小区ID）。它是一个用于标识移动网络中每个小区的唯一编号，帮助用户设备（UE）识别和连接到特定的基站。

## 锁小区
### 广和通

GTCELLLOCK
此命令用于强制 UE 注册特定的小区（固定的小区和频率）。
`AT+GTCELLLOCK=<mode>[,<rat>,<type>,<earfcn>[,<PCI>][,<scs>][,<nrband>]]`
- mode 功能开关
  - 0:关闭此功能
  - 1:开启此功能
- rat 制式
  - 0:LTE
  - 1:NR
- type 加锁类型
  - 0:锁PCI
  - 1:锁频点
- earfcn 频点
  - 0 - 4294967295
- PCI
  - 物理小区ID
  - RAT=0时，0-503 对应LTE
  - RAT=1时，0-1007 对应NR
- scs 子载波间隔
  - 0:15kHz
  - 1:30kHz
- nrband NR频段
  - 501 BAND_NR_1
  - 502 BAND_NR_2
  - ...
  - 5010 BAND_NR_10
  - 50512 BAND_NR_512


