# MAKCM USB设备识别分析

## PC主机看到的USB设备信息

### 关键发现

**结论：PC主机看到的是ESP32-S3开发板，而不是真实鼠标的原始信息**

---

## 详细分析

### 1. 设备识别信息对比

#### PC主机看到的（MAKCM设备）

| 参数 | 值 | 说明 |
|------|-----|------|
| **VID** | **0x303A** | Espressif Systems（固定） |
| **PID** | **0x1001** | ESP32-S3 通用设备（固定） |
| 制造商 | （真实鼠标的制造商）* | 从真实鼠标复制 |
| 产品名称 | （真实鼠标的产品名）* | 从真实鼠标复制 |
| 序列号 | （真实鼠标的序列号）* | 从真实鼠标复制 |
| USB版本 | （真实鼠标的USB版本） | 从真实鼠标复制 |
| 设备类别 | HID (0x03) | 从真实鼠标复制 |

*字符串描述符从真实鼠标复制，但VID/PID固定为Espressif

#### 真实鼠标原始信息

| 参数 | 示例值 | 说明 |
|------|--------|------|
| **VID** | 0x046D (Logitech) | 鼠标厂商ID |
| **PID** | 0xC332 (G502) | 鼠标产品ID |
| 制造商 | "Logitech" | 字符串描述符 |
| 产品名称 | "G502 HERO Gaming Mouse" | 字符串描述符 |
| 序列号 | "1234567890AB" | 字符串描述符 |

---

### 2. 代码证据分析

#### A. 真实鼠标信息被接收

```cpp
// InitSettings.cpp:145-146
descriptor_device.idVendor = doc["idVendor"];    // 接收真实鼠标VID
descriptor_device.idProduct = doc["idProduct"];  // 接收真实鼠标PID
```

右侧MCU读取真实鼠标的完整USB描述符，并通过JSON发送给左侧MCU。

#### B. 字符串描述符被使用

```cpp
// USBSetup.cpp:31-66
uint16_t const* tud_descriptor_string_cb(uint8_t index, uint16_t langid) {
    switch (index) {
        case 1:
            str = device_info.str_desc_manufacturer;  // 使用真实鼠标制造商
            break;
        case 2:
            str = device_info.str_desc_product;       // 使用真实鼠标产品名
            break;
        case 3:
            str = device_info.str_desc_serial_num;    // 使用真实鼠标序列号
            break;
    }
    // ... 返回字符串描述符
}
```

#### C. 但是VID/PID没有被设置！

```cpp
// USBSetup.cpp:76-88
void InitUSB() {
    USB.usbVersion(descriptor_device.bcdUSB);      // ✅ 设置USB版本
    USB.firmwareVersion(descriptor_device.bcdDevice); // ✅ 设置固件版本
    USB.usbPower(configuration_descriptor.bMaxPower); // ✅ 设置功耗
    USB.usbAttributes(configuration_descriptor.bmAttributes); // ✅ 设置属性
    USB.usbClass(descriptor_device.bDeviceClass);  // ✅ 设置设备类别
    USB.usbSubClass(descriptor_device.bDeviceSubClass); // ✅ 设置子类别
    USB.usbProtocol(descriptor_device.bDeviceProtocol); // ✅ 设置协议

    // ❌ 缺少！没有设置VID/PID
    // USB.VID(descriptor_device.idVendor);   // 这行不存在
    // USB.PID(descriptor_device.idProduct);  // 这行不存在

    Mouse.begin();
    USB.begin();
}
```

#### D. 默认VID/PID来源

```json
// boards/MAKCM.json:18-22
"hwids": [
  [
    "0x303A",   // Espressif VID
    "0x1001"    // ESP32-S3 通用PID
  ]
]
```

**结论**：由于代码中没有调用设置VID/PID的API，Arduino ESP32框架使用板子配置文件中的默认值。

---

### 3. PC主机如何识别设备

#### Windows设备管理器中显示

```
人体学输入设备
  └─ HID 兼容鼠标
      厂商ID: 0x303A (Espressif Systems)
      产品ID: 0x1001
      制造商: Logitech (或其他真实鼠标制造商)
      产品: G502 HERO Gaming Mouse (或其他真实鼠标名称)
```

#### Linux lsusb 输出

```bash
$ lsusb
Bus 001 Device 005: ID 303a:1001 Espressif Logitech G502 HERO Gaming Mouse
```

