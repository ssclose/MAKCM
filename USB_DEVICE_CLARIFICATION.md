# MAKCM USB设备识别 - 更正与澄清

## 重要更正

基于用户反馈：**G HUB确实可以识别设备**，这说明我之前的分析存在遗漏。

---

## 重新检查代码

### 1. HID报告描述符的处理

#### 右侧MCU确实请求了HID报告描述符

```cpp
// esp_usb_host.cpp:478
submitControl(0x81, 0x00, 0x22, currentInterfaceNumber,
              hid_descriptors[hidDescriptorCounter].wReportLength);
```

**解释**：
- `0x22` = HID Report Descriptor类型
- 请求真实鼠标的HID报告描述符

#### 接收和解析

```cpp
// esp_usb_host.cpp:755-778
void EspUsbHost::_onReceiveControl(usb_transfer_t *transfer) {
    // 1. 接收HID报告描述符（原始字节）
    uint8_t *p = &transfer->data_buffer[8];

    // 2. 检查是否为鼠标（查找 0x05 0x01 0x09 0x02 序列）
    if (p[i] == 0x05 && p[i + 1] == 0x01 &&
        p[i + 2] == 0x09 && p[i + 3] == 0x02) {
        isMouse = true;
    }

    // 3. 解析HID报告描述符
    HIDReportDescriptor descriptor = parseHIDReportDescriptor(
        &transfer->data_buffer[8],
        transfer->actual_num_bytes - 8
    );

    // 4. ⚠️ 关键：解析后释放，没有发送到左侧MCU
    usb_host_transfer_free(transfer);
}
```

**问题**：
- ✅ 接收了真实鼠标的HID报告描述符
- ✅ 解析了格式（按钮/轴位置）
- ❌ **没有发送原始字节给左侧MCU**
- ❌ 左侧MCU使用 `USBHIDMouse` 类的默认描述符

---

### 2. 左侧MCU的HID实现

#### 使用Arduino框架的USBHIDMouse类

```cpp
// USBSetup.cpp:22
USBHIDMouse Mouse;

// USBSetup.cpp:86-87
Mouse.begin();
USB.begin();
```

**USBHIDMouse类的特点**：
- 使用Arduino预定义的HID报告描述符
- 标准5键鼠标格式（左、右、中、前进、后退）
- X/Y轴为8位有符号整数
- 滚轮为8位有符号整数

#### 与真实鼠标HID的差异

| 项目 | 真实鼠标（如G502） | MAKCM (USBHIDMouse) |
|------|-------------------|---------------------|
| HID报告描述符 | 原始鼠标的描述符 | Arduino标准描述符 |
| 按键数量 | 可能5+个 | 5个（标准） |
| DPI按键 | ✅ 有 | ❌ 无 |
| 可编程按键 | ✅ 有 | ❌ 无 |
| 厂商特定功能 | ✅ 有 | ❌ 无 |
| 基本鼠标功能 | ✅ | ✅ |

---

### 3. VID/PID设置情况

#### 当前代码中未设置

```cpp
// USBSetup.cpp:76-88
void InitUSB() {
    USB.usbVersion(descriptor_device.bcdUSB);
    USB.firmwareVersion(descriptor_device.bcdDevice);
    USB.usbPower(configuration_descriptor.bMaxPower);
    USB.usbAttributes(configuration_descriptor.bmAttributes);
    USB.usbClass(descriptor_device.bDeviceClass);
    USB.usbSubClass(descriptor_device.bDeviceSubClass);
    USB.usbProtocol(descriptor_device.bDeviceProtocol);

    // ❌ 缺失！没有调用VID/PID设置方法
    // 可能的API（如果存在）：
    // USB.VID(descriptor_device.idVendor);
    // USB.PID(descriptor_device.idProduct);

    Mouse.begin();
    USB.begin();
}
```

#### 默认值来源

```json
// boards/MAKCM.json:18-22
"hwids": [
  ["0x303A", "0x1001"]  // Espressif VID:PID
]
```

---

## 用户反馈分析

### 关键问题：G HUB为什么能识别？

#### 可能性1：Arduino ESP32框架支持VID/PID设置

Arduino ESP32框架可能有以下API（需验证）：

```cpp
// 可能存在的API
USB.VID(0x046D);  // Logitech VID
USB.PID(0xC332);  // G502 PID

// 或者
USB.productID(0xC332);
USB.vendorID(0x046D);
```

**如果固件已经添加了这些行**，那么G HUB就能识别。

#### 可能性2：TinyUSB描述符回调

可能在其他文件中实现了 `tud_descriptor_device_cb()`：

