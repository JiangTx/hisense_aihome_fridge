# 海信爱家 冰箱 (Hisense AiHome Fridge) —— Home Assistant 自定义集成

把「海信爱家」App 里的**海信冰箱**接入 Home Assistant。**纯 custom_components 手动安装，不需要 HACS。**

> 目前**仅适配冰箱品类**（海信爱家云端 `connector.device.type.fridge`），故命名带「冰箱」。
> 其他品类（空调、洗衣机等）功能定义不同，需要另行适配。

已在 **冰箱 BCD-415WKR1DPGA-FM51** 上完成实机验证。

---

## 一、安装（二选一）

### 方式 A：把整个文件夹拷进 HA

1. 打开 Home Assistant 的配置目录（HAOS/Supervised 可用 Samba 或「文件编辑器」插件；
   Docker 版就是映射出来的 `config/` 目录）。
2. 确认目录结构如下（若没有 `custom_components` 就新建一个）：

```
config/
└── custom_components/
    └── hisense_aihome/          ← 整个文件夹放这里（域名保持不变）
        ├── __init__.py
        ├── api.py
        ├── binary_sensor.py
        ├── config_flow.py
        ├── const.py
        ├── coordinator.py
        ├── entity.py
        ├── manifest.json
        ├── number.py
        ├── select.py
        ├── sensor.py
        ├── switch.py
        └── translations/
            ├── en.json
            └── zh-Hans.json
```

### 方式 B：用 zip 上传

把 `hisense_aihome.zip` 解压到 `config/` 下，确保结果是 `config/custom_components/hisense_aihome/...`。

> ⚠️ 放好后**必须重启 Home Assistant**（设置 → 系统 → 重启 → 重启 Home Assistant）。

---

## 二、添加集成

1. 设置 → 设备与服务 → 右下角「**+ 添加集成**」
2. 搜索「**海信爱家 冰箱**」（或 `hisense_aihome`）
3. 输入「海信爱家」App 使用的**手机号**和**密码**
   - ⚠️ 必须是**设置过密码**的账号。如果你平时只用短信验证码 / 一键登录，请先在海信爱家 App 里设置一个密码。
4. 下一步会**列出账号下的所有设备**，勾选要接入的（冰箱等）
5. 完成 —— 一台设备会作为**一个设备卡片**出现，下面挂载全部实体

---

## 三、生成的实体（以冰箱为例）

所有实体都归属到**同一个设备**下（`device_info.identifiers` 按 device_id 统一），
所以「设置 → 设备」里只会看到一台「冰箱」，点进去是下面这一串：

| 类型 | 实体 | 说明 |
|---|---|---|
| `sensor` | 冷藏室实际温度 | `refrigeratorRealTemperature` |
| `sensor` | 冷冻室实际温度 | `freezeRealTemperature` |
| `sensor` | 冷藏室设定温度 | 回读值 |
| `sensor` | 冷冻室设定温度 | 回读值 |
| `sensor` | 变温抽屉 | 冰鲜 / 0°保鲜 / 母婴 |
| `sensor` | 在线状态 | 诊断类 |
| `sensor` | 故障 | 汇总所有故障/告警位，正常时显示「正常」 |
| `sensor` | **本月开门次数** | 来自「使用报告」，附分时段分布 |
| `number` | 冷藏室设定温度 | 可调 **2 ~ 8 ℃** |
| `number` | 冷冻室设定温度 | 可调 **-25 ~ -15 ℃** |
| `select` | 模式 | 智能 / 速冷 / 一键冰封 |
| `select` | 变温抽屉模式 | 冰鲜 / 0°保鲜 / 母婴 |
| `switch` | 除菌净化 | `ClearSwitch` |
| `binary_sensor` | 冷藏室 / 冷冻室 / 变温室 开门告警 | |

> **为什么「速冷」等没有做成开关？**
> 云端 `workMode` 是**单值互斥枚举**（0 智能 / 3 速冷 / 7 一键冰封），
> 任意时刻只能是其中一种。若做成三个独立开关，HA 设备页标题栏的「总开关」
> 会对全部开关同时下发 `turn_on`，三条命令互相覆盖，最终落在最后一个上
> —— 这正是「点总开关就触发速冷」的原因。而且实测下发
> `SetModes.QuickCooling` 这类子命令会把 `workMode` 置为非法值 `"--"`。
> 因此三种模式统一由「模式」下拉表达，`switch.py` 只保留真正独立的布尔量。
>
> **「使用报告」还有更多指标**（开门时长 / 平均时长 / 除菌次数 / 分时段分布…）
> 读取逻辑已全部写好，目前只启用了「本月开门次数」。
> 需要时在 `sensor.py` 的 `async_setup_entry` 里取消对应注释即可，
> 完整字段清单见 `const.py` 末尾「其他可读取数据」。

### 关于「使用报告」类统计

开门次数、开门时长、除菌次数**不在设备状态里**，而在海信智慧家云的一个独立看板接口
（App 首页「使用报告 / 月度使用情况」卡片，模板 5237）：

