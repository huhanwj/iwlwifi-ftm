# WiFi FTM 测量平台搭建
对于Linux Kernel在5.4及以上的系统，直接安装最新版本的`iw`及`hostapd`工具，以及较新的Intel无线网卡（如AX200、AX201），即可搭建测距测试平台。

执行测距需要设置Responder端 (AP)与Initiator端 (STA)，下面给出如何进行测距的操作说明：

## 测试平台配置
### 硬件选择
初期测试通过配置（仅供参考）：
* AP端：使用Intel Dual Band Wireless AC 8265 网卡
* STA端：使用Intel Wi-Fi 6E AX210 网卡

其他可用硬件：
1. 8000/9000系列 Intel Dual Band Wireless AC网卡 （如8260,8265,9260,9560等）
2. WiFi 6/6E系列网卡 （如AX200, AX201, AX210等）
3. 与上述网卡同芯片组的Killer网卡（罕见，如手边没有则不推荐使用，例如Killer 1650, Killer 1675等）

**AP端的Intel网卡驱动需要进行一定修改，下面会具体提到如何修改**

### 系统选择及软件、驱动安装
推荐使用Ubuntu 18.04.6 LTS或更新版本的Ubuntu操作系统，如使用最新的WiFi 6E网卡（AX210, AX211等），则推荐直接使用Ubuntu 20.04 LTS。

#### 工具安装
安装`iw`工具
```bash
sudo apt-get update
sudo apt-get install iw
```

安装`hostapd`及其配套软件
```bash
sudo apt-get update
sudo apt-get install net-tools hostapd isc-dhcp-server
```

#### 驱动安装
前面提到，对于AP端，我们需要对Intel网卡驱动进行一定修改，才能使其具备回应FTM请求的能力。

首先安装驱动安装环境依赖项：
```bash
sudo apt-get install gcc make linux-headers-$(uname -r) git-core
```

下载Intel驱动源码，在此推荐使用core58版本驱动:
```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/iwlwifi/backport-iwlwifi.git -b release/core58
```

修改已经下载的驱动源码中的以下2个文件：
1. /drivers/net/wireless/intel/iwlwifi/mvm/mac80211.c
> 搜索`if (fw_has_capa(&mvm->fw->ucode_capa, IWL_UCODE_TLV_CAPA_FTM_CALIBRATED)`字段，将其改为`if (1 || fw_has_capa(&mvm->fw->ucode_capa, IWL_UCODE_TLV_CAPA_FTM_CALIBRATED)`
2. /drivers/net/wireless/intel/iwlwifi/mvm/constants.h
> 搜索`#define IWL_MVM_TOF_IS_RESPONDER`字段，将其后面的值从0改为1

编译驱动：
```bash
make defconfig-iwlwifi-public
sed -i 's/CPTCFG_IWLMVM_VENDOR_CMDS=y/# CPTCFG_IWLMVM_VENDOR_CMDS is not set/' .config
make -j4
```

安装驱动:
```bash
sudo make install
```

完成后重启电脑使驱动生效

### Responder端（AP端）配置
首先创建配置文件`hostapd.conf`：
```bash
interface=wlp2s0 # 改成机器上的网卡接口名称
driver=nl80211
ssid=FTM-TEST # 改一个合适的SSID
hw_mode=g
ieee80211n=1
ht_capab=[HT40+][SHORT-GI-40]
channel=2 #选择合适的信道
wmm_enabled=1
wme_enabled=1
ctrl_interface=/var/run/hostapd
ctrl_interface_group=0
ftm_responder=1
ftm_Initiator=1
```

启动AP：
```bash
sudo hostapd 配置文件路径
```

若无法启动，可以尝试重启网卡：
```bash
sudo nmcli radio wifi off
sudo rfkill unblock wlan
sudo ifconfig wlp2s0 up # 改成当前的网卡接口名称
```

### Initiator端（STA端）配置
使用`iw`工具扫描 AP列表，获取支持FTM协议的AP：
```bash
sudo iw dev 网卡设备名 scan > scan.txt #输出扫描结果到scan.txt
```

在`scan.txt`中搜索FTM，找到刚刚在AP端打开的AP，记下MAC地址和中心频率，创建配置文件，方法入下：
```bash
echo MAC地址 bw=40 cf=中心频率 asap > conf
```

执行测距：
```bash
sudo iw dev 网卡设备名 measurement ftm_request conf
```

距离计算：

在执行测距命令执行后，程序会输出一串数据，其中`RTT_AVG`即为我们需要的时间数据，将其除以2后乘以0.02998即可换算为以厘米为单位的距离。

