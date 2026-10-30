# zxtelnet

在 **Linux（Debian）** 上开启中兴光猫 / ONU 的 Telnet，不需要 Windows，不需要 wine。

这个仓库是在 Linux 上原生编译、实测跑通的版本，附带源码、编译产物与完整踩坑记录。

---

## 这是什么

「书院烧香中兴telnet开启工具（第二款）」是一个 Windows GUI 的 exe 加壳程序。它真正的干活部分是一个 Go 原生程序，内嵌了开源项目 [zteOnu](https://github.com/Septrum101/zteOnu) 的完整代码。

这类工具**只通过 HTTP 和 telnet 与光猫通信，与操作系统完全无关**——所以可以在 Debian 上直接原生编译运行，完全用不着 wine。

本仓库把这套能力搬到了 Linux，并记录了在真实设备上的实测结果。

## 快速开始

```bash
git clone https://github.com/czghzh/zxtelnet.git
cd zxtelnet

# 方式一：直接用预编译好的二进制
chmod +x zteonu-linux-amd64
./zteonu-linux-amd64 -i 192.168.1.1 --port 80 -u CUAdmin -p CUAdmin --telnet

# 方式二：装进系统
sudo install -m 755 zteonu-linux-amd64 /usr/local/bin/zteonu
zteonu -i 192.168.1.1 --port 80 -u CUAdmin -p CUAdmin --telnet
```

看到下面这段就是成功了：

```
step [0] reset factory: ok
step [1] request factory mode: ok
step [2] send sq: ok
step [3] check login auth: ok
step [4] enter factory mode: ok
temp user: xxxxxxxx, pass: xxxxxxxx
-----------------------------------
telnet verified, temp factory telnet is open
Permanent Telnet saved
user: root, pass: Zte521
restarting telnetd in place (no reboot)..
telnetd restarted, verifying permanent telnet..
permanent telnet verified after in-place restart
```

然后：

```bash
telnet 192.168.1.1
# Login: root
# Password: Zte521
```

## 两个必踩的坑

### 坑一：端口不是 8080

大多数教程默认 `--port 8080`，但**很多光猫的 8080 是关闭的，只有 80 开放**。

先探测再动手：

```bash
for p in 80 8080; do
    timeout 2 bash -c "</dev/tcp/192.168.1.1/$p" 2>/dev/null && echo "$p open" || echo "$p closed"
done
```

如果 8080 closed、80 open，就加 `--port 80`。

### 坑二：默认账号密码很可能不对

工具默认用 `telecomadmin / nE7jA%5m`，在定制固件上经常返回 HTTP 401（`errors user or password`）。

**按运营商试这些：**

| 运营商 | 用户名 | 密码 |
|---|---|---|
| 联通 | `CUAdmin` | `CUAdmin` |
| 电信 | `telecomadmin` | `nE7jA%5m` |
| 移动 | `CMCCAdmin` | `aDm8H%MdA` |
| 通用 | `admin` | `admin` |

本地实测的那台是联通定制固件，真正能用的是 **`CUAdmin / CUAdmin`**。

> 注意：网上文档常说「新版固件要把网卡 MAC 改成 `00:07:29:55:35:57`」。这条**不是万能药**——实测中换 MAC 后依然是 401，问题出在凭据而不是 MAC。别在 MAC 上浪费时间。

## 参数说明

| 参数 | 短参数 | 默认值 | 说明 |
|---|---|---|---|
| `--ip` | `-i` | `192.168.1.1` | 光猫 IP |
| `--port` | | `8080` | 光猫 HTTP 端口 |
| `--user` | `-u` | `telecomadmin` | 工厂模式用户名 |
| `--pass` | `-p` | `nE7jA%5m` | 工厂模式密码 |
| `--telnet` | | `false` | 固化永久 telnet，**原地重启 telnetd，不重启设备** |
| `--telnet-restart` | | `false` | 固化永久 telnet，通过**重启设备**生效 |
| `--tp` | | `23` | 光猫 telnet 端口 |
| `--iface` | | 自动 | 指定用哪块网卡的 MAC |
| `--mac` | `-m` | 自动 | 手动指定客户端 MAC |

**`--telnet` 和 `--telnet-restart` 二选一。** 推荐 `--telnet`，它只重启 telnetd 服务，不断网。

## 自己编译

```bash
# 需要 Go 1.26+
git clone --depth 1 https://github.com/Septrum101/zteOnu.git
cd zteOnu
go build -trimpath -ldflags '-s -w' -o zteonu .
```

没装 Go 的话：

```bash
curl -sL -o go.tgz https://go.dev/dl/go1.27.3.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go.tgz
export PATH=/usr/local/go/bin:$PATH
```

源码在本仓库 `src/` 目录也有一份。

## 实测设备

| 项目 | 值 |
|---|---|
| 型号 | 中兴 F7610U |
| 固件 | V1.2.0P1N28 |
| 定制 | 联通 |
| 光猫 IP | 192.168.1.1 |
| HTTP 端口 | 80（8080 关闭） |
| 有效凭据 | `CUAdmin / CUAdmin` |
| 固化结果 | `TelnetCfg.Lan_Enable = 1`，重启后保留 |

登录后确认设备状态：

```
/ # upgradetest getver
Main  version num: V1.2.0P1N28
/ # cat /proc/cpuusage
average: 0.01%
/ # cat /proc/tempsensor
temper value:62
```

## 开启 telnet 之后能做什么

```bash
# 查看/修改超级密码
sendcmd 1 DB set DevAuthInfo 0 User username
sendcmd 1 DB set DevAuthInfo 0 Pass password

# 查看设备信息
sendcmd 1 DB p DevInfo
upgradetest getver

# 改地区（执行后会重启 telnet，需重新开启）
cat /etc/init.d/regioncode
upgradetest sdefconf 210     # 广东

# 删 TR069 远程管理
sendcmd 1 DB set MgtServer 0 URL http://127.0.0.1
sendcmd 1 DB set MgtServer 0 Tr069Enable 0

# 保存！不保存重启就丢
sendcmd 1 DB save
```

## 安全提醒

`Zte521` 是公网皆知的默认密码，telnet 一旦固化就是常开的。建议：

1. 立刻改掉默认密码
2. 不要把这个端口映射到公网
3. 只在受信任的 LAN 内使用

## 致谢与许可

- 核心代码来自 [Septrum101/zteOnu](https://github.com/Septrum101/zteOnu)（原作者 thank243 的仓库已删除）
- 遵循 **AGPL-3.0** 许可，见 [LICENSE](LICENSE)
- 本仓库仅做 Linux 化编译、实测记录与文档整理

## 免责声明

仅供**自己拥有或获得授权**的设备使用。改配置前请先备份，操作后果自负。