```
POST https://api-wg.hismarttv.com/agw/dataBoard/getWMReportMonthData?accessToken=...
Content-Type: application/json

{"deviceid":"<序列号>","accessToken":"...","wifiId":"<comModuleId>","homeid":"...",
 "yearcode":"2026","monthcode":"10","type":"month","useJson":true}
```

返回：`switchCount`（开门次数）、`switchDuration`（累计开门秒）、`switchAvgDuration`（平均秒）、
`degermCount`（除菌次数）、`switchCountDis`（分时段分布）、`switchAvgRank`（全国排名）、
`titlename`（称号）、`bindDays`（已绑定天数）。

**历史可拉取范围**：不限固定月数，**从设备绑定日起的全部历史**。
本机实测可回溯至 **2024-10**（绑定日 2024-10-06，`bindDays=740`），共 24 个月有数据；
绑定之前的月份返回空。`type` 仅支持 `month` / `year`（`day` / `week` 会报错）。

集成会**每 3 小时**拉取一次当月报告（该接口按天更新，没必要高频请求）。

> ⚠️ **没有电费 / 度数（kWh）数据**。`POST /4.0/iot/databoard/fridge/energy/query`
> 接口存在，但本机型固件未采集电量，`totalEnergy` / `saveEnergy` 恒为 `"0"`。
> 这台冰箱在硬件层面就没有电量计。

---

## 四、原理

### 云端结构

| 用途 | 接口 |
|---|---|
| 登录 | `POST https://portal-account.hismarttv.com/mobile/se/signon` |
| 设备列表 + 实时状态 | `POST https://iot-aihome.hismarttv.com/4.0/iot/devices/list` |
| 设备功能定义 | `POST https://iot-aihome.hismarttv.com/4.0/iot/devices/function/getByProduct` |
| **下发控制** | `POST https://iot-aihome.hismarttv.com/4.0/iot/devices/execute` |



### 实测可用的命令

| 功能 | 命令 | 参数 |
|---|---|---|
| 统一模式 | `SetModes` | `{"modes":{"value":0/3/7}}`（0智能 3速冷 7一键冰封） |
| 冷藏室温度 | `SetTemperature.Refrigerator` | `{"refrigeratorSetTemperature":{"value":2~8}}` |
| 冷冻室温度 | `SetTemperature.Freeze` | `{"freezeSetTemperature":{"value":-25~-15}}` |
| 变温抽屉 | `VariationTemperature` | `{"variationTemperatureDrawer":{"value":0/1/2}}` |
| 除菌净化 | `ClearSwitch` | `{"clearSwitch":{"value":0/1}}` |

> `SetModes.Smart` / `SetModes.QuickCooling` / `SetModes.QuickIceBound` 这三个子命令
> 云端**会接受**（返回 SUCCESS），但实测会把 `workMode` 置为非法值 `"--"`，
> 且与 `SetModes` 抢同一字段。**不要使用**，请统一走 `SetModes`。

以下命令本机型**不支持**（返回 `Unsupported command`）：
`SetModes.QuickFrozen`、`SetModes.SuperFrozen`、`SetModes.QuickDeepFrozen`、
`SetModes.Holiday`、`SetModes.PowerSaving`、`SetModes.FastStorage`、`SetModes.Daily`、
`SetModes.AIControlTemp`、`SetAIEcoSwitch`、`Power`。

> 这些命令在下发点（H5 侧）确实存在，但**本机型固件不支持** ——
> 这是典型的「界面有、设备无」。调试时可用云端错误语义区分：
> `Unsupported command 'x'` = 命令名不对（或本机型不支持）；
> `Missing required action param 'x'` = 命令名对、参数名错；
> `out of range. Effective range is [...]` = 值越界，**提示里会直接给出合法范围**。

### token 有效期

`accessToken` 有效期 **3 天**（`tokenExpiredTime: 259200` 秒），
`refreshToken` 150 天。集成在收到鉴权失败时会用保存的账号密码**自动重新登录**，
无需手工干预。

---

## 五、排错

| 现象 | 原因 / 处理 |
|---|---|
| 添加集成时提示「账号或密码错误」 | 密码必须是**海信爱家 App 当前的密码**；若从未设置过，先在 App 里设置 |
| 登录成功但设备列表为空 | 账号下确实没设备，或设备绑在另一个海信账号（海信有「爱家」和「智慧家」两套云） |
| 提示「无法连接海信云端」 | 检查 HA 主机的外网访问 |
| 控制无响应 | 云端返回 `SUCCESS` 仅供参考 —— 部分命令云端接受但固件忽略；本集成已剔除本机型不支持的命令 |
| 想改刷新频率 | 集成卡片 → 配置 → 修改「状态刷新间隔」（默认 60 秒） |

若海信更新 App 导致签名失效，重新抓包替换 `const.py` 里的
`PORTAL_APP_SECRET` / `AIHOME_SIGN_KEY` 即可，其余代码无需改动。

---

## 六、说明

- 本集成通过**海信官方云端**通信，需要设备在线且海信云可用；不是本地直连。
- 不含任何个人凭据；账号密码只保存在你自己的 HA 配置里。
- `PORTAL_APP_SECRET` / `AIHOME_SIGN_KEY` 是随 App 下发的客户端常量，非用户凭据。
