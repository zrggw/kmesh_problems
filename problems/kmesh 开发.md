ipsec部分功能的测试代码编写

基于我对 Kmesh IPSec 代码的分析，我为您制定一个完整的 IPSec 测试方案：

## Kmesh IPSec 测试方案

### 1. **单元测试层 (Unit Tests)**

#### 1.1 IPSec Handler 核心功能测试
**文件**: `pkg/controller/encryption/ipsec/ipsec_handler_test.go`

**测试覆盖**:
- `LoadIPSecKeyFromFile()` - 密钥文件加载
- `generateIPSecKey()` - 密钥生成算法
- `createStateRule()` - XFRM State 规则创建 (Mock Netlink)
- `createPolicyRule()` - XFRM Policy 规则创建 (Mock Netlink)
- `CreateXfrmRule()` - 完整规则创建流程
- `Clean()` - 清理 XFRM 规则
- `Flush()` - 刷新所有规则

**Mock 策略**:
```go
// 使用 gomock 或 testify/mock 模拟 netlink 操作
type MockNetlink interface {
    XfrmStateAdd(*netlink.XfrmState) error
    XfrmPolicyAdd(*netlink.XfrmPolicy) error
    // ...
}
```

#### 1.2 IPSec Controller 测试
**文件**: `pkg/controller/encryption/ipsec/ipsec_controller_test.go`

**测试覆盖**:
- `handleKNIAdd()` - 节点新增处理
- `handleKNIUpdate()` - 节点更新处理
- `handleKNIDelete()` - 节点删除处理
- `handleOneNodeInfo()` - 单节点处理逻辑
- `generalKNIMapKey()` - LPM Key 生成
- `handleTc()` - TC 程序管理

**Mock 依赖**:
- Kubernetes Client
- eBPF Map 操作
- Netlink 操作

### 2. **集成测试层 (Integration Tests)**

#### 2.1 端到端 IPSec 流程测试
**文件**: `test/integration/ipsec_integration_test.go`

**测试场景**:
- 完整的密钥加载 → 规则创建 → 流量加密/解密流程
- 多节点场景下的 IPSec 规则协调
- 密钥轮换和更新机制
- 故障恢复和清理机制

#### 2.2 文件监听测试
**测试内容**:
- IPSec 密钥文件变更监听
- 热更新机制验证
- 文件格式验证和错误处理

### 3. **eBPF 程序测试扩展**

#### 3.1 增强现有 TC 测试
**扩展文件**: 
- tc_mark_encrypt_test.c
- tc_mark_decrypt_test.c

**新增测试用例**:
```c
// 测试不同 CIDR 场景
TEST("tc_encrypt_multiple_cidrs")
TEST("tc_decrypt_invalid_mark") 
TEST("tc_encrypt_lpm_lookup_miss")
```

#### 3.2 新增 IPSec 相关 eBPF 测试
**文件**: `test/bpf_ut/ipsec_policy_test.c`

**测试内容**:
- LPM Trie Map 查找逻辑
- Mark 设置的正确性
- 不同协议下的包处理

### 4. **性能测试层**

#### 4.1 IPSec 操作性能测试
**文件**: `test/performance/ipsec_perf_test.go`

**测试指标**:
- XFRM 规则创建/删除的延迟
- 大量节点场景下的规则管理性能
- 密钥生成算法的性能
- 内存使用情况

#### 4.2 并发安全测试
**测试场景**:
- 并发密钥更新
- 并发节点信息处理
- 竞态条件检测

### 5. **模拟测试环境**

#### 5.1 容器化测试环境
**文件**: `test/testdata/ipsec/docker-compose.yml`

**环境组件**:
- 模拟的 Kubernetes 集群
- 多个 Kmesh 节点
- 网络命名空间隔离
- IPSec 流量模拟器

#### 5.2 测试数据和配置
**目录结构**:
```
test/testdata/ipsec/
├── configs/
│   ├── valid_key.json
│   ├── invalid_key.json
│   └── expired_key.json
├── crd_samples/
│   └── kmesh_node_info.yaml
└── network_configs/
    └── test_topology.yaml
```

### 6. **错误场景测试**

#### 6.1 异常处理测试
- 无效密钥格式处理
- Netlink 操作失败处理
- 网络不可达场景
- 权限不足场景

#### 6.2 边界条件测试
- 空 CIDR 列表
- 极大数量的节点
- 密钥长度边界值
- SPI 值边界测试

### 7. **测试工具和辅助功能**

#### 7.1 IPSec 状态检查工具
**文件**: `test/tools/ipsec_inspector.go`
- 检查 XFRM 规则状态
- 验证密钥正确性
- 网络连通性测试

#### 7.2 测试数据生成器
**文件**: `test/tools/ipsec_data_generator.go`
- 生成测试用的密钥文件
- 创建模拟的节点信息
- 网络拓扑配置生成

### 8. **CI/CD 集成**

#### 8.1 自动化测试流水线
```yaml
# .github/workflows/ipsec_tests.yml
- name: IPSec Unit Tests
- name: IPSec Integration Tests  
- name: IPSec eBPF Tests
- name: IPSec Performance Tests
```

#### 8.2 测试覆盖率要求
- 单元测试覆盖率 > 85%
- 集成测试覆盖主要流程
- 错误场景覆盖率 > 70%

### 9. **实施优先级**

**Phase 1** (高优先级):
1. IPSec Handler 核心功能单元测试
2. 密钥生成和文件加载测试
3. 基本的 Mock Netlink 测试

**Phase 2** (中优先级):
1. IPSec Controller 集成测试
2. eBPF 程序测试扩展
3. 错误场景测试

**Phase 3** (低优先级):
1. 性能测试套件
2. 端到端集成测试
3. 测试工具完善

### 10. **技术选型建议**

**测试框架**:
- Go 标准测试库 + testify
- gomock 用于 Mock 依赖
- Ginkgo/Gomega (可选，用于 BDD 风格测试)

**eBPF 测试**:
- 继续使用现有的 Cilium 测试框架
- 扩展现有的测试宏和工具

**容器化测试**:
- Docker + Kind 用于 Kubernetes 环境模拟
- Testcontainers-go 用于集成测试

这个方案涵盖了从单元测试到端到端测试的完整测试体系，可以确保 IPSec 功能的可靠性和稳定性。您希望我详细展开其中哪个部分，或者开始实施某个特定的测试模块吗？
