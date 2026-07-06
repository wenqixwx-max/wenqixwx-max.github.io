---
title: "关于jvmti怎么实现动态修改agent启动阶段（jvmtiPhase）"
date: "2026-07-06T12:02:17.350Z"
excerpt: "如何动态agent实现加载非当前阶段的能力"
tags: ["代码", "jvmti", "java"]
---这种通过在运行时（Live 阶段）动态修改 JVM 内部状态变量 `JvmtiEnvBase::_phase`，以绕过 `AddCapabilities` 阶段限制的技术，在高级 JVM 诊断工具、APM 探针以及逆向工程中是一种常见底层 Hook 手段。

为了满足准确性与结构化的要求，以下对该方案的原理、潜在风险以及实现代码进行了规范化梳理。

---

## 核心技术原理

JVMTI（JVM Tool Interface）规定某些核心能力（如 `can_access_local_variables`、`can_tag_objects` 等）必须在 JVM 启动阶段（`JVMTI_PHASE_ONLOAD`）开启。一旦 JVM 进入运行阶段（`JVMTI_PHASE_LIVE`），`AddCapabilities` 函数内部会校验全局状态变量 `JvmtiEnvBase::_phase`，若不为 `ONLOAD` 则拒绝执行并返回错误码。

该方案的规避原理为：

1. **内存定位**：通过遍历 `jvm.dll` 的 `.data` 段，搜寻当前值为 `JVMTI_PHASE_LIVE`（整数值通常为 `4`）的内存地址。
2. **状态欺骗（Race Mitigation）**：由于可能存在多个符合条件的地址，采用“动态修改 -> 尝试注入 -> 校验结果 -> 恢复状态”的试探法（Probe）来精确锁定真实的 `_phase` 变量地址。
3. **绕过校验**：在锁定地址后，通过修改内存数据暂时将该变量的值写入为 `JVMTI_PHASE_ONLOAD`（整数值通常为 `1`），调用 `AddCapabilities` 完成能力注册，随后立即将状态恢复为 `JVMTI_PHASE_LIVE`，确保 JVM 核心逻辑不因状态错乱而崩溃。

---

## 关键风险与注意事项

在生产或严苛环境中使用此方法，必须注意以下技术风险：

* **多线程竞争条件（Race Condition）**：在修改 `_phase` 为 `ONLOAD` 的极短时间内，如果有其他 JVM 内部线程读取了该变量，可能会触发意料之外的逻辑错误（例如触发了只有 OnLoad 阶段才允许的初始化逻辑）。
* **内存保护属性（VirtualProtect）**：虽然 `.data` 段通常是可读写的（`PAGE_READWRITE`），但部分安全增强型 JVM 或操作系统保护机制可能会将其重映射为只读。修改前应使用 `VirtualProtect` 显式确保可写性。
* **JVM 跨版本兼容性**：
* **符号名与结构变动**：不同 OpenJDK 分支（如 Oracle JDK、Corretto、Dragonwell）或不同主版本（Java 8 vs Java 17 vs Java 21），`JvmtiEnvBase::_phase` 的内存排布或枚举定义可能发生变化。
* **硬编码值变化**：需确保 `JVMTI_PHASE_LIVE` 和 `JVMTI_PHASE_ONLOAD` 的枚举整型值在目标 JVM 中未发生改变。



---

## 规范化代码实现与文档

以下是对您提供的代码进行的规范化重构，加入了关键的内存保护处理（`VirtualProtect`）以确保在 Windows 环境下的写入安全性，同时精简了冗余逻辑。

### 规范化函数实现

