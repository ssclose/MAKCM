# MAKCM 协议文档

## 项目概述

**MAKCM (Kmbox Alternative)** 是一个基于双ESP32-S3微控制器的开源鼠标控制系统，用于实现远程鼠标输入注入和透传功能。本文档详细描述了系统的通信协议、命令接口和数据格式。

**版本**: 1.0
**最后更新**: 2024

---

## 目录

1. [系统架构](#系统架构)
2. [通信层协议](#通信层协议)
3. [命令协议规范](#命令协议规范)
4. [数据结构定义](#数据结构定义)
5. [通信流程](#通信流程)
6. [错误处理](#错误处理)
7. [使用示例](#使用示例)

---

## 系统架构

### 硬件架构

```
┌──────────────────────────────────────────────────────────────┐
│                         MAKCM 系统                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐        UART1         ┌─────────────┐      │
│  │  左侧 MCU    │◄──────────────────►│  右侧 MCU    │      │
│  │ (USB Device) │    4Mbps 内部通信    │ (USB Host)  │      │
│  └──────┬───────┘                     └──────┬───────┘      │
│         │                                    │              │
│         │ USB HID                            │ USB          │
│         │ (Mouse)                            │              │
│         ▼                                    ▼              │
│    ┌─────────┐                         ┌──────────┐        │
│    │   PC    │                         │真实鼠标   │        │
│    └─────────┘                         └──────────┘        │
│         ▲                                                   │
│         │ UART0 (115200-5Mbps)                             │
│         │                                                   │
│    ┌─────────┐                                             │
│    │Python工具│                                             │
│    └─────────┘                                             │
└──────────────────────────────────────────────────────────────┘
```

### 功能模块

#### 左侧 MCU (Device Mode)
- **文件**: `MAKCM_ESP32s3_Device_Mouse_Left/`
- **角色**: USB HID 鼠标设备
- **功能**:
  - 接收串口命令（UART0, UART1）
  - 模拟鼠标输入发送到PC
  - 处理USB描述符初始化
  - 命令解析和执行

#### 右侧 MCU (Host Mode)
- **文件**: `MAKCM_ESP32s3_HID_Mouse_Right/`
- **角色**: USB 主机
- **功能**:
  - 接收真实鼠标输入
  - 解析HID报告描述符
  - 转发鼠标事件到左侧MCU
  - USB设备管理

#### Python AIO 工具
- **文件**: `AIO_Tool/MAKCM_Aio_Tool.py`
- **功能**:
  - 用户界面
  - 固件烧录
  - 调试和测试命令发送

---

## 通信层协议

### 物理层

#### UART0 (PC ↔ 左侧MCU)
- **波特率**: 115200 - 5000000 bps (可调)
- **数据位**: 8
- **停止位**: 1
- **校验**: None
- **流控**: None
- **方向**: 双向
- **用途**: 用户命令、调试输出

#### UART1 (左侧MCU ↔ 右侧MCU)
- **波特率**: 4000000 bps (4Mbps)
- **数据位**: 8
- **停止位**: 1
- **校验**: None
- **流控**: None
- **方向**: 双向
- **用途**: 内部通信、鼠标事件、USB描述符传输

### 帧格式

所有命令均采用文本格式，以换行符 `\n` 结尾：

```
<命令>[(<参数1>[,参数2...])]\n
```

**特点**:
- 最大命令长度: 620 字节
- 环形缓冲区大小: 620 字节
- 行结束符: `\n` (回车符 `\r` 被忽略)
- 参数分隔符: `,`
- 参数包裹: `()`

**示例**:
```
km.move(100,50)\n
km.left(1)\n
DEBUG_ON\n
USB_HELLO\n
```

---

## 命令协议规范

### 命令分类

命令按功能和来源分为以下类别：

1. **鼠标控制命令** (km.*) - 来自PC或右侧MCU
2. **USB内部命令** (USB_*) - 右侧MCU与左侧MCU通信
3. **调试命令** (DEBUG_*, ESPLOG_*) - 调试和日志
4. **系统配置命令** (SERIAL_*) - 系统参数设置

---

### 1. 鼠标控制命令 (km.*)

这些命令用于控制鼠标行为，可通过UART0（来自PC）或UART1（来自右侧MCU的真实鼠标事件）发送。

#### km.move

**描述**: 相对移动鼠标指针

**格式**: `km.move(x,y)`

**参数**:
- `x`: 水平移动量 (int, -127 到 127)
- `y`: 垂直移动量 (int, -127 到 127)

**示例**:
```
km.move(50,30)    # 向右移动50像素，向下移动30像素
km.move(-20,10)   # 向左移动20像素，向下移动10像素
```

**实现位置**:
- 发送端: `esp_usb_host.cpp:878`
- 接收端: `handleCommands.cpp:139-143, 219-235`

**特殊处理**:
- 使用互斥锁保护，确保串口命令优先级高于USB鼠标
- 通过任务通知机制触发鼠标移动任务
- 原子变量防止并发问题

**源代码**:
```cpp
// 右侧MCU发送
serial1Send("km.move(%d,%d)\n", report.x, report.y);

// 左侧MCU接收处理
void handleKmMoveCommand(const char *command) {
    int x, y;
    sscanf(command + strlen("km.move") + 1, "%d,%d", &x, &y);
    {
        std::lock_guard<std::mutex> lock(commandMutex);
        moveX = x;
        moveY = y;
    }
    if (mouseMoveTaskHandle != NULL) {
        xTaskNotifyGive(mouseMoveTaskHandle);
    }
}
```

---

#### km.moveto

**描述**: 绝对移动鼠标指针到指定坐标

**格式**: `km.moveto(x,y)`

**参数**:
- `x`: 目标X坐标 (int)
- `y`: 目标Y坐标 (int)

**示例**:
```
km.moveto(500,300)  # 移动到屏幕坐标(500, 300)
```

**实现**:
```cpp
void handleKmMoveto(const char *command) {
    int x, y;
    sscanf(command + strlen("km.moveto") + 1, "%d,%d", &x, &y);
    handleMoveto(x, y);
}

void handleMoveto(int x, int y) {
    Mouse.move(x - mouseX, y - mouseY);
    mouseX = x;
    mouseY = y;
}
```

---

#### km.getpos

**描述**: 获取当前鼠标位置

**格式**: `km.getpos()`

**参数**: 无

**返回**: `km.pos(x,y)`

**示例**:
```
发送: km.getpos()
返回: km.pos(1024,768)
```

**实现**:
```cpp
void handleGetPos() {
    Serial0.println("km.pos(" + String(mouseX) + "," + String(mouseY) + ")");
}
```

---

#### km.left

**描述**: 鼠标左键控制

**格式**: `km.left(state)`

**参数**:
- `state`: `0` = 释放, `1` = 按下

**示例**:
```
km.left(1)  # 按下左键
km.left(0)  # 释放左键
```

**实现**:
```cpp
// 按下
void handleKmMouseButtonLeft1(const char *command) {
    if (!isLeftButtonPressed.exchange(true)) {
        handleMouseButton(MOUSE_BUTTON_LEFT, true);
    }
}

// 释放
void handleKmMouseButtonLeft0(const char *command) {
    if (isLeftButtonPressed.exchange(false)) {
        handleMouseButton(MOUSE_BUTTON_LEFT, false);
    }
}
```

**特性**:
- 使用原子变量防止重复按键事件
- 状态缓存，避免随机点击

---

#### km.right

**描述**: 鼠标右键控制

**格式**: `km.right(state)`

**参数**:
- `state`: `0` = 释放, `1` = 按下

**示例**:
```
km.right(1)  # 按下右键
km.right(0)  # 释放右键
```

---

#### km.middle

**描述**: 鼠标中键控制

**格式**: `km.middle(state)`

**参数**:
- `state`: `0` = 释放, `1` = 按下

**示例**:
```
km.middle(1)  # 按下中键
km.middle(0)  # 释放中键
```

---

#### km.side1

**描述**: 鼠标侧键1（前进键）控制

**格式**: `km.side1(state)`

**参数**:
- `state`: `0` = 释放, `1` = 按下

**示例**:
```
km.side1(1)  # 按下侧键1
km.side1(0)  # 释放侧键1
```

**映射**: MOUSE_BUTTON_FORWARD

---

#### km.side2

**描述**: 鼠标侧键2（后退键）控制

**格式**: `km.side2(state)`

**参数**:
- `state`: `0` = 释放, `1` = 按下

**示例**:
```
km.side2(1)  # 按下侧键2
km.side2(0)  # 释放侧键2
```

**映射**: MOUSE_BUTTON_BACKWARD

---

#### km.wheel

**描述**: 鼠标滚轮滚动

**格式**: `km.wheel(delta)`

**参数**:
- `delta`: 滚动量 (int, 正数向上滚动，负数向下滚动)

**示例**:
```
km.wheel(3)   # 向上滚动3个单位
km.wheel(-2)  # 向下滚动2个单位
```

**实现**:
```cpp
void handleKmWheel(const char *command) {
    int wheelMovement;
    sscanf(command + strlen("km.wheel") + 1, "%d", &wheelMovement);
    handleMouseWheel(wheelMovement);
}
```

---

### 2. USB 内部命令 (USB_*)

这些命令用于左右MCU之间的USB设备管理和描述符传输。

#### USB_HELLO

**描述**: USB设备连接通知

**方向**: 右侧MCU → 左侧MCU

**格式**: `USB_HELLO`

**触发**: 当右侧MCU检测到USB鼠标连接时

**响应**: 左侧MCU开始请求USB描述符

**流程**:
```
1. 右侧MCU检测到USB设备连接
2. 发送 USB_HELLO
3. 左侧MCU设置 deviceConnected = true
4. 开始USB描述符请求序列
```

**实现**:
```cpp
// 右侧MCU
serial1Send("USB_HELLO\n");

// 左侧MCU
void handleUsbHello(const char *command) {
    deviceConnected = true;
    usbReady = true;
    processingUsbCommands = true;
    currentCommandIndex = 0;
    sendNextCommand();  // 开始请求描述符
}
```

---

#### USB_GOODBYE

**描述**: USB设备断开通知

**方向**: 右侧MCU → 左侧MCU

**格式**: `USB_GOODBYE`

**触发**: 当USB鼠标断开连接时

**响应**: 左侧MCU重置所有鼠标状态并重启

**实现**:
```cpp
void handleUsbGoodbye(const char *command) {
    Serial0.println("USB Device disconnected. Restarting!");
    handleMove(0, 0);
    handleMouseButton(MOUSE_BUTTON_LEFT, false);
    handleMouseButton(MOUSE_BUTTON_RIGHT, false);
    handleMouseButton(MOUSE_BUTTON_MIDDLE, false);
    handleMouseButton(MOUSE_BUTTON_FORWARD, false);
    handleMouseButton(MOUSE_BUTTON_BACKWARD, false);
    handleMouseWheel(0);
    vTaskDelay(100);
    ESP.restart();
}
```

---

#### USB_ISNULL

**描述**: 无USB设备连接

**方向**: 右侧MCU → 左侧MCU

**格式**: `USB_ISNULL`

**触发**: 当没有USB设备连接时

**响应**: 设置 `deviceConnected = false`

---

#### USB_INIT

**描述**: USB初始化完成通知

**方向**: 左侧MCU → 右侧MCU

**格式**: `USB_INIT`

**触发**: 当左侧MCU完成所有USB描述符的接收和初始化后

**意义**: 通知右侧MCU可以开始发送鼠标事件

---

#### USB描述符传输命令

以下命令用于传输USB描述符信息，所有数据均采用JSON格式。

##### USB_sendDeviceInfo:

**描述**: 传输设备基本信息

**格式**: `USB_sendDeviceInfo:{json}`

**JSON结构**:
```json
{
  "speed": 2,
  "dev_addr": 1,
  "vMaxPacketSize0": 64,
  "bConfigurationValue": 1,
  "str_desc_manufacturer": "Logitech",
  "str_desc_product": "USB Gaming Mouse",
  "str_desc_serial_num": "1234567890"
}
```

**字段说明**:
- `speed`: USB速度 (0=低速, 1=全速, 2=高速)
- `dev_addr`: 设备地址
- `vMaxPacketSize0`: 端点0最大包大小
- `bConfigurationValue`: 配置值
- `str_desc_manufacturer`: 制造商字符串
- `str_desc_product`: 产品名称字符串
- `str_desc_serial_num`: 序列号字符串

---

##### USB_sendDescriptorDevice:

**描述**: 传输USB设备描述符

**格式**: `USB_sendDescriptorDevice:{json}`

**JSON结构**:
```json
{
  "bLength": 18,
  "bDescriptorType": 1,
  "bcdUSB": 512,
  "bDeviceClass": 0,
  "bDeviceSubClass": 0,
  "bDeviceProtocol": 0,
  "bMaxPacketSize0": 64,
  "idVendor": 1133,
  "idProduct": 49970,
  "bcdDevice": 12288,
  "iManufacturer": 1,
  "iProduct": 2,
  "iSerialNumber": 3,
  "bNumConfigurations": 1
}
```

**字段说明**:
- `bLength`: 描述符长度
- `bDescriptorType`: 描述符类型
- `bcdUSB`: USB版本号
- `bDeviceClass`: 设备类别码
- `idVendor`: 厂商ID
- `idProduct`: 产品ID
- `bcdDevice`: 设备版本号

---

##### USB_sendEndpointDescriptors:

**描述**: 传输端点描述符数组

**格式**: `USB_sendEndpointDescriptors:{json_array}`

**JSON结构**:
```json
[
  {
    "bLength": 7,
    "bDescriptorType": 5,
    "bEndpointAddress": 129,
    "endpointID": 1,
    "direction": "IN",
    "bmAttributes": 3,
    "attributes": "Interrupt",
    "wMaxPacketSize": 64,
    "bInterval": 1
  }
]
```

**字段说明**:
- `bEndpointAddress`: 端点地址
- `endpointID`: 端点ID (0-15)
- `direction`: 数据方向 ("IN" / "OUT")
- `bmAttributes`: 端点属性位掩码
- `attributes`: 传输类型 ("Control", "Isochronous", "Bulk", "Interrupt")
- `wMaxPacketSize`: 最大包大小
- `bInterval`: 轮询间隔

---

##### USB_sendInterfaceDescriptors:

**描述**: 传输接口描述符数组

**格式**: `USB_sendInterfaceDescriptors:{json_array}`

**JSON结构**:
```json
[
  {
    "bLength": 9,
    "bDescriptorType": 4,
    "bInterfaceNumber": 0,
    "bAlternateSetting": 0,
    "bNumEndpoints": 1,
    "bInterfaceClass": 3,
    "bInterfaceSubClass": 1,
    "bInterfaceProtocol": 2,
    "iInterface": 0
  }
]
```

**字段说明**:
- `bInterfaceClass`: 接口类别 (3 = HID)
- `bInterfaceSubClass`: 接口子类别 (1 = Boot Interface)
- `bInterfaceProtocol`: 接口协议 (2 = Mouse)

---

##### USB_sendHidDescriptors:

**描述**: 传输HID描述符数组

**格式**: `USB_sendHidDescriptors:{json_array}`

**JSON结构**:
```json
[
  {
    "bLength": 9,
    "bDescriptorType": 33,
    "bcdHID": 273,
    "bCountryCode": 0,
    "bNumDescriptors": 1,
    "bReportType": 34,
    "wReportLength": 148
  }
]
```

**字段说明**:
- `bcdHID`: HID版本号
- `bCountryCode`: 国家代码
- `bReportType`: 报告描述符类型
- `wReportLength`: 报告描述符长度

---

##### USB_sendIADescriptors:

**描述**: 传输接口关联描述符

**格式**: `USB_sendIADescriptors:{json}`

**JSON结构**:
```json
{
  "bLength": 8,
  "bDescriptorType": 11,
  "bFirstInterface": 0,
  "bInterfaceCount": 2,
  "bFunctionClass": 3,
  "bFunctionSubClass": 0,
  "bFunctionProtocol": 0,
  "iFunction": 0
}
```

---

##### USB_sendEndpointData:

**描述**: 传输端点数据关联信息

**格式**: `USB_sendEndpointData:{json_array}`

**JSON结构**:
```json
[
  {
    "bInterfaceNumber": 0,
    "bInterfaceClass": 3,
    "bInterfaceSubClass": 1,
    "bInterfaceProtocol": 2,
    "bCountryCode": 0
  }
]
```

---

##### USB_sendUnknownDescriptors:

**描述**: 传输未识别的描述符

**格式**: `USB_sendUnknownDescriptors:{json_array}`

**JSON结构**:
```json
[
  {
    "bLength": 10,
    "bDescriptorType": 255,
    "data": "0A FF 01 02 03 04 05 06 07 08"
  }
]
```

---

##### USB_sendDescriptorconfig:

**描述**: 传输配置描述符

**格式**: `USB_sendDescriptorconfig:{json}`

**JSON结构**:
```json
{
  "bLength": 9,
  "bDescriptorType": 2,
  "wTotalLength": 59,
  "bNumInterfaces": 2,
  "bConfigurationValue": 1,
  "iConfiguration": 0,
  "bmAttributes": 160,
  "bMaxPower": 50
}
```

**字段说明**:
- `wTotalLength`: 配置描述符总长度
- `bNumInterfaces`: 接口数量
- `bmAttributes`: 配置属性 (bit7=总线供电, bit6=自供电, bit5=远程唤醒)
- `bMaxPower`: 最大功耗 (单位：2mA)

---

### 3. 调试命令

#### DEBUG_ON / DEBUG_OFF

**描述**: 开启或关闭调试模式

**格式**:
- `DEBUG_ON`
- `DEBUG_OFF`

**方向**: PC → 左侧MCU → 右侧MCU

**示例**:
```
DEBUG_ON   # 启用调试输出
DEBUG_OFF  # 禁用调试输出
```

---

#### DEBUG_<level>

**描述**: 设置调试级别

**格式**: `DEBUG_<0-6>`

**参数**:
- `level`: 调试级别 (0-6)
  - 0: None
  - 1: Error
  - 2: Warning
  - 3: Info
  - 4: Debug
  - 5: Verbose
  - 6: Very Verbose

**示例**:
```
DEBUG_3  # 设置为Info级别
DEBUG_5  # 设置为Verbose级别
```

---

#### ESPLOG_

**描述**: 从右侧MCU传输日志消息到左侧MCU

**方向**: 右侧MCU → 左侧MCU → PC

**格式**: `ESPLOG_<message>`

**示例**:
```
ESPLOG_Mouse device connected
ESPLOG_HID Report Descriptor parsed
```

**实现**:
```cpp
void handleEspLog(const char *command) {
    const char *message = command + strlen("ESPLOG_");
    if (strlen(message) > 0) {
        Serial0.println(message);
    }
}
```

---

#### PRINT_Parsed_Descriptors

**描述**: 打印已解析的USB描述符

**格式**: `PRINT_Parsed_Descriptors`

**响应**: 输出所有已保存的USB描述符信息

---

### 4. 系统配置命令

#### SERIAL_<speed>

**描述**: 动态调整UART0波特率

**格式**: `SERIAL_<speed>`

**参数**:
- `speed`: 波特率 (115200 - 5000000)

**示例**:
```
SERIAL_115200   # 设置为115200 bps
SERIAL_921600   # 设置为921600 bps
SERIAL_4000000  # 设置为4Mbps
```

**实现**:
```cpp
void handleSerial0Speed(const char *command) {
    int speed;
    if (sscanf(command + strlen("SERIAL_"), "%d", &speed) == 1) {
        if (speed >= 115200 && speed <= 5000000) {
            Serial0.end();
            vTaskDelay(1000 / portTICK_PERIOD_MS);
            Serial0.begin(speed);
            Serial0.onReceive(serial0ISR);
            Serial0.println("Serial0 speed change successful.");
        }
    }
}
```

---

## 数据结构定义

### DeviceInfo

设备基本信息结构体

```cpp
struct DeviceInfo {
    uint8_t speed;                      // USB速度
    uint8_t dev_addr;                   // 设备地址
    uint8_t vMaxPacketSize0;            // 端点0最大包大小
    uint8_t bConfigurationValue;        // 配置值
    char str_desc_manufacturer[64];     // 制造商字符串
    char str_desc_product[64];          // 产品字符串
    char str_desc_serial_num[64];       // 序列号字符串
};
```

---

### DescriptorDevice

USB设备描述符

```cpp
struct DescriptorDevice {
    uint8_t bLength;                    // 描述符长度
    uint8_t bDescriptorType;            // 描述符类型 (1)
    uint16_t bcdUSB;                    // USB规范版本
    uint8_t bDeviceClass;               // 设备类别码
    uint8_t bDeviceSubClass;            // 设备子类别码
    uint8_t bDeviceProtocol;            // 设备协议码
    uint8_t bMaxPacketSize0;            // 端点0最大包大小
    uint16_t idVendor;                  // 厂商ID (VID)
    uint16_t idProduct;                 // 产品ID (PID)
    uint16_t bcdDevice;                 // 设备版本号
    uint8_t iManufacturer;              // 制造商字符串索引
    uint8_t iProduct;                   // 产品字符串索引
    uint8_t iSerialNumber;              // 序列号字符串索引
    uint8_t bNumConfigurations;         // 配置数量
};
```

---

### usb_endpoint_descriptor_t

端点描述符

```cpp
struct usb_endpoint_descriptor_t {
    uint8_t bLength;                    // 描述符长度
    uint8_t bDescriptorType;            // 描述符类型 (5)
    uint8_t bEndpointAddress;           // 端点地址
    uint8_t endpointID;                 // 端点ID (0-15)
    String direction;                   // 方向 ("IN"/"OUT")
    uint8_t bmAttributes;               // 属性位掩码
    String attributes;                  // 传输类型
    uint16_t wMaxPacketSize;            // 最大包大小
    uint8_t bInterval;                  // 轮询间隔
};
```

---

### usb_interface_descriptor_t

接口描述符

```cpp
struct usb_interface_descriptor_t {
    uint8_t bLength;                    // 描述符长度
    uint8_t bDescriptorType;            // 描述符类型 (4)
    uint8_t bInterfaceNumber;           // 接口编号
    uint8_t bAlternateSetting;          // 备用设置
    uint8_t bNumEndpoints;              // 端点数量
    uint8_t bInterfaceClass;            // 接口类别 (3=HID)
    uint8_t bInterfaceSubClass;         // 接口子类别
    uint8_t bInterfaceProtocol;         // 接口协议
    uint8_t iInterface;                 // 接口字符串索引
};
```

---

### usb_hid_descriptor_t

HID描述符

```cpp
struct usb_hid_descriptor_t {
    uint8_t bLength;                    // 描述符长度
    uint8_t bDescriptorType;            // 描述符类型 (33)
    uint16_t bcdHID;                    // HID版本号
    uint8_t bCountryCode;               // 国家代码
    uint8_t bNumDescriptors;            // 描述符数量
    uint8_t bReportType;                // 报告描述符类型
    uint16_t wReportLength;             // 报告描述符长度
};
```

---

### HIDReportDescriptor

HID报告解析结构

```cpp
struct HIDReportDescriptor {
    uint8_t reportId;                   // 报告ID
    uint8_t buttonSize;                 // 按键位域大小
    uint8_t xAxisSize;                  // X轴数据位大小
    uint8_t yAxisSize;                  // Y轴数据位大小
    uint8_t wheelSize;                  // 滚轮数据位大小
    uint8_t buttonStartByte;            // 按键数据起始字节
    uint8_t xAxisStartByte;             // X轴数据起始字节
    uint8_t yAxisStartByte;             // Y轴数据起始字节
    uint8_t wheelStartByte;             // 滚轮数据起始字节
};
```

---

### endpoint_data_t

端点关联数据

```cpp
struct endpoint_data_t {
    uint8_t bInterfaceNumber;           // 接口编号
    uint8_t bInterfaceClass;            // 接口类别
    uint8_t bInterfaceSubClass;         // 接口子类别
    uint8_t bInterfaceProtocol;         // 接口协议
    uint8_t bCountryCode;               // 国家代码
};
```

---

### DescriptorConfiguration

配置描述符

```cpp
struct DescriptorConfiguration {
    uint8_t bLength;                    // 描述符长度
    uint8_t bDescriptorType;            // 描述符类型 (2)
    uint16_t wTotalLength;              // 配置描述符集总长度
    uint8_t bNumInterfaces;             // 接口数量
    uint8_t bConfigurationValue;        // 配置值
    uint8_t iConfiguration;             // 配置字符串索引
    uint8_t bmAttributes;               // 配置属性
    uint8_t bMaxPower;                  // 最大功耗 (2mA单位)
};
```

---

## 通信流程

### 1. 系统初始化流程

```
┌──────────┐                          ┌──────────┐
│  右侧MCU  │                          │  左侧MCU  │
└─────┬────┘                          └─────┬────┘
      │                                     │
      │ 1. 检测到USB鼠标连接                 │
      │                                     │
      │ 2. USB_HELLO                        │
      ├────────────────────────────────────►│
      │                                     │
      │                                     │ 3. 开始请求描述符
      │                                     │
      │ 4. sendDeviceInfo                   │
      │◄────────────────────────────────────┤
      │                                     │
      │ 5. USB_sendDeviceInfo:{json}        │
      ├────────────────────────────────────►│
      │                                     │
      │ 6. sendDescriptorDevice             │
      │◄────────────────────────────────────┤
      │                                     │
      │ 7. USB_sendDescriptorDevice:{json}  │
      ├────────────────────────────────────►│
      │                                     │
      │ 8. sendEndpointDescriptors          │
      │◄────────────────────────────────────┤
      │                                     │
      │ 9. USB_sendEndpointDescriptors:[...]│
      ├────────────────────────────────────►│
      │                                     │
      │ ... (重复其余6个描述符) ...          │
      │                                     │
      │                                     │ 10. 初始化USB设备
      │                                     │
      │ 11. USB_INIT                        │
      │◄────────────────────────────────────┤
      │                                     │
      │ 12. 开始发送鼠标事件                 │
      │                                     │
```

### 描述符请求序列

左侧MCU按以下顺序请求9个描述符：

1. `sendDeviceInfo` → 接收 `USB_sendDeviceInfo:{json}`
2. `sendDescriptorDevice` → 接收 `USB_sendDescriptorDevice:{json}`
3. `sendEndpointDescriptors` → 接收 `USB_sendEndpointDescriptors:[...]`
4. `sendInterfaceDescriptors` → 接收 `USB_sendInterfaceDescriptors:[...]`
5. `sendHidDescriptors` → 接收 `USB_sendHidDescriptors:[...]`
6. `sendIADescriptors` → 接收 `USB_sendIADescriptors:{json}`
7. `sendEndpointData` → 接收 `USB_sendEndpointData:[...]`
8. `sendUnknownDescriptors` → 接收 `USB_sendUnknownDescriptors:[...]`
9. `sendDescriptorconfig` → 接收 `USB_sendDescriptorconfig:{json}`

完成后，左侧MCU调用 `InitUSB()` 并发送 `USB_INIT` 确认。

---

### 2. 鼠标事件处理流程

#### 通过真实鼠标输入

```
┌──────────┐                          ┌──────────┐
│  真实鼠标 │                          │  右侧MCU  │
└─────┬────┘                          └─────┬────┘
      │                                     │
      │ HID Report                          │
      ├────────────────────────────────────►│
      │                                     │
      │                                     │ 1. 解析HID Report
      │                                     │    - 提取按键状态
      │                                     │    - 提取X/Y移动
      │                                     │    - 提取滚轮值
      │                                     │
      │                                     │ 2. 生成km.*命令
      │                                     │
                                            ▼
                                      ┌──────────┐
                                      │  左侧MCU  │
                                      └─────┬────┘
                                            │
      ┌─────────────────────────────────────┤
      │ km.move(x,y) / km.left(1) / etc.    │
      │                                     │
      │                                     │ 3. 处理命令
      │                                     │
      │                                     │ 4. Mouse.move()
      │                                     │    Mouse.press()
      │                                     │
                                            ▼
                                       ┌────────┐
                                       │   PC   │
                                       └────────┘
                                       USB HID输入
```

#### 通过Python工具直接控制

```
┌──────────┐                          ┌──────────┐
│Python工具 │                          │  左侧MCU  │
└─────┬────┘                          └─────┬────┘
      │                                     │
      │ UART0: km.move(100,50)\n            │
      ├────────────────────────────────────►│
      │                                     │
      │                                     │ 1. 接收命令 (ISR触发)
      │                                     │ 2. 环形缓冲区存储
      │                                     │ 3. 命令解析
      │                                     │ 4. 互斥锁保护
      │                                     │ 5. 任务通知
      │                                     │ 6. Mouse.move(100,50)
      │                                     │
                                            ▼
                                       ┌────────┐
                                       │   PC   │
                                       └────────┘
                                       USB HID输入
```

---

### 3. 命令优先级处理

**优先级顺序**（从高到低）：

1. **调试命令** (DEBUG_*, ESPLOG_*)
2. **系统配置命令** (SERIAL_*)
3. **USB内部命令** (USB_*)
4. **鼠标控制命令** (km.*)

**特殊处理**:

- `km.move` 使用互斥锁，UART0的串口命令**优先于**UART1的USB鼠标输入
- 按键状态使用原子变量缓存，防止重复事件
- USB描述符传输期间，鼠标控制命令被阻塞（`processingUsbCommands = true`）

**源代码**:
```cpp
void processCommand(const char *command) {
    // 优先级1: 调试命令
    for (const auto &entry : debugCommandTable) {
        if (strncmp(command, entry.command, strlen(entry.command)) == 0) {
            entry.handler(command);
            return;
        }
    }

    // 优先级2: 系统配置命令
    for (const auto &entry : serial0CommandTable) {
        if (strncmp(command, entry.command, strlen(entry.command)) == 0) {
            entry.handler(command);
            return;
        }
    }

    // 优先级3: USB内部命令
    for (const auto &entry : usbCommandTable) {
        if (strncmp(command, entry.command, strlen(entry.command)) == 0) {
            entry.handler(command);
            return;
        }
    }

    // 优先级4: 鼠标控制命令（仅在非USB初始化期间）
    if (!processingUsbCommands) {
        for (const auto &entry : normalCommandTable) {
            if (strncmp(command, entry.command, strlen(entry.command)) == 0) {
                entry.handler(command);
                return;
            }
        }
    }
}
```

---

## 错误处理

### 1. 缓冲区溢出

**检测**:
```cpp
if (!serial0RingBuffer.isFull()) {
    serial0RingBuffer.push(byte);
} else {
    Serial0.println("Serial0 ring buffer overflow detected.");
}
```

**处理**:
- 输出错误消息
- 丢弃当前字节
- 继续处理

---

### 2. 无效命令

**检测**: 命令不匹配任何命令表

**处理**:
```cpp
void handleDebugcommand(const char *command) {
    Serial0.println(command);  // 回显未识别的命令
}
```

---

### 3. USB设备断开

**检测**: 右侧MCU检测到USB断开事件

**处理**:
1. 发送 `USB_GOODBYE`
2. 左侧MCU重置所有鼠标状态
3. 延迟100ms后重启MCU

```cpp
void handleUsbGoodbye(const char *command) {
    Serial0.println("USB Device disconnected. Restarting!");
    // 重置所有鼠标状态
    handleMove(0, 0);
    handleMouseButton(MOUSE_BUTTON_LEFT, false);
    handleMouseButton(MOUSE_BUTTON_RIGHT, false);
    handleMouseButton(MOUSE_BUTTON_MIDDLE, false);
    handleMouseButton(MOUSE_BUTTON_FORWARD, false);
    handleMouseButton(MOUSE_BUTTON_BACKWARD, false);
    handleMouseWheel(0);
    vTaskDelay(100);
    ESP.restart();
}
```

---

### 4. 参数解析错误

**检测**: `sscanf()` 返回值检查

**处理**:
```cpp
if (sscanf(command + strlen("SERIAL_"), "%d", &speed) == 1) {
    // 参数解析成功
} else {
    Serial0.println("Invalid SERIAL command. Expected format: SERIAL_<speed>");
}
```

---

### 5. 参数范围检查

**示例**: 串口速度范围检查
```cpp
if (speed >= 115200 && speed <= 5000000) {
    // 参数在有效范围内
} else {
    Serial0.println("Speed was out of bounds. Min: 115200, Max: 5000000.");
}
```

---

## 使用示例

### 示例1: 通过Python发送鼠标移动命令

```python
import serial

# 打开串口 (UART0)
ser = serial.Serial('COM3', 115200, timeout=1)

# 发送鼠标移动命令
ser.write(b'km.move(100,50)\n')

# 等待响应
response = ser.readline()
print(response.decode())

ser.close()
```

---

### 示例2: 完整的点击操作

```python
import serial
import time

ser = serial.Serial('COM3', 921600, timeout=1)

# 移动到目标位置
ser.write(b'km.moveto(500,300)\n')
time.sleep(0.1)

# 按下左键
ser.write(b'km.left(1)\n')
time.sleep(0.05)

# 释放左键
ser.write(b'km.left(0)\n')

ser.close()
```

---

### 示例3: 获取鼠标位置

```python
import serial

ser = serial.Serial('COM3', 115200, timeout=1)

# 请求鼠标位置
ser.write(b'km.getpos()\n')

# 读取响应: km.pos(x,y)
response = ser.readline().decode().strip()
print(response)  # 输出: km.pos(1024,768)

# 解析位置
if response.startswith('km.pos('):
    coords = response[7:-1].split(',')
    x = int(coords[0])
    y = int(coords[1])
    print(f"当前位置: X={x}, Y={y}")

ser.close()
```

---

### 示例4: 调整串口速度

```python
import serial
import time

# 初始速度115200
ser = serial.Serial('COM3', 115200, timeout=1)

# 请求提高速度到4Mbps
ser.write(b'SERIAL_4000000\n')
time.sleep(2)  # 等待MCU重新配置

# 关闭并以新速度重新打开
ser.close()
ser = serial.Serial('COM3', 4000000, timeout=1)

# 测试连接
ser.write(b'km.getpos()\n')
print(ser.readline().decode())

ser.close()
```

---

### 示例5: 启用调试输出

```python
import serial

ser = serial.Serial('COM3', 115200, timeout=1)

# 开启调试
ser.write(b'DEBUG_ON\n')

# 设置调试级别为Verbose
ser.write(b'DEBUG_5\n')

# 现在会看到详细的调试信息
while True:
    if ser.in_waiting:
        print(ser.readline().decode(), end='')

ser.close()
```

---

### 示例6: 滚轮和多按键操作

```python
import serial
import time

ser = serial.Serial('COM3', 921600, timeout=1)

# 按住右键
ser.write(b'km.right(1)\n')
time.sleep(0.5)

# 向上滚动3次
for _ in range(3):
    ser.write(b'km.wheel(3)\n')
    time.sleep(0.1)

# 释放右键
ser.write(b'km.right(0)\n')

# 点击侧键1（前进）
ser.write(b'km.side1(1)\n')
time.sleep(0.05)
ser.write(b'km.side1(0)\n')

ser.close()
```

---

### 示例7: 连续移动（游戏控制）

```python
import serial
import time

ser = serial.Serial('COM3', 4000000, timeout=1)  # 使用高速率

# 模拟圆形移动
import math

radius = 100
steps = 36  # 36步完成一个圆

for i in range(steps):
    angle = (2 * math.pi * i) / steps
    x = int(radius * math.cos(angle))
    y = int(radius * math.sin(angle))

    command = f'km.move({x},{y})\n'
    ser.write(command.encode())
    time.sleep(0.01)  # 10ms延迟

ser.close()
```

---

## 性能参数

### 通信性能

| 参数 | 值 |
|------|-----|
| UART0 最大速率 | 5 Mbps |
| UART1 速率 | 4 Mbps |
| 环形缓冲区大小 | 620 bytes |
| 命令最大长度 | 620 bytes |
| LED闪烁时间 | 25 ms |
| USB断开重启延迟 | 100 ms |
| USB初始化延迟 | 700 ms |
| 串口速度切换延迟 | 1000 ms |

### 鼠标参数

| 参数 | 值 |
|------|-----|
| 移动范围 (相对) | -127 到 +127 |
| 移动范围 (绝对) | int范围 |
| 滚轮范围 | int范围 |
| 支持按键数量 | 5 (左、右、中、侧1、侧2) |
| 按键状态缓存 | 原子变量 |

### 资源占用

| 资源 | 每MCU配置 |
|------|---------|
| Flash | 4MB Quad SPI |
| PSRAM | 2MB |
| 任务堆栈高水位 | 500 bytes 余量 |

---

## 技术特性

### 并发控制

- **互斥锁**: `km.move` 命令使用 `std::mutex` 保护
- **原子变量**: 所有按键状态使用 `std::atomic<bool>`
- **任务通知**: 使用 FreeRTOS `xTaskNotifyGive()` 触发任务

### 中断驱动

- **UART ISR**: 所有UART通信使用中断服务程序（ISR）触发
- **环形缓冲区**: 620字节环形缓冲区用于非阻塞接收
- **无轮询**: 移除所有 `vTaskDelay(1)` 实现全速日志

### USB描述符解析

- **自动解析**: 自动解析HID报告描述符，无需定制固件
- **JSON序列化**: 所有USB描述符使用JSON格式传输
- **动态适配**: 支持各种鼠标设备，包括特殊的12-bit轴鼠标

---

## 附录

### A. 命令速查表

| 命令 | 参数 | 功能 | 来源 |
|------|------|------|------|
| `km.move(x,y)` | x,y: int | 相对移动 | PC/右侧MCU |
| `km.moveto(x,y)` | x,y: int | 绝对移动 | PC |
| `km.getpos()` | - | 获取位置 | PC |
| `km.left(0/1)` | 0/1 | 左键 | PC/右侧MCU |
| `km.right(0/1)` | 0/1 | 右键 | PC/右侧MCU |
| `km.middle(0/1)` | 0/1 | 中键 | PC/右侧MCU |
| `km.side1(0/1)` | 0/1 | 侧键1 | PC/右侧MCU |
| `km.side2(0/1)` | 0/1 | 侧键2 | PC/右侧MCU |
| `km.wheel(n)` | n: int | 滚轮 | PC/右侧MCU |
| `USB_HELLO` | - | 设备连接 | 右侧MCU |
| `USB_GOODBYE` | - | 设备断开 | 右侧MCU |
| `USB_ISNULL` | - | 无设备 | 右侧MCU |
| `USB_INIT` | - | 初始化完成 | 左侧MCU |
| `DEBUG_ON` | - | 开启调试 | PC |
| `DEBUG_OFF` | - | 关闭调试 | PC |
| `DEBUG_<0-6>` | 级别 | 调试级别 | PC |
| `SERIAL_<speed>` | 波特率 | 速度设置 | PC |

### B. USB 描述符类型代码

| 代码 | 描述符类型 |
|------|-----------|
| 1 | Device Descriptor |
| 2 | Configuration Descriptor |
| 3 | String Descriptor |
| 4 | Interface Descriptor |
| 5 | Endpoint Descriptor |
| 11 | Interface Association Descriptor |
| 33 | HID Descriptor |
| 34 | HID Report Descriptor |

### C. USB 类别代码

| 代码 | 类别 |
|------|------|
| 0 | Device |
| 3 | HID (Human Interface Device) |
| 8 | Mass Storage |
| 9 | Hub |

### D. HID 子类别和协议

| 子类别 | 协议 | 说明 |
|--------|------|------|
| 0 | 0 | None |
| 1 | 1 | Keyboard (Boot Interface) |
| 1 | 2 | Mouse (Boot Interface) |

### E. 端点属性

| 属性值 | 传输类型 |
|--------|---------|
| 0 | Control |
| 1 | Isochronous |
| 2 | Bulk |
| 3 | Interrupt |

---

## 文档版本历史

| 版本 | 日期 | 变更说明 |
|------|------|---------|
| 1.0 | 2024 | 初始版本 - 完整协议文档 |

---

## 参考资料

- **源代码**: https://github.com/ssclose/MAKCM
- **Discord社区**: https://discord.gg/6TJBVtdZbq
- **USB规范**: USB 2.0 Specification
- **HID规范**: HID Usage Tables 1.12
- **ESP-IDF文档**: https://docs.espressif.com/

---

**文档结束**
