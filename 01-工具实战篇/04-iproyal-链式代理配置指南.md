# 20260225 极速、低成本、防封号：IPRoyal + 机场链式代理配置完全指南

**强烈推荐 IPRoyal！**

> - **购买的时候一定要加额外说明，比如是用来使用claude code，这样更有助于申请到更纯净的IP。**
> - **不一定非常纯净，购买完一定要先去查一下这个IP的纯净度再做配置！！！**

- 极速：200ms+ 的延时（基本不超过300ms，这可是链式代理）
- 低成本：2.4 $/月
- 超高带宽：60M+ 下载 30M+ 上传
- 防封号：IP检测全绿，且已被标识是家庭宽带IP

## 0. 前言

很多在玩 Claude、ChatGPT 时，经常遇到被封号或者 IP 限制的问题。普通的机房 IP 容易被风控，而直接买住宅 IP 往往速度极慢（时延 1000ms+）且价格昂贵。

本教程记录了一种 **"黄金组合"** 方案：利用 **机场节点作为前置中转** + **IPRoyal 住宅 IP 作为落地出口**。

- **特点：** 成本极低、网速极快（时延 200ms 左右）、带宽大（60M+）、配置简单、极其稳定。
- **核心原理：** 电脑 -> 机场节点（解决跨境速度） -> 住宅IP（解决账号权重/防封） -> 目标网站。

---

## 1. 准备工作

1. **机场订阅：** 一个速度尚可的普通机场（建议选择支持 UDP 转发的）。推荐 [sakuracat](https://sakuracat.com/)
2. **IPRoyal 静态住宅IP：** [IPRoyal Dashboard](https://dashboard.iproyal.com/) 购买其静态住宅 IP（Static Residential）或按流量计费的住宅代理。获取其 `IP`, `Port`, `Username`, `Password`。
3. **软件环境：** [Clash Verge (Rev)](https://github.com/clash-verge-rev/clash-verge-rev) 建议版本较新，使用 **Mihomo (Clash Meta)** 内核。

---

## 2. PC 核心配置：脚本注入法 (Script Injection)

在 Clash Verge 中，不要直接在订阅里改，也不建议在 `Merge` 里硬写。最稳妥的方法是使用 **"全局扩展脚本 (Script)"**。

### 步骤：

1. 打开 **Clash Verge**，进入 **"配置 (Profiles)"** 页面。
2. 找到右下角的 **全局扩展脚本** script 蓝色按钮并点击。
3. 删除原有内容，粘贴以下脚本：

```JavaScript
function main(config) {
  // 1. 定义您的美国住宅节点基础信息
  const residentialBase = {
    name: "美国静态住宅",
    type: "http",
    server: "xxx.xxx.xxx.xxx",
    port: 12323,
    username: "xxx",
    password: "xxx",
    udp: true
  };

  // 2. 寻找或创建一个"自动优选"策略组作为前置
  const autoGroup = config.find(g => g.type === "url-test") || config;

  if (autoGroup) {
    // 3. 创建链式节点
    const disasterRecoveryNode = {
      ...residentialBase,
      name: "住宅(自动容灾)",
      "dialer-proxy": autoGroup.name
    };

    // 4. 将新节点加入代理列表
    if (!config.proxies) config.proxies = [];
    config.proxies.push(residentialBase);
    config.proxies.push(disasterRecoveryNode);

    // 5. 把新节点塞进手动选择列表
    config.forEach(g => {
      if (g.type === "select" || g.name === "Proxy") {
        g.proxies.unshift(disasterRecoveryNode.name);
      }
    });
  }

  return config;
}
```

4. 点击 **Save (保存)**。
5. 回到主界面，点击右上角的 **刷新图标**。

---

## 3. 验证

- 访问 [speedtest.net](https://www.speedtest.net/) 测速
- 访问 [whoer.net](https://whoer.net) 、[ping0.cc](https://ping0.cc/) 验证 IP，显示为美国且为住宅带宽即大功告成！

---

## 4. 填坑笔记（必看）

### 坑点 1：报错 `use or proxies missing`
- **现象：** 在 `Merge` 框里写了 `proxies` 后保存报错。
- **避坑：** 不要在 Merge 里定义节点！所有节点定义和链式逻辑全部写在 **Script** 里。

### 坑点 2：报错 `unsupported type: relay was removed`
- **现象：** 使用旧的 `type: relay` 语法导致红框报错。
- **避坑：** 使用 **`dialer-proxy`** 属性，直接在节点定义里指定前置。

### 坑点 3：名字对不上
- **现象：** 脚本跑了，但是代理列表里没出现新节点。
- **避坑：** 脚本中使用了 `config.find(g => g.type === "select")` 来自动找组。如果不行，把组名写死。

### 坑点 4：如果IP不纯净，可以联系客服退款
退款可以在右下角找客服，大约10分钟后就会收到退款。

退款如果不用这个IP了，可以把「全局扩展脚本」复原为以下内容：

```javascript
function main(config, profileName) {
  return config;
}
```

---

## 5. 总结

这套方案通过 **机场中转 + 拨号代理注入**，完美解决了住宅 IP "慢、贵、难配置"的痛点。
只要 Script 写得好，从此不封号！
