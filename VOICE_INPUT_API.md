# 小狼毫输入法语音输入接口文档

## 概述

本文档描述了为小狼毫输入法（Weasel）添加的语音输入专用接口。这些接口允许外部应用程序（如语音输入法）直接将文本发送到目标应用程序，绕过传统的按键模拟方式。

## 修改内容

### 1. 新增 IPC 命令

```cpp
WEASEL_IPC_SEND_STRING  // 语音输入直接上屏
```

### 2. 新增接口方法

#### Client 端（应用程序调用）

```cpp
// include/WeaselIPC.h
class Client {
public:
    // 直接发送文本上屏（用于语音输入）
    bool SendString(const std::wstring& text);
};
```

#### Server 端（输入法处理）

```cpp
// include/WeaselIPC.h
struct RequestHandler {
    // 提交文本到指定会话
    virtual void CommitText(const std::wstring& text, DWORD session_id) {}
};
```

## 使用流程

### 基本使用步骤

```cpp
#include <WeaselIPC.h>

// 1. 创建客户端对象
weasel::Client client;

// 2. 连接到输入法服务
if (!client.Connect()) {
    // 连接失败处理
    return false;
}

// 3. 启动会话
client.StartSession();

// 4. 发送语音识别的文本
std::wstring recognizedText = L"这是一段语音识别的文本";
if (client.SendString(recognizedText)) {
    // 发送成功
} else {
    // 发送失败处理
}

// 5. 结束会话
client.EndSession();

// 6. 断开连接
client.Disconnect();
```

### 完整示例

```cpp
#include <WeaselIPC.h>
#include <iostream>

class VoiceInputClient {
private:
    weasel::Client client_;
    bool connected_;

public:
    VoiceInputClient() : connected_(false) {}
    
    ~VoiceInputClient() {
        if (connected_) {
            Disconnect();
        }
    }
    
    bool Connect() {
        if (client_.Connect()) {
            connected_ = true;
            client_.StartSession();
            return true;
        }
        return false;
    }
    
    void Disconnect() {
        if (connected_) {
            client_.EndSession();
            client_.Disconnect();
            connected_ = false;
        }
    }
    
    bool SendText(const std::wstring& text) {
        if (!connected_) {
            std::wcerr << L"未连接到输入法服务" << std::endl;
            return false;
        }
        
        if (text.empty()) {
            return true;  // 空文本直接返回成功
        }
        
        bool result = client_.SendString(text);
        if (!result) {
            std::wcerr << L"发送文本失败" << std::endl;
        }
        return result;
    }
    
    bool IsConnected() const {
        return connected_;
    }
};

// 使用示例
int main() {
    VoiceInputClient voiceClient;
    
    // 连接到小狼毫输入法
    if (!voiceClient.Connect()) {
        std::wcerr << L"无法连接到小狼毫输入法服务" << std::endl;
        return 1;
    }
    
    std::wcout << L"成功连接到小狼毫输入法" << std::endl;
    
    // 模拟语音识别结果
    std::vector<std::wstring> voiceResults = {
        L"你好",
        L"这是一段测试文本",
        L"小狼毫输入法语音输入测试"
    };
    
    // 发送识别结果
    for (const auto& text : voiceResults) {
        std::wcout << L"发送: " << text << std::endl;
        if (voiceClient.SendText(text)) {
            std::wcout << L"发送成功" << std::endl;
        } else {
            std::wcerr << L"发送失败" << std::endl;
        }
    }
    
    // 断开连接
    voiceClient.Disconnect();
    std::wcout << L"已断开连接" << std::endl;
    
    return 0;
}
```

## 接口详细说明

### Client::SendString

**功能**：直接发送文本到输入法，由输入法负责上屏

**参数**：
- `text`：要发送的宽字符串（Unicode）

**返回值**：
- `true`：发送成功
- `false`：发送失败（可能是未连接、会话未启动等）

**注意事项**：
1. 调用前必须先调用 `Connect()` 和 `StartSession()`
2. 文本会直接上屏，不会经过 Rime 的拼音处理流程
3. 支持中英文混合文本

### RequestHandler::CommitText

**功能**：服务端处理文本提交的回调接口

**参数**：
- `text`：要提交的文本
- `session_id`：目标会话 ID

**实现说明**：
- 在 `RimeWithWeaselHandler` 中实现
- 通过 TSF（Text Service Framework）将文本插入到目标应用程序
- 无需创建合成（composition），直接上屏

## 架构流程

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   语音输入法     │────▶│   Weasel IPC     │────▶│   Weasel 服务   │
│  (VoiceIME)     │     │   Client         │     │   (Server)      │
└─────────────────┘     └──────────────────┘     └────────┬────────┘
                                                          │
                                                          ▼
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   目标应用程序   │◀────│   TSF 层         │◀────│  RimeWithWeasel │
│  (记事本/Word等) │     │  (上屏文本)       │     │  (CommitText)   │
└─────────────────┘     └──────────────────┘     └─────────────────┘
```

## 与传统 SendInput 方式的对比

| 特性 | SendInput 模拟按键 | Weasel IPC 直接上屏 |
|------|-------------------|-------------------|
| 实现方式 | 模拟键盘输入 | 通过输入法框架直接插入文本 |
| 中英文混合 | 容易出错（切换输入法状态） | 完美支持 |
| 特殊字符 | 可能无法输入 | 完全支持 |
| 速度 | 慢（需要逐字输入） | 快（一次性上屏） |
| 依赖 | 依赖键盘布局和输入法状态 | 依赖输入法服务运行 |
| 兼容性 | 部分应用可能不响应 | 所有支持 TSF 的应用 |

## 编译说明

### 需要编译的文件

修改以下7个文件后，需要重新编译小狼毫输入法：

1. `include/WeaselIPC.h`
2. `WeaselIPC/WeaselClientImpl.h`
3. `WeaselIPC/WeaselClientImpl.cpp`
4. `WeaselIPCServer/WeaselServerImpl.h`
5. `WeaselIPCServer/WeaselServerImpl.cpp`
6. `include/RimeWithWeasel.h`
7. `RimeWithWeasel/RimeWithWeasel.cpp`

### 编译命令

```bash
# 使用 build.bat
.\build.bat arm64 installer

# 或使用 xmake
.\xbuild.bat arm64 installer
```

## 故障排除

### 常见问题

**1. 连接失败**
- 检查小狼毫输入法服务是否运行
- 检查管道名称是否正确

**2. 发送失败**
- 确保已调用 `StartSession()`
- 检查会话 ID 是否有效

**3. 文本没有上屏**
- 确保目标应用程序有输入焦点
- 检查输入法是否正确安装

### 调试方法

```cpp
// 启用调试日志
#include <logging.h>

// 在代码中添加日志
DLOG(INFO) << "Sending text: " << text;
```

## 版本历史

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.0 | 2026-03-05 | 初始版本，添加 WEASEL_IPC_SEND_STRING 命令 |

## 参考

- [小狼毫输入法官方仓库](https://github.com/rime/weasel)
- [TSF (Text Service Framework) 文档](https://docs.microsoft.com/en-us/windows/win32/tsf/text-services-framework)
- [Rime 输入法引擎](https://rime.im/)
