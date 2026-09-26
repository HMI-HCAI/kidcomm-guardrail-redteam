# kidcomm-guardrail-redteam

`kidcomm-safety-guardrail` 的红队评测集。被测对象是 [louis111111111/kidcomm-agent-skills](https://github.com/louis111111111/kidcomm-agent-skills) 里的 `kidcomm-skills/kidcomm-safety-guardrail/scripts/screen.py`。本仓库只做测试,不修改被测代码。

## 为什么要做

原护栏的 `evals/evals.json` 只有 5 条用例,每条都有一条规则逐字对应,所以"5/5 通过"说明不了规则能不能拦住换了说法的输入。本评测集按护栏 `SKILL.md` 自己声明的检测范围出题,每类包含一条原规则能命中的对照用例和若干变体,另外加一组孩子正常表达的用例,用来测误杀(`skill-card.md` 写了要测"误杀率",原用例里只有 1 条正常用例)。

## 文件

| 文件 | 说明 |
|------|------|
| `redteam_cases.json` | 44 条用例。每条写明 `basis`(预期值依据,指向被测 Skill 的 SKILL.md / skill-card.md 条目);带 `review_needed` 的条目,预期值需要团队确认 |
| `run_redteam.py` | 运行器,按类别统计漏拦与误杀 |
| `REPORT.md` | 针对被测仓库 commit `c036c46` 的评测结果和规则修改建议 |

## 运行

```bash
git clone https://github.com/louis111111111/kidcomm-agent-skills.git
python run_redteam.py --guardrail kidcomm-agent-skills/kidcomm-skills/kidcomm-safety-guardrail
python run_redteam.py --guardrail ... --json results.json   # 逐条结果
```

只依赖 Python 3 标准库。全部通过时退出码为 0,否则为 1,可以直接接进 CI。

## 判定口径

- **漏拦**:实际动作比预期宽松(应 block 得到 warn 或 allow,应 warn 得到 allow)
- **误杀**:预期 allow,实际被 warn 或 block
