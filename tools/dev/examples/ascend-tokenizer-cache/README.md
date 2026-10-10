# Ascend tokenizer-cache backend integration

| Delivery | Status |
| --- | --- |
| Backend startup and offline verification helpers | Implemented and locally checked |
| semantic-router runtime/policy changes | None; backend integration example |
| End-to-end integration and hardware acceptance | Pending |

## Component integration

Merge this provider entry into a maintained v0.3 recipe. The served model alias
must match the backend's --served-model-name option. Router containers need a
reachable backend address in place of the loopback example.

~~~yaml
providers:
  models:
    - name: ascend-local
      provider_model_id: local-model
      api_format: openai
      backend_refs:
        - name: ascend-primary
          endpoint: 127.0.0.1:8000
          protocol: http
          provider: vllm
          weight: 1
~~~

Select ascend-local through the recipe's existing modelRefs. This fragment is
not a complete recipe. Classifier and embedding tokenization in the model
runtime keep their existing behavior.

## Backend contract

| Item | Value |
| --- | --- |
| vLLM-Ascend source | 4ea839b813fecbd1aba3affa63e3c896e75b123b |
| Paired vLLM reference | ced6857afa0ea7b2e3f0846a62e1394e90f15607 |
| Cache option | additional_config.tokenizer_cache_gb; default 0 |
| Local checks | Python 3.9.6, stdlib; no image or inference service used |
| Hardware, throughput, quality | Not run |
| Task branch | xxx_tokenizer_cache |
| Component base SHA | 5c3869fc7b8c7585f4a0dcbc89a4da1ccb34d1c9 |
| Delivery SHA | Commit containing this example; inspect with git log -1 -- . |

Install the paired sources in an Ascend environment with its compatible
Ascend SDK, torch-npu and model dependencies. source_contract.json records fingerprints
of the relevant installed patch/config/renderer files. The launcher verifies
these files before execution. This detects this specific interface mismatch;
it does not certify the complete environment or all model templates.

