# OQTOPUS Agent Skills 設計メモ

対象: [oqtopus-client](https://github.com/oqtopus-team/oqtopus-client) を AI エージェントから使えるようにするための Skill / MCP のパッケージング方針。

---

## 1. 全体方針: Skill 優先、MCP は後から薄く

**OpenAPI から 1:1 で MCP サーバーを自動生成する方向は採らない。** 理由:

1. **ジョブ投入が単一エンドポイントではない** — `POST /jobs`(register) → presigned URL への S3 アップロード → `POST /jobs/{id}/submit` の3段プロトコル。生ツールで LLM に踏ませるのは事故のもと。`services/client.py:496` の `run_job` 系がすでに吸収している。
2. **結果ペイロードが大きい** — counts 辞書や sselog アーカイブをそのままツール戻り値にするとコンテキストが破綻する。境界での要約・切り詰めが必須で、自動生成では実現できない。
3. **待ちがある** — `wait_for_job` はポーリングループ。300秒ブロックする MCP ツールは行儀が悪い。
4. **危険なエンドポイントが混ざる** — `DELETE /users/me`、`DELETE /api-token` をエージェントに握らせる理由がない。
5. **量子ワークフローはコード形** — パラメータ付き回路、VQE の最適化ループ、numpy での後処理はツール呼び出しの列で表現できない（`examples/userprogram_vqe.py`）。

oqtopus-client はすでに良い抽象になっている（`OqtopusJobSpec.sampling(...)` + `client.run_sampling(req, timeout=300)` の2行）。この上に MCP を被せても情報が減るだけ。

### 層構造

| 層 | 内容 | 状態 |
|---|---|---|
| 層1 | **Skill** — エージェントに `uv run python` で oqtopus-client を書かせる | 最優先。これから作る |
| 層2 | **薄い MCP** — 5〜7ツール。読み取り系 + 非ブロッキング submit | 層1が回ってから判断 |
| 層3 | oqtopus-cloud コントリビュータ向け Skill | 別リポ・別スコープ（後述） |

層2を作る場合のツール案（書き込み系はここまで。token/user 管理は入れない）:

- `list_devices` / `get_device` — 読み取り専用・キャッシュ可。calibration_data は要約して返す
- `submit_job` — **非ブロッキング**で job_id だけ返す
- `get_job_status` — 安い。ポーリングはエージェント側のターンに任せる
- `get_job_result` — **切り詰め必須**。counts は上位N件＋総 shots、full dump はファイル書き出し
- `list_jobs` — セッションをまたいだジョブ追跡

実装はスクラッチではなく oqtopus-client を叩く薄いラッパにする（3段投入の重複実装を避ける）。

---

## 2. 認証: 完全に oqtopus-client に任せる。自前管理しない

`src/oqtopus_client/services/config.py` が全部持っている。

- `OqtopusConfig.from_file(section, path=...)` — INI形式。パス省略時は `$XDG_CONFIG_HOME/oqtopus/config.ini`、無ければ `~/.config/oqtopus/config.ini`（config.py:78-86）
- `OqtopusConfig.from_env()` — `OQTOPUS_BASE_URL` / `OQTOPUS_API_TOKEN` / `OQTOPUS_PROXY`
- SSE コンテナ内では `OQTOPUS_ENV=sse_container` で config 不要（config.py:71-75）。**userprogram.py はローカルと同じコードのままでよい**

Skill / MCP に「認証」の機能は要らない。書くべきなのは**規約**:

1. **`config.ini` を絶対に読ませない。** api_token が平文で入っている。エージェントには section 名だけ扱わせ、読むのはライブラリに任せる
2. **`print(config)` を禁止。** `OqtopusConfig` は `@dataclass(frozen=True)` で `api_token` がフィールドなので、デフォルトの `__repr__` にトークンがそのまま出る。デバッグで設定オブジェクトを print するのは AI がやりがち。トレースバックに載ると出力ログにも残る
3. **`create_api_token` / `delete_api_token` / `delete_current_user` はエージェントに触らせない**

> MCP 経路の数少ない明確な利点: サーバー起動時に `from_file()` しておけば、**トークンが MCP プロトコル境界を一度も越えない**。セキュリティを重く見るなら MCP 寄りの判断材料になる。

---

## 3. 回路生成: Qiskit を採用。ローカル transpile はしない

### Qiskit vs OpenQASM3 直書き → **Qiskit**

- 学習データ量が圧倒的。`circuit.h(0)` / `circuit.cx(0,1)` はまず間違えない。QASM3 直書きは 2系の記憶と混ざる（`creg`/`qreg`、`include "qelib1.inc"` など）
- **エラーが即座に Python 例外で返る**。QASM3 のミスはサーバー側で弾かれるためキュー待ちを経てから失敗が判明する = 自己修正の効率が桁違い
- プログラム的構成ができる（n量子ビット GHZ、QFT、VQE ansatz を手で展開しなくてよい）
- 人間の可読性も高い

quri-parts も `convert_to_qasm_str` で同じことができる（`examples/run_sampling_quri_parts.py`）が、AI の生成精度という一点で Qiskit に分がある。

### ローカル transpile は**入れない**

OQTOPUS 側が OpenQASM のゲートを解釈してサーバーでトランスパイルする。`spec/openapi.yaml` に `jobs.S3TranspileResult` があるのとも整合。

- サーバー側トランスパイラは calibration_data / `calibrated_at` を持つ（`services/device.py:96,120`）。ローカルで `basis_gates` に決め打ちすると、キャリブレーション情報なしの劣った割り当てを固定してしまう
- 依存が減り、qiskit のバージョン差の影響も減る

→ 手順は `qasm3.dumps(circuit)` 直行。投入前チェックに残るのは `dev.n_qubits`（device.py:70）と `dev.status`（device.py:43）程度。トランスパイル結果はジョブ結果から取得してデバッグに使う。

### ⚠ 未検証（SKILL.md の核心を左右する）

非 transpile の回路に `qasm3.dumps()` をかけると、標準外のゲートに対し `gate ... { ... }` の定義ブロックを出力することがある。**これが OQTOPUS 側のパーサで通るかは仕様書から読めない。** qulacs に1回投げれば確定する。通るなら SKILL.md は「Qiskit で組んで dumps して投げる」だけの非常に薄いものになる。

---

## 4. パッケージング: `vercel-labs/skills` + Codex / Claude Code plugin の併存

ユーザーは oqtopus-client を clone しない。**別リポジトリ**にして CLI 一発で入る形にする。

### なぜ [vercel-labs/skills](https://github.com/vercel-labs/skills) か

OQTOPUS は OSS の量子クラウドで、ユーザーが特定のエージェントを使っているとは限らない。skills CLI は Claude Code / Cursor / Cline / OpenCode など **75以上のエージェント**に同じ `SKILL.md` を配れる。このプロジェクトの性質では配布範囲の広さが効く。

さらに、同じ `skills/` を Codex の `.codex-plugin/plugin.json` と Claude Code の `.claude-plugin/plugin.json` の両方から参照できる。**3経路を1リポジトリで同時に満たせる**ので、排他ではない。

### リポジトリ構成

```
oqtopus-team/oqtopus-agents-plugin/
├── README.md                    # 3経路のインストール手順
├── skills/
│   └── oqtopus/
│       ├── SKILL.md
│       └── references/
│           ├── qasm.md          # QASM3 / Qiskit の書き方
│           └── job-types.md     # sampling / estimation / multi_manual / sse の選び分け
├── .codex-plugin/
│   └── plugin.json              # Codex plugin の必須 manifest
├── .agents/
│   └── plugins/
│       └── marketplace.json     # Codex の Git/repo marketplace
├── .claude-plugin/
│   ├── plugin.json              # Claude Code plugin manifest
│   └── marketplace.json         # "source": "./"
└── (将来) .mcp.json             # 層2の MCP をルートに置く
```

### Codex plugin として加える方法

Codex 側は公式の [Package your plugin](https://developers.openai.com/plugins/build/plugins) に従う。最小 manifest は `.codex-plugin/plugin.json`。パスは plugin ルート相対で `./` から始める。

```json
{
  "name": "oqtopus",
  "version": "0.1.0",
  "description": "Build and run quantum workloads on OQTOPUS.",
  "skills": "./skills/"
}
```

層2の MCP を追加するときだけ、同じ manifest に `"mcpServers": "./.mcp.json"` を加える。Codex は、ルートに `.mcp.json` があるだけでは plugin の MCP として読み込まない。

`.agents/plugins/marketplace.json` は、リポジトリのルート自体を plugin として公開する。

```json
{
  "name": "oqtopus",
  "interface": {
    "displayName": "OQTOPUS"
  },
  "plugins": [
    {
      "name": "oqtopus",
      "source": {
        "source": "local",
        "path": "./"
      },
      "policy": {
        "installation": "AVAILABLE",
        "authentication": "ON_INSTALL"
      },
      "category": "Developer Tools"
    }
  ]
}
```

`source.path` は `.agents/plugins/` からではなく marketplace ルート（この場合はリポジトリルート）からの相対パス。`./` を指定すると同じリポジトリの `.codex-plugin/plugin.json` が解決される。

### インストール

```bash
# エージェント横断
npx skills add oqtopus-team/oqtopus-agents-plugin

# Codex plugin
codex plugin marketplace add oqtopus-team/oqtopus-agents-plugin
codex plugin add oqtopus --marketplace oqtopus
# インストール後は新しい Codex セッションを開始する

# Claude Code plugin
/plugin marketplace add oqtopus-team/oqtopus-agents-plugin
/plugin install oqtopus@oqtopus
```

Codex では `codex` を起動して `/plugins` を開き、`oqtopus` marketplace からインストールする方法でもよい。更新時は `codex plugin marketplace upgrade oqtopus` を実行し、plugin を更新してから新しいセッションで確認する。

### 経路ごとの差

| | skills CLI | Codex plugin | Claude Code plugin |
|---|---|---|---|
| Skill | ○ | ○ | ○ |
| MCP（層2） | ✗ 運ばない | ○ `mcpServers` → `.mcp.json` | ○ `.mcp.json` |
| バージョン | ✗ 常に最新 | ○ `.codex-plugin/plugin.json` | ○ `.claude-plugin/plugin.json` |
| インストール UI | 各エージェント依存 | Codex CLI の `/plugins` | Claude Code の `/plugin` |

`.mcp.json` をルートに置き、Codex manifest の `mcpServers` と Claude Code 側の設定から参照すれば、**Codex / Claude Code ユーザーだけ MCP も受け取り、skills CLI ユーザーは Skill だけ受け取る**という自然な段階差になる。

Codex と ChatGPT の公開 plugin directory への掲載は、GitHub marketplace 配布とは別工程。初期は上記の Git-backed marketplace で配布・検証し、利用実績と層2 MCP が固まった段階で OpenAI の [plugin submission](https://developers.openai.com/plugins/deploy/submission) から審査・公開する。

### 注意点

- Codex / Claude Code の両 `plugin.json` で `name` / `version` / `description` を一致させる。リリース時に片方だけ上げない
- Codex の `.codex-plugin/` 内には `plugin.json` だけを置く。`skills/`、`.mcp.json`、assets は plugin ルートに置く
- Codex marketplace の各 plugin entry には `policy.installation` / `policy.authentication` / `category` を明示する
- frontmatter は `name` + `description` が全経路共通の必須項目。WIP を隠すなら `metadata.internal: true`
- `npx` = Node 必須が Python ユーザーには地味な摩擦。README に3経路 + 手動コピーのフォールバックを併記する
- CI で JSON 構文、両 manifest の name/version 一致、`skills` と `mcpServers` の参照先、Codex marketplace の `source.path` を検証する

---

## 5. 版ズレ対策

別リポにする唯一の代償。Skill が oqtopus-client の API を説明する以上、離すと嘘をつき始める。効く順に:

1. **SKILL.md を薄く保つ** — 固定するのは `__init__.py` の `__all__` に出ている安定面（`OqtopusClient` / `OqtopusConfig` / `OqtopusJobSpec` / `run_sampling` 等）だけ。引数の細部は「`python -c 'help(...)'` で確認しろ」「readthedocs を見ろ」に委ねる。**Skill が薄いほど壊れにくい**
2. **CI スモークテスト** — SKILL.md に載せたコード片を PyPI 最新の oqtopus-client に対して実際に走らせる。qulacs シミュレータ相手なら実機キューを消費しないので、デモ環境のトークンを secrets に置ければ版ズレは実質潰せる
3. **バージョン番号を合わせる**（plugin 経路のみ有効） — plugin の `version` を oqtopus-client のマイナーに追随させる（client 1.1.x → plugin 1.1.x）

### 設計上の好都合

**「版ズレに強い書き方」と「エージェント横断で移植性が高い書き方」が同じ方向を向く。** どちらも「薄く、環境依存を持たず、詳細は実行時に確認させる」に収束する。具体的には:

- `${CLAUDE_PLUGIN_ROOT}` を使わない
- `allowed-tools` など Claude Code 固有の frontmatter に依存しない（書いてもよいが、無視されても成立する内容にする）
- バンドルスクリプトを前提にしない。`references/*.md` への相対参照だけにする

---

## 6. スコープ外: oqtopus-cloud

[oqtopus-cloud](https://github.com/oqtopus-team/oqtopus-cloud) は Python backend + Terraform/AWS + 運用 runbook。ユーザー向けではなく**コントリビュータ向け**で、彼らは確実に clone する。

→ **plugin にしない。`.claude/skills/` を oqtopus-cloud リポジトリに直接コミットする。** ユーザー向け Skill と混ぜる理由がない。

内容としては: OAS 変更 → クライアント再生成のフロー（上流 `backend/oas/user/openapi.yaml` を oqtopus-client の `spec/Makefile` が `download-oas` で引いてくる関係）、デプロイ手順など。

---

## 7. SKILL.md に書くべき内容（草案）

READMEに書いてあることではなく、**LLM が実際に間違えるところ**を書く。

- 認証規約（§2 の3項目）
- Qiskit で回路を組んで `qasm3.dumps()` する型。Bell state をカノニカル例に（`examples/run_sampling_qiskit.py`）
- 投入前チェック: `list_devices` で `n_qubits` / `status` を確認してから投げる
- ジョブタイプの選び分け: sampling / estimation / multi_manual / sse
- **SSE (Server-Side Execution)** — エージェントと相性が非常に良い。「Python ファイルを書いて投げると QPU 近傍でサーバー実行される」= エージェントの自然な作業形態そのもの。`run_sse_file` + `userprogram.py` の書式（何が import できるか、結果の返し方）を明文化する
- qiskit は oqtopus-client の依存に入っていない（examples でのみ使用）ので `uv pip install qiskit` の一文が要る。嫌な環境向けに QASM3 直書きパスも残す二段構え
- `spec/openapi.yaml` は 47KB ≒ 15k tokens。**SKILL.md 本文に貼らない。** 参照だけ置く（progressive disclosure）

---

## 8. 未決事項 / 次のアクション

- [ ] **`qasm3.dumps()` の `gate` 定義ブロックが OQTOPUS で通るか**を実測（Bell状態 + 非基本ゲート回路の2本を qulacs に投げて比較）
- [ ] 進め方: **A** リポ雛形を先に作り QASM 周りは実測後に確定 / **B** 検証スクリプトを先に書く（手戻りがないがデモ環境トークンが必要）
- [x] リポジトリ名の確定 → `oqtopus-agents-plugin`（2026-08-11 決定）。marketplace 名（仮: `oqtopus`）は未定
- [ ] `.codex-plugin/plugin.json` と `.agents/plugins/marketplace.json` を雛形に含め、ローカル marketplace から `codex plugin add` できることを CI またはリリース前チェックで確認
- [ ] 層2 MCP を作るかの判断は層1が回り始めてから