**注意**：设备被识别为 "Espressif" 制造的设备，但产品名称是真实鼠标的名称。

---

### 4. 与真实鼠标的差异

#### 明显差异

| 项目 | 真实鼠标 | MAKCM设备 | 检测难度 |
|------|---------|----------|---------|
| **VID (厂商ID)** | 0x046D等 | **0x303A** | ⚠️ **容易检测** |
| **PID (产品ID)** | 各不相同 | **0x1001** | ⚠️ **容易检测** |
| 制造商字符串 | Logitech等 | 相同 | ✅ 相同 |
| 产品名称字符串 | G502等 | 相同 | ✅ 相同 |
| 序列号 | 唯一值 | 相同 | ✅ 相同 |
| HID报告描述符 | 原始值 | 重构值 | ⚠️ 可能不同 |
| USB速度 | FS/HS | 相同 | ✅ 相同 |
| 轮询间隔 | bInterval | 相同 | ✅ 相同 |

#### 检测方法

**方法1：检查VID/PID**
```python
import usb.core

# 查找所有USB鼠标
devices = usb.core.find(find_all=True, bDeviceClass=3)

for dev in devices:
    if dev.idVendor == 0x303A and dev.idProduct == 0x1001:
        print("检测到MAKCM设备！")
        print(f"VID: {hex(dev.idVendor)}")
        print(f"PID: {hex(dev.idProduct)}")
```

**方法2：VID/PID不匹配检测**
```python
# 罗技G502的真实VID/PID
REAL_VID = 0x046D
REAL_PID = 0xC332

if dev.idVendor != REAL_VID or dev.idProduct != REAL_PID:
    # 产品名称说是G502，但VID/PID不对
    print("警告：设备标识不匹配！")
    print(f"声称产品: {dev.product}")
    print(f"实际VID/PID: {hex(dev.idVendor)}:{hex(dev.idProduct)}")
    print(f"预期VID/PID: {hex(REAL_VID)}:{hex(REAL_PID)}")
```

**方法3：Windows注册表检查**
```
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Enum\USB\VID_303A&PID_1001\
```

如果看到这个路径，说明连接了ESP32-S3设备。

---

### 5. 反检测建议

如果希望让设备更难被识别为MAKCM，需要修改代码：

#### 修改方案1：动态设置VID/PID（需要代码修改）

```cpp
// 在 USBSetup.cpp 的 InitUSB() 中添加
void InitUSB() {
    // ... 现有代码 ...

    // 添加这两行（需要ESP32库支持）
    USB.VID(descriptor_device.idVendor);   // 使用真实鼠标VID
    USB.PID(descriptor_device.idProduct);  // 使用真实鼠标PID

    Mouse.begin();
    USB.begin();
}
```

**注意**：Arduino ESP32框架可能不支持运行时修改VID/PID。这需要深入TinyUSB库层面修改。

#### 修改方案2：使用TinyUSB设备描述符回调

```cpp
// 实现 tud_descriptor_device_cb() 回调
uint8_t const* tud_descriptor_device_cb(void) {
    static tusb_desc_device_t desc_device;

    desc_device.bLength = 18;
    desc_device.bDescriptorType = TUSB_DESC_DEVICE;
    desc_device.bcdUSB = descriptor_device.bcdUSB;
    desc_device.bDeviceClass = descriptor_device.bDeviceClass;
    desc_device.bDeviceSubClass = descriptor_device.bDeviceSubClass;
    desc_device.bDeviceProtocol = descriptor_device.bDeviceProtocol;
    desc_device.bMaxPacketSize0 = descriptor_device.bMaxPacketSize0;

    // 使用真实鼠标的VID/PID
    desc_device.idVendor = descriptor_device.idVendor;
    desc_device.idProduct = descriptor_device.idProduct;

    desc_device.bcdDevice = descriptor_device.bcdDevice;
    desc_device.iManufacturer = 1;
    desc_device.iProduct = 2;
    desc_device.iSerialNumber = 3;
    desc_device.bNumConfigurations = 1;

    return (uint8_t const*)&desc_device;
}
```

这需要在 `USBSetup.cpp` 中实现，并确保TinyUSB使用这个回调而不是默认值。

---

### 6. 实际影响

#### 对普通使用的影响

**✅ 无影响**
- Windows正确识别为HID鼠标
- 所有鼠标功能正常工作
- 游戏和应用程序无法区分