```cpp
#include <windows.h>
#include <jvmti.h>
#include <vector>
#include <iostream>

// 辅助函数：安全修改内存阶段值（包含内存保护属性恢复）
bool SafeWritePhase(jvmtiPhase* address, jvmtiPhase newValue) {
    DWORD oldProtect;
    // 显式确保内存页具备可写权限
    if (!VirtualProtect(address, sizeof(jvmtiPhase), PAGE_EXECUTE_READWRITE, &oldProtect)) {
        return false;
    }

    *address = newValue;

    // 恢复原有的内存保护属性
    VirtualProtect(address, sizeof(jvmtiPhase), oldProtect, &oldProtect);
    return true;
}

jvmtiPhase* FindJvmtiPhasePtr(jvmtiEnv* jvmti) {
    if (!jvmti) return nullptr;

    // 1. 定位 jvm.dll 的模块句柄
    HMODULE hJvm = nullptr;
    if (!GetModuleHandleExA(GET_MODULE_HANDLE_EX_FLAG_FROM_ADDRESS | 
                            GET_MODULE_HANDLE_EX_FLAG_UNCHANGED_REFCOUNT,
                            (LPCSTR)jvmti->functions->GetPhase, 
                            &hJvm)) {
        return nullptr;
    }

    // 2. 解析 PE 头，定位主 .data 段
    auto* dosHdr = reinterpret_cast<IMAGE_DOS_HEADER*>(hJvm);
    auto* ntHdr = reinterpret_cast<IMAGE_NT_HEADERS*>(reinterpret_cast<BYTE*>(hJvm) + dosHdr->e_lfanew);
    auto* section = IMAGE_FIRST_SECTION(ntHdr);

    BYTE* dataBase = nullptr;
    SIZE_T dataSize = 0;

    for (WORD i = 0; i < ntHdr->FileHeader.NumberOfSections; i++) {
        DWORD chars = section[i].Characteristics;
        // 筛选可读写且不可执行的段（标准 .data 段特征）
        if ((chars & IMAGE_SCN_MEM_READ) && 
            (chars & IMAGE_SCN_MEM_WRITE) && 
            !(chars & IMAGE_SCN_MEM_EXECUTE)) {
            
            // 选取特征最明显、体积最大的数据段
            if (section[i].SizeOfRawData > dataSize) {
                dataBase = reinterpret_cast<BYTE*>(hJvm) + section[i].VirtualAddress;
                dataSize = section[i].SizeOfRawData;
            }
        }
    }

    if (!dataBase || dataSize < sizeof(jvmtiPhase)) return nullptr;

    // 3. 扫描匹配特征值为 JVMTI_PHASE_LIVE 的候选地址
    std::vector<jvmtiPhase*> candidates;
    for (SIZE_T off = 0; off + sizeof(jvmtiPhase) <= dataSize; off += sizeof(jvmtiPhase)) {
        auto* ptr = reinterpret_cast<jvmtiPhase*>(dataBase + off);
        if (*ptr == JVMTI_PHASE_LIVE) {
            candidates.push_back(ptr);
        }
    }

    // 4. 动态探针验证：伪装 ONLOAD 阶段并尝试添加能力
    jvmtiCapabilities probeCaps;
    std::memset(&probeCaps, 0, sizeof(probeCaps));
    probeCaps.can_access_local_variables = 1; // 选取典型的仅 ONLOAD 支持的能力

    jvmtiPhase* targetPtr = nullptr;
    for (jvmtiPhase* ptr : candidates) {
        // 尝试临时修改为 ONLOAD
        if (!SafeWritePhase(ptr, JVMTI_PHASE_ONLOAD)) continue;

        // 调用测试
        jvmtiError err = jvmti->AddCapabilities(&probeCaps);

        // 无论何种结果，必须立即恢复 LIVE 状态，降低对 JVM 运行的影响
        SafeWritePhase(ptr, JVMTI_PHASE_LIVE);

        if (err == JVMTI_ERROR_NONE) {
            targetPtr = ptr;
            break;
        }
    }

    return targetPtr;
}

```

### 执行流程演进规范

1. **扫描识别**：代码筛选出所有值为 `4` (`JVMTI_PHASE_LIVE`) 的地址，缩小目标范围。
2. **精确过滤**：因为盲目修改非 `_phase` 内存会导致不可预知的崩溃，通过 `AddCapabilities` 返回值（`JVMTI_ERROR_NONE`）作为判定真伪的原子断言。
3. **后续使用**：一旦 `FindJvmtiPhasePtr` 返回了正确的内存指针，后续若需要继续开启其他能力（如 `can_tag_objects`），直接重复 **“修改为 ONLOAD -> AddCapabilities -> 恢复为 LIVE”** 的三步操作即可。
