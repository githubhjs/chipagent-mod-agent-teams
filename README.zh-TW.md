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

## 該用哪一個

先選對模式，這一步選錯最浪費時間。

| 你要的 | 用哪個 | 怎麼做到 |
|---|---|---|
| 多種策略競賽，勝者出線 | `chip-swarm race` | 獨立行程，腳本裁決 |
| 多個假設獨立查證再合流 | `chip-swarm investigate` | 唯讀調查者＋合流 |
| 多個獨立任務並行 | `chip-swarm fanout` | 各自 worktree |
| 團隊在同一 session 內 | [agent-teams](https://github.com/githubhjs/chipagent-mod-agent-teams) | 不開新行程，用 subagent |
| 探索／設計／審查一個功能 | [feature-dev](https://github.com/githubhjs/chipagent-mod-feature-dev) | 平行探索再設計 |

關鍵差別：[chip-swarm](https://github.com/githubhjs/chipagent-mod-swarm) 開的是**真的獨立行程**，每隊有自己的 context，結果由腳本判定；agent-teams 和 feature-dev 在當前 session 內跑，共用同一份 context。

chip-swarm 是 CLI，不是 plugin：它沒有 marketplace.json 也沒有 plugin.json，把 `bin/chip-swarm` 放進 `PATH` 就好。agent-teams 和 feature-dev 是 Claude 格式的 plugin，用 `--type claude` 安裝。

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

### 確認 ChipAgents 真的載入了

已在 ChipAgents 0.42.3 上確認安裝成功（Josh_HUANG 實測：`plugin list` 有列出這個 plugin，安裝目錄裡也有它的 agents、commands 和 skills）。但 ChipAgents 會不會**把 Claude 格式的 agents 和 commands 載入 session**，還沒確認。可以用以下兩個方法快速檢查，記得使用和 session 相同的 `chipagents`：

```bash
# 1. plugin 的指令找得到嗎？（batch mode 可以用 --command 執行自訂指令）
chipagents batch --command "team-status" --output /tmp/agent-teams-cmd.txt; echo "rc=$?"

# 2. agent 看得到這些 profile 和指令嗎？
chipagents batch --prompt "List the slash commands and subagent profiles available to you. Names only." \
  --output /tmp/agent-teams-caps.txt
grep -E 'team-review|team-debug|team-feature|team-reviewer|team-debugger|team-implementer' /tmp/agent-teams-caps.txt
```

如果找不到指令，或清單裡沒有這些名稱，代表檔案有裝上但沒有被載入，請附上輸出回報。

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

- **已在 ChipAgents 0.42.3 安裝，但還沒實際跑過**（見下方實跑紀錄）。 已通過 `claude plugin validate --strict`。ChipAgents 怎麼派出 subagent、能不能同時跑多個，取決於 ChipAgents 本身。指令的寫法是：支援的話就同一個 wave 一起派，否則依序派出。
- **reviewer 和 debugger 是唯讀的**，沒有 Edit/Create 權限。它們用 shell heredoc 寫出自己的 findings 檔；如果你的指令允許清單擋掉這種寫法，就由 lead 把它們回傳的報告存檔。
- 原版的 `team-spawn` 和 `team-delegate` 已移除，它們的團隊範本併入 team-composition-patterns skill。

每個檔案的修改都列在 [PORTING.md](PORTING.md)（原版 commit `0874a27`）。

## 實跑紀錄：ChipAgents 0.42.3（2026-10-20）

由 Josh_HUANG 在 CentOS 8 機器上安裝並檢查：

| plugin | agents | commands | skills |
|---|---|---|---|
| agent-teams | 4 | 5 | 6 |
| feature-dev | 3 | 1 | 0 |

- 兩個都是 Claude 格式 marketplace，用 `--type claude` 加入後安裝，`plugin list` 都列得出來。
- **兩個都完全沒有用 hook**：hooks.toml、hooks.json、agent 型 hook、JSON block、post_turn 全部是 0。這點很重要：[hook 機制筆記](https://github.com/githubhjs/chipagents-loop-kit/blob/main/docs/chipagents-hook-mechanics.md)裡每一種安靜失效模式都和 hook 有關，這兩個只用 agents 與 commands，全部繞開了。
- Claude 專屬的 `TeamCreate`／`SendMessage` 搜尋命中 5 筆，逐筆看過後確認**全部在文件裡**，是在說明「原版用這些、改用共享目錄取代」，實作中 0 筆。移植是乾淨的。

沒有驗證的事：

- **agents/commands 是否真的被載入 session。** 兩個 plugin 都裝好了、`plugin list` 也列得出來、檔案確實在安裝目錄，但 `installed.json` 只記錄 enabled/format/path，沒有 agents/commands/skills。檔案存在不等於會被載入；請用[確認 ChipAgents 真的載入了](#確認-chipagents-真的載入了)裡的檢查。
- 指令的**實際執行**。只驗證到安裝與結構。
- ChipAgents 是否同時跑多個 subagent。subagent 的回合會觸發 `post_turn` hook，但不會出現在 run log，只能靠副作用觀察；並行度本身沒測。需要確定的並行，請用 [chip-swarm](https://github.com/githubhjs/chipagent-mod-swarm)，它的每隊是獨立行程（README 有實測耗時）。

## 相關 repo

- [chipagent-mod-swarm](https://github.com/githubhjs/chipagent-mod-swarm)：把 ChipAgents 團隊當成平行行程執行（race、investigate、fanout）
- [chipagent-mod-feature-dev](https://github.com/githubhjs/chipagent-mod-feature-dev)：Anthropic 的 feature-dev 流程的 ChipAgents 版
- [chipagents-loop-kit](https://github.com/githubhjs/chipagents-loop-kit)：讓 agent 在目標完成前不停

## 授權

與原版相同，採用 MIT（[LICENSE](LICENSE)）。原作者為 Seth Hobson（[wshobson/agents](https://github.com/wshobson/agents)）。

## 作者

Chih-Hsueh "Josh" HUANG（ChipAgents 移植）。
