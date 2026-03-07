# 20260222 Quantumult X 手机端配置教程 (含美区 ID 充值)

本教程涵盖了从美区 Apple ID 充值到 Quantumult X 深度配置的全过程。

---

## 教程信息

- **适用对象**: 需要在 iOS 端实现科学上网并优化国内 App 体验的用户
- **核心流程**: 礼品卡充值 -> 软件下载 -> 基础配置 -> 进阶分流

---

## 一、 美区 Apple ID 充值 (支付宝渠道)

由于美区 App Store 不支持国内 Visa/银联卡，使用支付宝购买礼品卡是最稳妥的方式。

### 1. 购买礼品卡

1. **选择平台**: 推荐使用 `Seagm.com` 或 `OffGamers`（均明确支持支付宝）。
2. **操作步骤**:
   - 访问平台官网，搜索 "Apple App Store & iTunes United States"。
   - 选择面额（Quantumult X 价格为 $9.9，建议充值 $10 或以上）。
   - 支付方式选择 **Alipay (支付宝)**，扫码完成支付。
   - 在个人中心或注册邮箱中查收 **16 位兑换码**。

### 2. 兑换至 Apple ID

1. 手机打开 **App Store**，确保已登录美区账号。
2. 点击右上角**个人头像**。
3. 选择 **Redeem Gift Card or Code (兑换礼品卡或代码)**。
4. 手动输入 16 位代码，点击右上角 **Redeem** 完成充值。

---

## 二、 下载与初次启动

1. 在 App Store 搜索 `Quantumult X` 并购买下载（目前价格约为 $9.9）。
2. 打开软件，会弹出 VPN 配置请求，点击 **允许 (Allow)** 并验证手机密码/FaceID。

---

## 三、 节点与分流配置 (核心步骤)

### 1. 导入机场订阅

1. 点击主界面右下角 **风车图标**。
2. 在 `节点 (Servers)` 栏目下，点击 **引用 (Resource Search)**。
3. 点击右上角 **`+`**。
4. **标签**: 填入机场名称；**资源路径**: 粘贴机场提供的订阅链接。
5. 点击右上角确定。返回列表，**向左滑动**该条目点击 **更新 (Update)**。

### 2. 进阶：导入专用分流规则 (最专业、最省心)

这是最推荐的方法，因为 Twitter 和 Discord 背后域名极多，手动配置容易遗漏。

1. 打开 QX，点击右下角 **风车图标**。
2. 找到 **分流 (Filter)** -> **引用 (Resource Search)**。
3. 点击右上角 **`+`**。
4. 分别添加以下链接（如遇 404 可将 `raw.githubusercontent.com` 替换为 `raw.gitmirror.com`）：
   - **Twitter 规则**: `https://raw.githubusercontent.com/blackmatrix7/ios_rule_script/master/rule/QuantumultX/Twitter/Twitter.list`
   - **Discord 规则**: `https://raw.githubusercontent.com/blackmatrix7/ios_rule_script/master/rule/QuantumultX/Discord/Discord.list`
5. **关键步骤**: 添加并保存后，回到列表页，**长按**刚添加的规则条目，选择 **策略 (Policy)**，将其指向你的 **代理 (Proxy)** 或机场节点组。
6. 点击 **更新 (Update)**。

### 3. 手动补充特定 App 走代理 (以 WhatsApp 为例)

如果自动规则不生效，建议手动添加本地规则：

1. 点击 **风车图标** -> **分流 (Filter)** -> **分流列表 (Filter list)**。
2. 点击右上角 **`+`**，依次直接粘贴并保存以下行：
   - `HOST-KEYWORD, whatsapp, Proxy`
   - `HOST-SUFFIX, whatsapp.net, Proxy`
   - `HOST-SUFFIX, fbcdn.net, Proxy` (解决 WhatsApp 图片/语音无法接收问题)
   - `HOST-KEYWORD, twitter, Proxy`
   - `HOST-KEYWORD, discord, Proxy`

### 4. 优化国内 App 体验 (解决微信等走代理问题)

**关键操作：将保底规则改为直连。**

1. 滑动到设置页面底部，点击 **配置文件 (Files)** -> **编辑 (Edit)**。
2. 划到代码最底部，找到 `FINAL, Proxy`。
3. 将其修改为：`FINAL, Direct`。
4. **意义**: 只有你指定的 App 走代理，其他国内 App（微信、美团等）默认直连，速度最快且省流量。

---

## 四、 进阶：MitM 证书安装 (去广告必备)

1. **生成**: 设置页面 -> 工具 -> **MitM** -> **生成证书**。
2. **安装**: 点击 **安装证书**，在 Safari 弹窗中点允许。
3. **激活**:
   - 打开手机 **设置** -> **通用** -> **关于本机** -> **证书信任设置** -> 开启 `Quantumult X` 开关。
   - 回到 QX 开启 MitM 开关。

---

## 五、 日常维护小贴士

- **确认生效**: 查看 QX 主页底部的 **日志 (文件夹图标)**，查看域名后的标记是 `Proxy` 还是 `Direct`。
- **测速**: 在主页点击节点列表，点击"波纹"图标进行延迟测试。
- **模式切换**: 长按主页底部风车图标，平时保持在 **分流模式 (Filter)** 即可。