Source: [Ascend frontend cache](https://github.com/vllm-project/vllm-ascend/pull/16298).
The patch remains an unmerged dependency. Engine CLI arguments belong to the
backend startup command; do not insert them into an OpenAI request body.

## Commands

Dry-run command construction without installing the backend:

~~~bash
python3 launch_backend.py --model Qwen/Qwen3-0.6B --cache-gb 4 --dry-run \
  -- --served-model-name local-model --max-model-len 8192
python3 -m unittest -v test_example.py
~~~

After preparing the pinned backend, start OFF and ON servers in separate
terminals, with separately allocated resources:

~~~bash
python3 launch_backend.py --model Qwen/Qwen3-0.6B --cache-gb 0 --port 8000 \
  -- --served-model-name local-model --max-model-len 8192
python3 launch_backend.py --model Qwen/Qwen3-0.6B --cache-gb 4 --port 8001 \
  -- --served-model-name local-model --max-model-len 8192
python3 verify_tokens.py --baseline http://127.0.0.1:8000 \
  --cached http://127.0.0.1:8001 --model local-model --rounds 3 \
  --output token-conformance.json
~~~

The service commands above are provided for hardware execution and were not
run locally. The comparison requires complete token-ID equality across OFF,
ON and repeated synthetic inputs. Report a failure on different IDs,
nondeterminism or malformed responses. Its per-call timing is diagnostic;
sequential /tokenize calls are not an inference load benchmark.

For workload evaluation, keep the same model, template, API-process count,
hardware, arrival trace and KV-routing policy. Measure tokenizer CPU time,
process RSS, TTFT, completion latency and success-task cost in the existing
benchmark harness. Include tools, truncation and template changes in model
conformance before deployment. No performance result is supplied here.

## Boundaries

The cache stores text-segment-to-token-ID results in host memory. Its budget
is an estimate per tokenizer instance rather than an RSS limit. API-process
count and DP-rank count are separate. A cache hit does not establish physical
KV residency, reduce logical request token count, or enable cross-model KV
reuse. Cache-OFF preserves the budget switch; wrappers still require the
paired renderer interfaces.

Public offline checks cover launcher argument preservation, invalid budgets,
installed-file mismatch and complete-ID comparison failure. These checks do
not substitute for native Router builds, real backend conformance, cluster
tests, generation/eviction recovery or P/D acceptance.

## Native render and serving conformance (2026-10-10)

`verify_serving.py` supplements `/tokenize` checks with the actual chat-render
protocol and, with `--serve`, generation request ingress. It compares complete
OFF/ON token IDs, repetition, growing conversations, Unicode, tenant salts and
tools. Serving must return the same `prompt_token_ids` and prompt usage.

~~~bash
python3 -m unittest -v test_example.py test_serving.py
python3 verify_serving.py --baseline http://127.0.0.1:8000 \
  --cached http://127.0.0.1:8001 --model local-model --rounds 3 --serve \
  --output render-serving-conformance.json
~~~

`BACKEND_API_KEY`, if required, is read from the environment. Never commit it.
The six harness tests use local HTTP fixture servers, not an inference model.
The serving command requires prepared OFF/ON backends and is **not executed**
on NPU in this delivery. Output timing is diagnostic only; the harness does not
measure generation quality, inference throughput, or full Agent success.
Generation text hashes are recorded without treating nondeterministic text as
a tokenizer-equivalence failure. Real workload evaluation must use held-out
Agent tasks and include retries, failures, complete latency and total cost.

## Verification plan — 2026-10-11

| Evidence boundary | Current scope |
| --- | --- |
| This change | Repository-specific acceptance plan; no new production-routing implementation |
| Existing software checks | Historical native/fixture results above; not a real model or NPU qualification |
| Live acceptance | Pending; no hardware has been allocated for this plan |
| Release | Correctness → native ingress → fault recovery → isolated performance → full-task evaluation |

先验收 **token ID 一致性**，再验收 **事件索引真实性**，最后验收 **任务收益**。三者不能互相替代。
Host 分词缓存、Router whole-prompt L0、引擎物理 KV residency 是三个独立对象；逻辑 token 数不因编码缓存命中而下降。

### Installation prerequisites and immutable experiment record

| Field | Required record |
| --- | --- |
| Repository | Base SHA from this README; task branch `xxx_tokenizer_cache`; tested HEAD from `git rev-parse HEAD` |
| Paired engine | Ascend `4ea839b813fecbd1aba3affa63e3c896e75b123b`; vLLM `ced6857afa0ea7b2e3f0846a62e1394e90f15607`; check whether each dependency has merged at execution time |
| Container | Full registry/tag and actually pulled digest; additional dependencies, OS/architecture, Python/Rust/Go versions; no invented image or digest |
| Model | Weight revision/digest, tokenizer files, chat-template digest, LoRA revision, dtype/quantization and context length |
| Hardware | Allocated physical cards, device model, driver/runtime versions, interconnect, TP/PP/DP topology, API-process count |
| Runtime | Complete launcher command and additional-config, routing config digest, connector/version, event endpoints, seeds and TTL/capacity |
| Workload | Dataset revision/hash, task/repository/time split, prompt/output lengths, arrivals/concurrency, warm-up, timeout and retry budget |

Run the installed-file contract check in `launch_backend.py` before starting workers. Keep credentials in environment variables. Public artifacts contain synthetic inputs and aggregate evidence; keep restricted raw inputs and results local. Model serving, load tests and NPU occupancy need explicitly assigned resources; none are started by this document.

### Shared backend conformance commands

Run from this example directory. All real-service commands below are **plans, not completed tests**. OFF and ON need separately allocated comparable resources; a sequential crossover on the same cards must restart/reset between conditions and alternate order. Keep KV policy, API-process count, weights and template identical.

~~~bash
python3 -m unittest -v test_example.py test_serving.py
python3 launch_backend.py --model Qwen/Qwen3-0.6B --cache-gb 0 --port 8000 \
  -- --served-model-name local-model --max-model-len 8192
python3 launch_backend.py --model Qwen/Qwen3-0.6B --cache-gb 4 --port 8001 \
  -- --served-model-name local-model --max-model-len 8192
python3 verify_tokens.py --baseline http://127.0.0.1:8000 \
  --cached http://127.0.0.1:8001 --model local-model --rounds 3 \
  --output token-conformance.json
python3 verify_serving.py --baseline http://127.0.0.1:8000 \
  --cached http://127.0.0.1:8001 --model local-model --rounds 3 --serve \
  --output render-serving-conformance.json
~~~

`Qwen/Qwen3-0.6B` is an inexpensive text smoke-test example, not DeepSeek/DP16 certification. Use the actual production weights/template for final conformance. `verify_serving.py` calls backend render/generate, **not the entire upstream router/Agent**. Capture actual upstream ingress separately under the cases below. Check every complete token ID, not only count; reject malformed IDs, changed `usage.prompt_tokens`, and render/serving mismatch. Extend the live corpus to empty/truncated inputs, tools and unsupported modalities without silently skipping failures. Output text hashes are diagnostic; generation quality requires an independent task judge.

### Repository-specific cases

| ID | Test object | Input / failure / execution | Independent oracle and pass condition | Current state |
| --- | --- | --- | --- | --- |
| SR-01 | 模型选择与后端契约 | 同一任务固定配置与模型池；请求通过 Envoy ExtProc，记录最终模型别名、backend、trace ID | 实际 backend 的 served-model-name 与决策一致；完整消息、tools、salt、采样参数不被遗漏 | 完整 Envoy 链待执行 |
| SR-02 | 分词缓存隔离 | 相同输入分别开关后端缓存；再改变 template、模型、tenant salt、工具与多模态输入 | 完整 token IDs 与直接后端一致；缓存只改变 CPU 编码开销，不改变选模或 token 用量 | 已有后端 fixture；实服务待测 |
| SR-03 | 会话连续性 | 同 session 多轮修复；工具消息很长；下一轮要求升级模型；同时并发不同 tenant | 保留最近用户目标、完整工具 ID 配对；上下文预算合规；换模型重新渲染，不借用原模型 KV | 待执行 |
| SR-04 | 反馈学习与泄漏 | 按任务族、repo、时间分开训练/验证；先记录选模，再写质量反馈；模型版本变化 | 测试标签不参与当次选模；分类器/Bandit开销计入总成本；旧画像按版本隔离 | 待执行 |
| SR-05 | 错误与限流 | 429、5xx、超时、无效JSON、首token前后断流、后端下线 | 重试有上限；已开始的stream不拼接第二个答案；能力和权限约束仍生效 | 待执行 |
| SR-06 | 选模价值 | 固定Runtime，固定强模型/固定主力/简单规则/语义路由四组；相同任务和工具预算 | 成功率非劣界预先登记；成功任务成本包含失败和重试；任务完成P95不恶化超过业务阈值 | 待执行 |

### Failure reproduction and result recording

Each case has five fields in the actual run record: **phenomenon**, **steps** (fixed versions/input/flags), **expected versus observed**, **confirmed cause versus hypothesis**, **fix and rerun result**. Preserve the first failing trace and every unresolved/skipped condition. Assign the request/attempt ID before ingress; attach route ID, model/template identity, worker incarnation, endpoint/rank, event sequence and engine response to the same trace. A recovered request does not establish a repaired root cause.

### Performance and quality gates (proposed; not achieved)

| Gate | Required comparison / measurement |
| --- | --- |
| G0 — correctness | Zero full-token-ID mismatches, unauthorized cross-identity credit, unsafe handoff and resource leaks in the declared deterministic corpus; explicitly count unknown/skipped cases |
| G1 — native ingress | Actual router/Agent ingress reaches the registered model/endpoint/rank with unchanged semantics; standalone backend probes cannot pass this gate |
| G2 — recovery | Scripted gaps, clear/restart, cancellation, saturation and malformed events revoke trust correctly; no exact claim before validated reconciliation |
| G3 — isolated benefit | Cache OFF/ON × routing policy factorial; fixed hardware/trace/quality; cold/warm/churn separately; measure CPU encode, RSS, actual KV reuse, TTFT P99, completion P95, SLO goodput |
| G4 — task value | Fixed strong, fixed main and simple rules as controls; held-out tasks; total spend including failed attempts, retries, classifier and fusion divided by successful tasks; quality and latency constraints decided before running |

Use at least five paired seeds/arrival traces for the initial comparison and report paired bootstrap 95% confidence intervals. This repeat count and all performance thresholds are **project proposals**, not an upstream guarantee; compute the necessary sample size from variance and the business non-inferiority margin. Zero observed failures does not prove impossibility outside the tested corpus. Do not select the best run or tune on the held-out set. A busy server with no completions must report zero goodput and undefined successful-task cost rather than omit the run.

Before a load test, inspect the **pinned** engine benchmark entrypoint with `vllm bench serve --help`; record the exact supported flags. If that revision lacks this CLI, use its checked-in serving benchmark entrypoint and record its SHA. Do not copy speculative `kv_aware` flags into a runnable command: at Router base `91b65ee4`, that policy is not wired. A trace runner, task judge and fault injector are still required beyond the current conformance helpers.

The paired vLLM revision above was inspected at `vllm/entrypoints/cli/benchmark/serve.py`, `vllm/benchmarks/serve.py` and `vllm/benchmarks/datasets/datasets.py`; the following flag names exist there. This is a **planned synthetic cold-load example**, not a measured workload, warmed-prefix replay or Agent evaluation. Run only after G0/G1 pass and resources are assigned. The rate/length/concurrency values are illustrative and must be replaced by the preregistered workload; `127.0.0.1:8080` must be the actual reachable upstream ingress. Use the installed tokenizer matching production weights, not the model alias as a tokenizer path.

~~~bash
vllm bench serve --backend openai-chat \
  --base-url http://127.0.0.1:8080 --endpoint /v1/chat/completions \
  --model local-model --tokenizer Qwen/Qwen3-0.6B \
  --dataset-name random --random-input-len 4096 --random-output-len 128 \
  --num-prompts 1000 --request-rate 4 --max-concurrency 32 --seed 1 \
  --percentile-metrics ttft,e2el --metric-percentiles 50,95,99 \
  --save-result --result-dir benchmark-results
~~~

Repeat across the fixed policy/cache conditions with separate result directories, alternating run order and identical generated inputs/arrival seeds. `e2el` is end-to-end request latency, not full Agent-task time. Random prompts do not establish prefix-reuse accuracy; warm/churn/PD and model-selection quality require the dedicated traces and oracles in the table. This CLI does not measure classifier/Agent loop costs unless those are actually on the tested path.

Full RFC implementation and shared validation design:
https://github.com/Liuchenbing-2026/Model_test/blob/router/agent-routing-evaluation/项目交付/任务列表/任务13%20llm-d%20Router调研与可复现指南/详细任务/路由方案/README.md#validation-plan-20261011

### Native verification entrypoints

Run from repository root; documentation check is separate from real ExtProc E2E:

~~~bash
make check CHANGED_FILES="tools/dev/examples/ascend-tokenizer-cache/README.md"
make verify PROFILE=complexity-remote-backend
~~~

`complexity-remote-backend` is a maintained profile at the fixed base; prepare its declared environment and replace endpoints through supported configuration. This plan does not mark that profile passed. Validate the profile's model/runtime prerequisites before using it for Ascend; classifier or model download failures must be recorded separately from backend routing errors.
