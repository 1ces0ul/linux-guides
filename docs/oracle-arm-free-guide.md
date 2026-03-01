# Oracle Cloud 免费 ARM 实例（4核24G）抢购教程

> 利用 Oracle Cloud Always Free Tier 获取永久免费的 4 OCPU + 24GB 内存 ARM 服务器。

## 目录

- [背景](#背景)
- [免费额度说明](#免费额度说明)
- [注册 Oracle Cloud 账号](#注册-oracle-cloud-账号)
- [手动创建实例（了解流程）](#手动创建实例了解流程)
- [为什么抢不到](#为什么抢不到)
- [自动抢购脚本](#自动抢购脚本)
- [抢购技巧](#抢购技巧)
- [抢到后的操作](#抢到后的操作)
- [常见问题](#常见问题)

---

## 背景

Oracle Cloud 提供 Always Free Tier（永久免费层），其中最有价值的是基于 Ampere Arm 架构的 A1 Flex 实例：

- **4 OCPU（ARM 核心）+ 24GB 内存**
- **200GB 引导卷存储**（与 AMD 实例共享）
- **每月 10TB 出口流量**
- **永久免费**，不是试用期

这个配置放在任何云厂商都要花钱，Oracle 是唯一提供如此大方免费额度的。

## 免费额度说明

每个 Oracle Cloud 账号的 Always Free 计算资源：

### ARM（Ampere A1.Flex）
- 总共 4 OCPU + 24GB 内存，可任意拆分
- 例如：1 台 4核24G，或 2 台 2核12G，或 4 台 1核6G
- 引导卷最大 200GB（与 AMD 共享总额度）

### AMD（VM.Standard.E2.1.Micro）
- 2 台固定配置实例
- 每台 1 OCPU + 1GB 内存
- 相对容易创建

### 最优方案
- 1 台 ARM 4核24G + 2 台 AMD 1核1G = 总共 3 台免费服务器

## 注册 Oracle Cloud 账号

1. 打开 https://cloud.oracle.com
2. 点击 "Sign Up for Free"
3. 填写邮箱、姓名、国家
4. **选择 Home Region** — 非常重要，选了不能改
   - 推荐冷门区域（见[抢购技巧](#抢购技巧)）
   - 热门区域（东京、首尔、新加坡）资源长期紧张
5. 绑定信用卡验证身份（不会扣费）

### 关于 Home Region

- 免费账号（Free Tier）只能使用 Home Region
- 升级到 Pay As You Go 后才能订阅其他区域
- 升级不会自动扣费，只要使用 Always Free 资源就永远免费
- 但升级时会有临时预授权（约 $100 / £70），几天后自动退回

## 手动创建实例（了解流程）

### 1. 创建 VCN（虚拟云网络）

实例需要网络，首先创建 VCN：

1. 左上角菜单 → Networking → Virtual Cloud Networks
2. 点 "Start VCN Wizard"
3. 选 "Create VCN with Internet Connectivity"
4. 名称随意填（如 `vcn-1`），其他保持默认
5. 点 Create，等待完成

### 2. 创建计算实例

1. 左上角菜单 → Compute → Instances
2. 点 "Create Instance"

#### Image（镜像）
- 点 "Change Image"
- 选择 **aarch64** 版本的镜像（ARM 架构）
- 推荐：Ubuntu 22.04 aarch64 或 Oracle Linux 8/9 aarch64
- ⚠️ 不要选 x86_64 的镜像，跟 ARM 不兼容
- ⚠️ Ubuntu 24.04 aarch64 可能与 A1.Flex 有兼容性问题，建议用 22.04

#### Shape（实例规格）
- 点 "Change Shape"
- 选择 **Ampere** 标签页（不是 AMD 或 Intel）
- 选择 VM.Standard.A1.Flex
- OCPU 填 4，Memory 填 24

#### Networking（网络）
- VCN：选择刚创建的 VCN
- Subnet：选择 Public Subnet
- Public IP：选 "Assign a public IPv4 address"

#### Boot Volume（引导卷）
- 勾选 "Specify a custom boot volume size"
- 大小填 100（GB）

#### SSH Key
- 上传你的 SSH 公钥，或让 Oracle 生成
- **务必保存私钥文件**（.key），丢了就无法登录

### 3. 点击 Create

大概率会看到错误：

```
Out of host capacity
```

这就是为什么需要自动抢购脚本。

## 为什么抢不到

Oracle 的免费 ARM 资源池有限，全球大量用户在抢同一批资源。热门区域的资源几乎永远是满的。

只有当有人释放实例（删除服务器、账号过期等）时，才会短暂出现可用资源，这个窗口可能只有几秒钟。

手动在网页上反复点 Create 是几乎不可能抢到的，需要用脚本自动化、24 小时不间断尝试。

## 自动抢购脚本

### 原理

通过 Oracle Cloud 的 API 自动循环发送创建实例的请求。每 30-60 秒尝试一次，轮流遍历所有可用域（Availability Domain），直到有资源释放时立即抢到。

### 前置准备

#### 1. 获取 API 密钥

1. 登录 Oracle Cloud 控制台
2. 右上角点头像 → "My Profile"
3. 左侧菜单 → "API Keys"
4. 点 "Add API Key" → "Generate API Key Pair"
5. **下载 Private Key（.pem 文件）** — 这是 API 私钥，务必保存
6. 点 "Add"，会显示配置预览

配置预览示例：

```ini
[DEFAULT]
user=ocid1.user.oc1..aaaaaaaa...
fingerprint=12:34:56:78:...
tenancy=ocid1.tenancy.oc1..aaaaaaaa...
region=uk-london-1
key_file=<path to your private keyfile>
```

记下这些信息。

> ⚠️ API Key 和 SSH Key 是两个不同的东西：
> - SSH Key — 用来登录服务器
> - API Key — 用来调用 Oracle API（脚本需要的）

#### 2. 获取必要的 OCID

| 参数 | 获取方式 |
|------|---------|
| Compartment ID | 菜单 → Identity & Security → Compartments → 复制 root compartment 的 OCID |
| Availability Domain | 菜单 → Compute → Create Instance → Placement 栏显示的 AD 名称 |
| Subnet ID | 菜单 → Networking → VCN → 点进 Public Subnet → 复制 OCID |
| Image ID | 通过脚本自动查询（见下方） |

#### 3. 准备运行环境

需要一台 24 小时运行的服务器（如你已有的云服务器）：

```bash
# 安装 Python 和 OCI SDK
pip3 install oci
```

### 配置文件

创建目录和配置：

```bash
mkdir -p ~/oracle-arm
```

将 API 私钥保存到 `~/oracle-arm/oci_api_key.pem`，然后：

```bash
chmod 600 ~/oracle-arm/oci_api_key.pem
```

创建 OCI 配置文件 `~/oracle-arm/config`：

```ini
[DEFAULT]
user=ocid1.user.oc1..aaaaaaaa（你的 user OCID）
fingerprint=12:34:56:78（你的 fingerprint）
tenancy=ocid1.tenancy.oc1..aaaaaaaa（你的 tenancy OCID）
region=uk-london-1（你的 Home Region）
key_file=/root/oracle-arm/oci_api_key.pem
```

```bash
chmod 600 ~/oracle-arm/config
```

### 查询可用镜像

先验证 API 连通性并查询 Ubuntu aarch64 镜像 ID：

```python
import oci

config = oci.config.from_file("/root/oracle-arm/config")

# 验证连接
identity = oci.identity.IdentityClient(config)
user = identity.get_user(config['user']).data
print(f"连接成功！用户: {user.name}")

# 查询 Ubuntu ARM 镜像
compute = oci.core.ComputeClient(config)
images = compute.list_images(
    compartment_id=config["tenancy"],
    operating_system="Canonical Ubuntu",
    shape="VM.Standard.A1.Flex",
    sort_by="TIMECREATED",
    sort_order="DESC",
    limit=5
).data

for img in images:
    print(f"{img.display_name} | {img.id}")
```

从输出中选一个镜像 ID（推荐 Ubuntu 22.04 aarch64）。

### 抢购脚本

创建 `~/oracle-arm/grab.py`：

```python
#!/usr/bin/env python3
"""Oracle ARM A1.Flex 自动抢购脚本"""
import oci
import time
import random
import sys
from datetime import datetime

# ===================== 配置区 =====================

CONFIG_FILE = "/root/oracle-arm/config"

# 替换为你自己的 OCID
COMPARTMENT_ID = "ocid1.tenancy.oc1..aaaaaaaa（你的 tenancy OCID）"
SUBNET_ID = "ocid1.subnet.oc1.xxx（你的 subnet OCID）"
IMAGE_ID = "ocid1.image.oc1.xxx（你查询到的镜像 OCID）"

# 可用域列表（替换为你的区域实际 AD）
# 在 Create Instance 页面的 Placement 栏查看
# 无效的 AD 会返回 404，从列表中移除即可
ADS = [
    "xxxx:REGION-AD-1",
    "xxxx:REGION-AD-2",
]

# 实例配置
INSTANCE_NAME = "arm-free-1"
OCPUS = 4
MEMORY_GB = 24
BOOT_VOLUME_GB = 100

# SSH 公钥（替换为你自己的）
SSH_PUB_KEY = "ssh-rsa AAAA... your-key"

# 重试间隔（秒）
MIN_INTERVAL = 30
MAX_INTERVAL = 60

# ===================== 脚本逻辑 =====================

def log(msg):
    print(f"[{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}] {msg}", flush=True)

def main():
    config = oci.config.from_file(CONFIG_FILE)
    compute = oci.core.ComputeClient(config)

    attempt = 0
    ad_index = 0

    while True:
        attempt += 1
        ad = ADS[ad_index % len(ADS)]
        ad_index += 1

        log(f"第 {attempt} 次尝试 | AD: {ad}")

        try:
            launch_details = oci.core.models.LaunchInstanceDetails(
                compartment_id=COMPARTMENT_ID,
                availability_domain=ad,
                display_name=INSTANCE_NAME,
                shape="VM.Standard.A1.Flex",
                shape_config=oci.core.models.LaunchInstanceShapeConfigDetails(
                    ocpus=OCPUS,
                    memory_in_gbs=MEMORY_GB,
                ),
                source_details=oci.core.models.InstanceSourceViaImageDetails(
                    source_type="image",
                    image_id=IMAGE_ID,
                    boot_volume_size_in_gbs=BOOT_VOLUME_GB,
                ),
                create_vnic_details=oci.core.models.CreateVnicDetails(
                    subnet_id=SUBNET_ID,
                    assign_public_ip=True,
                ),
                metadata={
                    "ssh_authorized_keys": SSH_PUB_KEY,
                },
            )

            response = compute.launch_instance(launch_details)
            instance = response.data

            log(f"🎉 抢到了！Instance ID: {instance.id}")
            log(f"状态: {instance.lifecycle_state}")
            log(f"请等待实例启动完成后获取公网 IP")
            sys.exit(0)

        except oci.exceptions.ServiceError as e:
            if e.status == 500 and "Out of host capacity" in str(e.message):
                log(f"❌ 容量不足，继续等待...")
            elif e.status == 429:
                log(f"⚠️ 请求太频繁，等待更长时间...")
                time.sleep(120)
                continue
            elif "LimitExceeded" in str(e.code):
                log(f"⚠️ 已达到免费额度限制，可能已有实例存在")
                sys.exit(1)
            else:
                log(f"❌ 错误: {e.status} - {e.message}")
        except Exception as e:
            log(f"❌ 未知错误: {e}")

        wait = random.randint(MIN_INTERVAL, MAX_INTERVAL)
        log(f"等待 {wait} 秒后重试...")
        time.sleep(wait)

if __name__ == "__main__":
    main()
```

### 运行脚本

```bash
# 后台运行，日志输出到文件
cd ~/oracle-arm
nohup python3 -u grab.py >> grab.log 2>&1 &

# 查看实时日志
tail -f grab.log

# 查看最近日志
tail -20 grab.log

# 停止脚本
pkill -f grab.py
```

### 日志示例

```
[2026-03-01 05:49:07] 第 1 次尝试 | AD: MiMZ:UK-LONDON-1-AD-1
[2026-03-01 05:49:08] ❌ 容量不足，继续等待...
[2026-03-01 05:49:08] 等待 47 秒后重试...
[2026-03-01 05:49:55] 第 2 次尝试 | AD: MiMZ:UK-LONDON-1-AD-2
[2026-03-01 05:49:56] ❌ 容量不足，继续等待...
...
[2026-03-01 14:23:17] 第 892 次尝试 | AD: MiMZ:UK-LONDON-1-AD-1
[2026-03-01 14:23:18] 🎉 抢到了！Instance ID: ocid1.instance.oc1...
```

## 抢购技巧

### 选择冷门区域

注册时选 Home Region 很关键。容易抢到的区域（相对冷门）：

| 区域 | 难度 |
|------|------|
| US West (Phoenix) | ⭐ 较易 |
| US West (San Jose) | ⭐ 较易 |
| Canada Southeast (Montreal) | ⭐ 较易 |
| US East (Ashburn) | ⭐⭐ 中等 |
| Germany Central (Frankfurt) | ⭐⭐ 中等 |
| UK South (London) | ⭐⭐⭐ 较难 |
| Japan East (Tokyo) | ⭐⭐⭐⭐ 极难 |
| South Korea Central (Seoul) | ⭐⭐⭐⭐ 极难 |
| Singapore | ⭐⭐⭐⭐ 极难 |

### 其他技巧

- **先抢小配置**：1 OCPU + 6GB 成功率更高，抢到后再扩容
- **升级账户**：Pay As You Go 用户的优先级高于 Free Tier（升级不自动扣费）
- **凌晨/周末**：资源释放概率更高
- **耐心**：有人跑了一周才抢到，属于正常情况
- **不要太频繁**：间隔设太短会被限流（429 错误），30-60 秒是合理范围

### 升级到 Pay As You Go

如果想提高成功率：

1. 控制台顶部有 "Upgrade" 提示
2. 选择 Credit Card，填写信用卡信息
3. 会有临时预授权（约 $100/£70），几天后自动退回
4. 升级后仍然使用 Always Free 资源不会扣费
5. 升级后可以订阅其他区域（免费账号只能用 Home Region）

## 抢到后的操作

### 获取公网 IP

1. 菜单 → Compute → Instances
2. 点击实例名称
3. 在 "Primary VNIC" 部分找到 Public IP

### SSH 连接

```bash
ssh -i your-private-key.key ubuntu@公网IP    # Ubuntu 系统
ssh -i your-private-key.key opc@公网IP       # Oracle Linux 系统
```

### 安全组配置

默认只开放 SSH（22 端口）。如需开放其他端口：

1. 菜单 → Networking → VCN → 点进你的 VCN
2. Security Lists → Default Security List
3. Add Ingress Rules，添加需要的端口

## 常见问题

### Q: "Out of host capacity" 是什么意思？

A: 当前区域和可用域没有空闲的 ARM 资源。这是正常现象，继续用脚本刷就行。

### Q: 出现 404 错误是怎么回事？

A: 某个 Availability Domain 不存在或你的账户没有权限使用。从脚本的 ADS 列表中移除该 AD。

### Q: 429 错误怎么办？

A: 请求太频繁被限流了。脚本会自动等待 120 秒后继续。如果频繁出现，可以把 MIN_INTERVAL 调大。

### Q: LimitExceeded 错误？

A: 你已经达到了免费额度上限。检查是否已经有 ARM 实例在运行（包括之前创建的）。

### Q: Image 和 Shape 不兼容？

A: 选错了镜像架构。ARM（Ampere A1.Flex）必须用 aarch64 的镜像，不能用 x86_64 的。

### Q: 脚本运行一会儿就停了？

A: 可能是 SSH 断开导致后台进程被杀。确保用 `nohup` 运行，或者用 `screen` / `tmux`：

```bash
# 用 tmux
tmux new -s oracle
python3 -u grab.py | tee grab.log
# Ctrl+B 然后按 D 分离
# tmux attach -t oracle 重新连接
```

### Q: 免费账号和 Pay As You Go 的区别？

A: 免费账号只能用 Home Region，Pay As You Go 可以订阅多个区域。两者都能使用 Always Free 资源，Pay As You Go 用户抢购优先级更高。升级不会自动扣费。

### Q: 抢到的实例会被回收吗？

A: Oracle 的 Always Free 资源是永久的，但有一个条件：如果你的实例连续 7 天 CPU 使用率低于 10%，Oracle 可能会回收。保持一定的使用率即可。

---

## 参考

- [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/)
- [Oracle Cloud Always Free Resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)
- [OCI Python SDK 文档](https://docs.oracle.com/en-us/iaas/tools/python/latest/)