#### 对安全软件的影响

**⚠️ 可能被检测**

某些反作弊软件会：
1. 枚举所有USB设备
2. 检查VID/PID是否匹配已知鼠标厂商
3. 检测到 `0x303A:0x1001` 可能标记为可疑设备

**示例检测逻辑**：
```cpp
// 伪代码：反作弊软件可能的检测逻辑
if (device.class == HID_MOUSE) {
    if (device.vid == 0x303A && device.pid == 0x1001) {
        // ESP32设备伪装成鼠标
        flag_suspicious();

        if (device.manufacturer.contains("Logitech") ||
            device.manufacturer.contains("Razer")) {
            // VID/PID与制造商字符串不匹配
            flag_high_risk();
        }
    }

    // 检查VID是否在已知鼠标厂商列表中
    if (!is_known_mouse_vendor(device.vid)) {
        flag_suspicious();
    }
}
```

#### 对驱动程序的影响

**⚠️ 可能无法使用厂商驱动**

- 罗技G HUB：不会识别为罗技鼠标（VID不匹配）
- 雷蛇Synapse：不会识别为雷蛇鼠标
- 只能使用Windows通用HID驱动

---

### 7. 其他可能的差异

#### HID报告描述符

虽然系统接收了真实鼠标的HID报告描述符，但左侧MCU使用的是 `USBHIDMouse` 类：

```cpp
// USBSetup.cpp:22
USBHIDMouse Mouse;
```

这个类使用Arduino预定义的HID报告描述符，而不是真实鼠标的原始描述符。

**可能的差异**：
- 按键数量可能不同
- 报告格式可能简化
- 某些高级功能可能缺失（如DPI切换、可编程按键）

#### USB配置描述符

代码中使用了真实鼠标的配置参数：
```cpp
USB.usbPower(configuration_descriptor.bMaxPower);
USB.usbAttributes(configuration_descriptor.bmAttributes);
```

但整体描述符结构由Arduino框架生成，可能与真实鼠标有细微差别。

---

## 总结对比表

### 快速识别MAKCM设备

| 检查项 | 方法 | 准确度 |
|--------|------|--------|
| VID/PID | 检查是否为 `303A:1001` | ⭐⭐⭐⭐⭐ 100% |
| VID/PID不匹配 | 对比产品名称与VID | ⭐⭐⭐⭐⭐ 100% |
| 厂商驱动 | 尝试连接品牌软件 | ⭐⭐⭐⭐ 高 |
| HID描述符 | 深度分析报告格式 | ⭐⭐⭐ 中 |
| 设备行为 | 检测输入延迟模式 | ⭐⭐ 低 |

### 用户体验差异

| 功能 | 真实鼠标 | MAKCM | 差异 |
|------|---------|-------|------|
| 基本鼠标功能 | ✅ | ✅ | 无 |
| 高刷新率支持 | ✅ 8000Hz | ✅ 8000Hz | 无 |
| 厂商驱动支持 | ✅ | ❌ | **有差异** |
| 可编程按键 | ✅ | ❌ | **有差异** |
| 板载配置文件 | ✅ | ❌ | **有差异** |
| RGB灯效控制 | ✅ | ❌ | **有差异** |
| DPI调节 | ✅ | ❌ | **有差异** |
| Windows识别 | ✅ | ✅ | 无 |
| 反作弊检测 | ✅ 通过 | ⚠️ 可能被检测 | **有差异** |

---

## 建议

### 对于开发者

1. **实现VID/PID透传**
   - 修改代码使用真实鼠标的VID/PID
   - 需要深入TinyUSB层面修改

2. **实现完整HID描述符透传**
   - 不使用 `USBHIDMouse` 类
   - 手动构造与真实鼠标相同的HID报告

3. **添加厂商特定功能**
   - 解析厂商自定义HID报告
   - 实现DPI切换等高级功能

### 对于用户

1. **正常使用**
   - 基本鼠标功能完全正常
   - 适合日常办公和游戏

2. **反作弊环境**
   - 某些游戏可能检测到非标准设备
   - 建议在允许的环境中使用

3. **厂商软件**
   - 无法使用品牌驱动（如G HUB）
   - 只能使用Windows通用驱动

---

**文档版本**: 1.0
**分析日期**: 2024
**基于代码**: MAKCM 最新版本
