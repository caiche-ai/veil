# 本地隐私识别评测

本工具复用 Presidio 的中文归档，调用 Veil 真实引擎、JSON 脱敏/还原及加密 SQLite 存储。完整结果与解释见 [实测报告](../../reports/privacy-eval/2026-10-28/REPORT.md) 和 [定位分析](../../reports/privacy-eval/2026-10-28/ANALYSIS.md)。生产规则不在本任务中修改。

```bash
SDKROOT=/Library/Developer/CommandLineTools/SDKs/MacOSX26.5.sdk cargo build --release --offline -p veil-store --example privacy_probe
python3 scripts/privacy_eval/run.py --presidio /Users/caic/code/presidio --output reports/privacy-eval/new-run
python3 scripts/privacy_eval/render.py reports/privacy-eval/new-run
python3 scripts/privacy_eval/verify.py reports/privacy-eval/new-run
```

输出目录必须不存在。Python 仅需标准库，Rust 依赖沿用 workspace。SDKROOT 为本机链接器兼容性设置，其他机器可使用其正常 SDK。`--duration-ms` 控制每条件循环窗口，默认 1000 ms，并发条件至少 3000 ms。实际总时间还包含全量计分、冷映射写入和原始样本压缩，不止窗口时间。

- `run.py`：加载并校验全部数据、评分、规则消融、密度扫描和线程并发诊断。
- `render.py`：从 JSON 重建数值报告，不重新运行识别。
- `verify.py`：用中文/emoji 字符范围、边界错、负例验证计分语义，复核全部预测的字节/字符范围、还原结果及原始延迟分位数。
- `crates/veil-store/examples/privacy_probe.rs`：只负责调用生产代码，JSONL 交互；不修改真实配置、不访问网络、不使用用户数据库。

每条效果记录保存全部标注、完整预测和错误片段；终端只打印组名和进度。原始数据包含合成隐私值，结果在本地保存。性能原始记录只保存调用耗时，条件中保留负载哈希；可按固定 seed 从来源数据重建。长文本效果集与性能六条抽样集的口径不同，不能混用。

默认规则以 `builtin_ruleset()` 为准。邮箱开启为独立消融，不自动启用其他缺失类别。精确实体与敏感字符覆盖分别报告，不把规则设计上的整字段替换全部当成隐私泄漏。所有调用均是本机离线管线；线程测试不是实际 HTTP 服务容量验收。


## Codex 真实模型补充测试

使用现有 ChatGPT 登录，通过独立临时 SQLite 和项目真实代理链路串行执行 31 轮，覆盖单轮、多轮记忆/更正、1,000/5,000/30,000 字符文档。需要允许本机端口与网络，并由 Codex 保存用于 resume 的测试会话。

```bash
SDKROOT=/Library/Developer/CommandLineTools/SDKs/MacOSX26.5.sdk cargo build --release --offline -p veil --example codex_e2e_gateway
python3 scripts/privacy_eval/run_codex_e2e.py --output reports/privacy-eval/new-codex-e2e
python3 scripts/privacy_eval/summarize_codex_e2e.py reports/privacy-eval/new-codex-e2e
python3 scripts/privacy_eval/verify_codex_e2e.py reports/privacy-eval/new-codex-e2e
```

`--smoke` 仅运行一条真实模型请求；正式报告使用完整 31 轮。观察记录器会保存合成测试正文、映射与原始响应，排除凭据头部，额外引入诊断开销。模型及 reasoning effort 沿用用户顶层配置，不修改基础配置文件。

将全部结果合为一个可离线打开的 HTML，并嵌入全量原始结果 ZIP：

```bash
python3 scripts/privacy_eval/consolidate.py reports/privacy-eval/2026-10-28 --output reports/privacy-eval/2026-10-28-complete.html
```


## 完整客户端 TTFT

通过 Codex App Server 的 `turn/start` → `item/agentMessage/delta` 在同一个测试客户端、同一单调时钟采集时间；区别于 Veil 收到请求到首文本的代理侧指标。另列建会话及首次进程启动等待，不包含桌面 UI 绘制。

```bash
python3 scripts/privacy_eval/run_codex_ttft.py --prior reports/privacy-eval/2026-10-28/codex-e2e --output reports/privacy-eval/new-codex-ttft
python3 scripts/privacy_eval/summarize_codex_ttft.py reports/privacy-eval/new-codex-ttft
```

本轮曾因容量繁忙和格式校验异常分批继续。完整报告保留三批原始日志、采样脚本快照及所有失败；`merge_codex_ttft.py` 针对本轮三批记录合并，不是通用自动重试器。`--start-index` 用于明确继续场景，若处于多轮会话中须从该会话首轮重新建立历史。