```cpp
// 可能存在的实现
uint8_t const* tud_descriptor_device_cb(void) {
    static tusb_desc_device_t desc = {
        .idVendor = descriptor_device.idVendor,   // 真实鼠标VID
        .idProduct = descriptor_device.idProduct, // 真实鼠标PID
        // ...
    };
    return (uint8_t const*)&desc;
}
```

**证据**：USBSetup.cpp中有注释提到这些回调函数（第28行）

```cpp
// USBSetup.cpp:25-29
/*
// prep migration
tud_descriptor_device_cb, tud_descriptor_configuration_cb, tud_descriptor_string_cb
*/
```

说明开发者计划或已经实现了这些回调。

#### 可能性3：G HUB的识别不仅基于VID/PID

G HUB可能通过以下方式识别：
- 字符串描述符（制造商、产品名）
- HID报告格式
- 设备行为特征
- 特殊的HID请求

但这种可能性较低，因为通常驱动软件首先检查VID/PID。

---

## 需要澄清的问题

### 问题1：固件是否可以设置VID/PID？

**答案**：理论上可以，但需要：

#### 方法A：使用Arduino ESP32 API（如果支持）

```cpp
void InitUSB() {
    // 设置VID/PID（如果API存在）
    USB.VID(descriptor_device.idVendor);
    USB.PID(descriptor_device.idProduct);

    // 其他设置...
    USB.usbVersion(descriptor_device.bcdUSB);
    // ...

    Mouse.begin();
    USB.begin();
}
```

**检查方法**：
```cpp
// 在InitUSB()函数中添加调试输出
Serial0.print("Setting VID: 0x");
Serial0.println(descriptor_device.idVendor, HEX);
Serial0.print("Setting PID: 0x");
Serial0.println(descriptor_device.idProduct, HEX);
```

#### 方法B：实现TinyUSB回调

```cpp
// 在USBSetup.cpp中添加
uint8_t const* tud_descriptor_device_cb(void) {
    static tusb_desc_device_t desc_device = {
        .bLength = sizeof(tusb_desc_device_t),
        .bDescriptorType = TUSB_DESC_DEVICE,
        .bcdUSB = descriptor_device.bcdUSB,
        .bDeviceClass = descriptor_device.bDeviceClass,
        .bDeviceSubClass = descriptor_device.bDeviceSubClass,
        .bDeviceProtocol = descriptor_device.bDeviceProtocol,
        .bMaxPacketSize0 = descriptor_device.bMaxPacketSize0,

        // 使用真实鼠标的VID/PID
        .idVendor = descriptor_device.idVendor,
        .idProduct = descriptor_device.idProduct,

        .bcdDevice = descriptor_device.bcdDevice,
        .iManufacturer = 1,
        .iProduct = 2,
        .iSerialNumber = 3,
        .bNumConfigurations = 1
    };
    return (uint8_t const*)&desc_device;
}
```

**注意**：这需要在 `Mouse.begin()` 之前调用，或者确保TinyUSB使用这个回调。

---

### 问题2：HID报告描述符是否完整复制？

**当前代码分析**：

```cpp
// esp_usb_host.cpp:775
// ❌ 解析后就释放了，没有发送到左侧MCU
HIDReportDescriptor descriptor = parseHIDReportDescriptor(...);
usb_host_transfer_free(transfer);  // 释放，数据丢失
```

**结论**：
- ❌ **没有**将原始HID报告描述符发送到左侧MCU
- ❌ 左侧MCU使用Arduino标准HID描述符
- ✅ 但基本功能足够（5键+XY+滚轮）

**如果需要完整复制**，需要：

1. **在右侧MCU添加发送**
```cpp
void EspUsbHost::_onReceiveControl(usb_transfer_t *transfer) {
    // 解析
    HIDReportDescriptor descriptor = parseHIDReportDescriptor(...);

    // ✅ 添加：发送原始字节到左侧MCU
    String hexData = "";
    for (int i = 8; i < transfer->actual_num_bytes; i++) {
        hexData += String(transfer->data_buffer[i], HEX) + " ";
    }
    serial1Send("USB_sendHIDReport:%s\n", hexData.c_str());

    usb_host_transfer_free(transfer);
}
```

2. **在左侧MCU接收和使用**
```cpp
// 需要实现自定义HID设备类，不能用USBHIDMouse
// 使用TinyUSB的tud_hid_descriptor_report_cb()
uint8_t const* tud_hid_descriptor_report_cb(uint8_t instance) {
    // 返回从右侧MCU接收到的原始HID报告描述符
    return stored_hid_report_descriptor;
}
```

---

## 测试和验证

### 如何确认VID/PID是什么？

