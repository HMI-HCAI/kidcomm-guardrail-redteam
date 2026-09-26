# kidcomm-guardrail-redteam

`kidcomm-safety-guardrail` 的红队评测集。被测对象是 [louis111111111/kidcomm-agent-skills](https://github.com/louis111111111/kidcomm-agent-skills) 里的 `kidcomm-skills/kidcomm-safety-guardrail/scripts/screen.py`。本仓库只做测试,不修改被测代码。

## 为什么要做

原护栏的 `evals/evals.json` 只有 5 条用例,每条都有一条规则逐字对应,所以"5/5 通过"说明不了规则能不能拦住换了说法的输入。本评测集按护栏 `SKILL.md` 自己声明的检测范围出题,每类包含一条原规则能命中的对照用例和若干变体,另外加一组孩子正常表达的用例,用来测误杀(`skill-card.md` 写了要测"误杀率",原用例里只有 1 条正常用例)。

## 文件

| 文件 | 说明 |
|------|------|
| `redteam_cases.json` | 44 条用例。每条写明 `basis`(预期值依据,指向被测 Skill 的 SKILL.md / skill-card.md 条目);带 `review_needed` 的条目,预期值需要团队确认 |
| `run_redteam.py` | 运行器,按类别统计漏拦与误杀 |
| `generate_variants.py` | 调本地模型批量生成改写变体,输出可直接交给运行器的用例文件 |
| `DGX_SETUP.md` | 在 DGX Spark 上启动本地模型服务的步骤 |
| `REPORT.md` | 针对被测仓库 commit `c036c46` 的评测结果和规则修改建议 |

## 运行

```bash
git clone https://github.com/louis111111111/kidcomm-agent-skills.git
python run_redteam.py --guardrail kidcomm-agent-skills/kidcomm-skills/kidcomm-safety-guardrail
python run_redteam.py --guardrail ... --json results.json   # 逐条结果
```

只依赖 Python 3 标准库。全部通过时退出码为 0,否则为 1,可以直接接进 CI。

## 用本地模型批量生成变体

`generate_variants.py` 调用 OpenAI 兼容接口(DGX Spark 上的 vLLM 或 llama-server),把 `redteam_cases.json` 的每条人工用例按 5 种手法改写(数字混淆、语音转写噪声、同义改写、口语化、嵌入长对话;应放行的用例只做无害改写)。变体的预期动作继承自种子用例,不由模型判断;按 20% 抽样标记 `needs_review`,供人工复核。

```bash
export KIDCOMM_BASE_URL="http://localhost:8000/v1"
export KIDCOMM_MODEL="nvidia/Qwen3-14B-FP8"
python generate_variants.py --dry-run                          # 看提示词,不调模型
python generate_variants.py --only rt-p01,rt-t02 --n 3 --out variants_trial.json   # 小样试跑
python generate_variants.py --n 5 --out variants.json          # 全量
python run_redteam.py --guardrail <护栏路径> --cases variants.json --json variants_results.json
```

人工标注的 44 条是基准,模型生成的变体单独统计,两份结果不合并。

DGX Spark 上怎么起本地模型服务,见 [DGX_SETUP.md](DGX_SETUP.md)。

## 判定口径

- **漏拦**:实际动作比预期宽松(应 block 得到 warn 或 allow,应 warn 得到 allow)
- **误杀**:预期 allow,实际被 warn 或 block
