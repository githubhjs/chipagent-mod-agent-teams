# chipagent-mod-agent-teams：ChipAgents 的 agent team

[English](README.md) | 繁體中文

這是 [wshobson/agents](https://github.com/wshobson/agents)（★41k）裡的 **agent-teams** plugin，移植到 ChipAgents，並針對 RTL 和驗證工作重寫。流程是：主 agent 帶領一組專職 subagent，各自獨立工作並回報，最後由主 agent 合併結果。

| 指令 | 團隊組成 | 產出 |
|---|---|---|
| `/team-review` | 每個 review 面向一個 `team-reviewer`（預設：functional、CDC、lint） | 去重並依嚴重度排序的問題清單 |
| `/team-debug` | 每個互相競爭的假設一個 `team-debugger` | 經過仲裁、附證據的根因（Analysis of Competing Hypotheses，競爭假設分析） |
| `/team-feature` | 每個模組或工作流一個 `team-implementer`，分 wave 進行 | 整合後通過 lint、elaboration 和模擬的程式碼 |
| `/team-status` | — | 最新或指定 run 的任務、發現和未解問題 |
| `/team-shutdown` | — | 封存 run，並清除它的 chip-swarm worktree |

六個 skill 承載方法論：
- 多 reviewer 模式
- 平行除錯
- 平行功能開發
- 任務協調
- 檔案式團隊協定
- 團隊組成範本

## 在 ChipAgents 裡團隊怎麼運作

原版依賴 Claude Code 實驗性的 Agent Teams 功能（`TeamCreate`、`SendMessage`、共用 task list），ChipAgents 沒有這些。這個移植版改用一個**run 目錄**，所有 agent 都讀寫這裡：

```
.chipagents/team/<run-id>/
├── brief.md          目標、範圍、背景、假設、變更紀錄
├── tasks.md          每個任務一行：id、負責人、負責的檔案、wave、狀態
├── contracts/        凍結的介面：SV package、interface、port list
├── findings/<agent>.md   每個 agent 的報告（取代傳訊息給 lead）
├── questions/<agent>.md  需要 lead 處理的阻礙
└── report.md         lead 合併後的結果
```

- **主 agent 就是 lead**：寫 brief、用完整自足的 prompt 派出 subagent，最後合併 `findings/`。
- 任務之間的相依改成 **wave**：前一個 wave 完成後，下一個 wave 才開始。
- 實作開始前，介面先**凍結在 `contracts/`**，因為團隊成員在執行中無法互相協調。
- 需要**真正的行程層級平行**時，每個指令都說明如何改用 [chip-swarm](https://github.com/githubhjs/chipagent-mod-swarm) 執行同樣的團隊：
  - review 和 debug 用 `chip-swarm investigate`
  - 功能工作流用 `chip-swarm fanout`

## 硬體化調整

**Review 面向**：
- 功能／規格符合度（FSM、reset 值、register map）
- CDC/RDC 和 reset
- lint 與可合成性
- 協定符合度（AXI/APB/valid-ready、backpressure）
- 時序／PPA
- 驗證品質（SVA、coverage、checker）
- 硬體安全（debug lock、secure register、lifecycle）
- DFT
- 另有選用的 scripts/tools 面向

**除錯假設分類**：
- RTL 邏輯
- testbench／stimulus
- reset／初始化／X
- CDC／順序
- handshake／協定
- register 設定
- 工具與環境（filelist、define、模擬器版本、seed）
- gate-level/SDF/時序

每項證據都必須附上 `file:line`、`signal@time` 和 seed。

**功能開發**：
- 負責的檔案是模組、package 和 testbench 元件。
- contract 是 SV package、interface 和 port list。
- 範例改用 DMA 和 AXI4-Lite 區塊，取代原本的 TypeScript/React。

## 安裝

```bash
chipagents plugin marketplace add githubhjs/chipagent-mod-agent-teams --type claude
chipagents plugin install agent-teams@chipagent-mod-agent-teams
```

如果是在 session 裡安裝，裝完執行 `/reload`。這個 repo 在 Claude Code 也能安裝（`claude plugin marketplace add githubhjs/chipagent-mod-agent-teams`），同樣的檔案協定可以搭配 Claude 的 subagent 使用。

## 使用方式

```
/team-review rtl/dma/ --reviewers functional,cdc,protocol
/team-debug "dma_tb: scoreboard mismatch at 18.4us" --hypotheses 3 --seed 1234
/team-feature "Add a second DMA channel" --plan-first
/team-status
/team-shutdown
```

`/team-feature --plan-first` 會在完成拆解和 contract 後停下來，等你核准（plan mode 或明確確認）之後，才開始派出 implementer。

## 注意事項

- **還沒在真正的 ChipAgents 裡跑過。** 已通過 `claude plugin validate --strict`。ChipAgents 怎麼派出 subagent、能不能同時跑多個，取決於 ChipAgents 本身。指令的寫法是：支援的話就同一個 wave 一起派，否則依序派出。
- **reviewer 和 debugger 是唯讀的**，沒有 Edit/Create 權限。它們用 shell heredoc 寫出自己的 findings 檔；如果你的指令允許清單擋掉這種寫法，就由 lead 把它們回傳的報告存檔。
- 原版的 `team-spawn` 和 `team-delegate` 已移除，它們的團隊範本併入 team-composition-patterns skill。

每個檔案的修改都列在 [PORTING.md](PORTING.md)（原版 commit `0874a27`）。

## 相關 repo

- [chipagent-mod-swarm](https://github.com/githubhjs/chipagent-mod-swarm)：把 ChipAgents 團隊當成平行行程執行（race、investigate、fanout）
- [chipagent-mod-feature-dev](https://github.com/githubhjs/chipagent-mod-feature-dev)：Anthropic 的 feature-dev 流程的 ChipAgents 版
- [chipagents-loop-kit](https://github.com/githubhjs/chipagents-loop-kit)：讓 agent 在目標完成前不停

## 授權

與原版相同，採用 MIT（[LICENSE](LICENSE)）。原作者為 Seth Hobson（[wshobson/agents](https://github.com/wshobson/agents)）。

## 作者

Chih-Hsueh "Josh" HUANG（ChipAgents 移植）。