#### Windows方法

1. **设备管理器**
   - 人体学输入设备 → HID兼容鼠标
   - 右键 → 属性 → 详细信息
   - 属性：硬件ID
   - 查看：`USB\VID_xxxx&PID_yyyy`

2. **USBDeview工具**
   - 下载NirSoft的USBDeview
   - 查找MAKCM设备
   - 查看VID/PID列

#### Linux方法

```bash
# 方法1：lsusb
lsusb
# 输出示例：
# Bus 001 Device 005: ID 303a:1001 Espressif ...  (如果未设置)
# Bus 001 Device 005: ID 046d:c332 Logitech ...   (如果已设置)

# 方法2：查看详细信息
lsusb -v -d 303a:1001
# 或
lsusb -v -d 046d:c332
```

#### Python脚本验证

```python
import usb.core
import usb.util

# 查找所有HID设备
devices = usb.core.find(find_all=True, bDeviceClass=0x03)

for dev in devices:
    print(f"\n设备信息:")
    print(f"  VID: 0x{dev.idVendor:04X}")
    print(f"  PID: 0x{dev.idProduct:04X}")
    print(f"  制造商: {usb.util.get_string(dev, dev.iManufacturer)}")
    print(f"  产品: {usb.util.get_string(dev, dev.iProduct)}")
    print(f"  序列号: {usb.util.get_string(dev, dev.iSerialNumber)}")

    # 检查是否为MAKCM
    if dev.idVendor == 0x303A and dev.idProduct == 0x1001:
        print("  ⚠️ 这是ESP32-S3设备（默认VID/PID）")
    elif dev.idVendor == 0x046D:  # Logitech
        print("  ✅ 这是Logitech设备（可能已设置真实VID/PID）")
```

---

## 关于G HUB识别的可能情况

### 情况1：固件已修改

如果G HUB确实能识别，最可能的原因是：

**固件已经实现了VID/PID设置，但不在当前GitHub仓库的代码中**

可能的修改：
```cpp
// 在InitUSB()中添加（可能在用户的本地版本）
void InitUSB() {
    USB.VID(descriptor_device.idVendor);   // ← 添加了这行
    USB.PID(descriptor_device.idProduct);  // ← 添加了这行

    USB.usbVersion(descriptor_device.bcdUSB);
    // ... 其他代码
}
```

### 情况2：G HUB的特殊识别机制

G HUB可能通过以下组合识别：
- 字符串描述符："Logitech" + "G502"
- HID报告格式匹配
- 设备类别/子类别/协议
- 特殊的HID Feature Report

但这不太可能，因为：
- 大多数驱动软件首先检查VID/PID
- 字符串不可靠（容易伪造）

### 情况3：使用了通用Logitech驱动

可能不是G HUB，而是Windows的通用Logitech驱动识别了设备。

---

## 结论和建议

### 当前状态（基于代码）

| 功能 | 状态 | 说明 |
|------|------|------|
| 接收真实鼠标VID/PID | ✅ | InitSettings.cpp接收 |
| 设置VID/PID到USB | ❓ | 代码中未见，但用户说G HUB可识别 |
| 接收HID报告描述符 | ✅ | 右侧MCU接收并解析 |
| 发送HID报告描述符 | ❌ | 未发送到左侧MCU |
| 使用真实HID描述符 | ❌ | 使用USBHIDMouse标准描述符 |
| 字符串描述符透传 | ✅ | 制造商、产品名、序列号 |

### 给用户的问题

为了准确分析，请帮忙确认：

1. **VID/PID验证**
   ```
   请在Windows设备管理器中查看：
   人体学输入设备 → HID兼容鼠标 → 属性 → 详细信息 → 硬件ID

   显示的是：
   A) USB\VID_303A&PID_1001  （ESP32默认）
   B) USB\VID_046D&PID_C332  （Logitech G502）
   C) 其他？
   ```

2. **固件版本**
   ```
   你使用的固件是：
   A) GitHub仓库中的原始代码
   B) 自己修改过的版本
   C) 其他来源的固件
   ```

3. **G HUB识别细节**
   ```
   G HUB显示的是：
   A) 完整识别（可以调DPI、编程按键等）
   B) 部分识别（只识别为Logitech鼠标，但功能受限）
   C) 仅基本功能（作为通用HID鼠标）
   ```

### 下一步

根据你的反馈，我可以：
1. 如果VID/PID已设置：找出设置的代码位置
2. 如果需要设置：提供完整的实现代码
3. 如果需要HID透传：提供完整的实现方案

---

**文档版本**: 2.0 (更正版)
**日期**: 2024
**状态**: 等待用户反馈确认
