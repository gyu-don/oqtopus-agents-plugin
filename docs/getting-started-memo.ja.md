# oqtopus-client 使い始めメモ

対象: https://oqtopus-client.readthedocs.io/en/latest/ を見ながら実際に `uv venv` + `uv pip install oqtopus-client qiskit` で手を動かして分かったこと。
実行環境: Python 3.12 / `oqtopus-client==1.1.7` / `qiskit==2.5.1`。接続先は誰でも使える demo 環境（`base_url = https://demo-api.oqtopus.io`）。**実クラウドへのジョブ投入まで確認済み。**

関連: [oqtopus-skills-design.md](../oqtopus-skills-design.md), [decision.ja.md](decision.ja.md)

---

## 1. インストール

```bash
pip install oqtopus-client   # or: uv pip install oqtopus-client
```

コアは量子SDK非依存で、`qiskit` を入れなくても `oqtopus_client` 単体で動く（26パッケージ中 qiskit 関連は明示的に足したときだけ増える）。依存は `aiohttp` / `pydantic` / `pyyaml` など軽量なもののみ。

## 2. 設定方法は3通り（優先順位は「設定ファイル」推奨）

```python
# 1. 設定ファイル（推奨）: $XDG_CONFIG_HOME/oqtopus/config.ini か ~/.config/oqtopus/config.ini
from oqtopus_client import OqtopusClient
client = OqtopusClient()  # 引数なし = [default] セクションを読む

# 2. 環境変数
# export OQTOPUS_BASE_URL=... / export OQTOPUS_API_TOKEN=...
from oqtopus_client import OqtopusClient, OqtopusConfig
client = OqtopusClient(OqtopusConfig.from_env())

# 3. コード直書き
client = OqtopusClient(OqtopusConfig(base_url="...", api_token="...", retry_max_attempts=3, retry_backoff_seconds=0.2))
```

`config.ini` の中身:
```ini
[default]
base_url = <url>
api_token = <token>
```

設定なしで `OqtopusClient()` を呼ぶと、ファイルが無くても素通りせず即座に例外になる:
```
ValueError: Section 'default' not found in config file: /home/penguin/.config/oqtopus/config.ini
```
→ 「設定ファイルが無ければ黙って環境変数にフォールバック」のような曖昧な挙動ではなく、明示的にエラーになる。Skill 側でエラーメッセージをそのままユーザーに見せれば十分に自己解決可能。

## 3. `OqtopusConfig` の `repr`/`str` に `api_token` は出ない（現行版）

以前の `oqtopus-client` では `@dataclass(frozen=True)` のデフォルト `__repr__` によって `api_token` が平文で表示される問題があった。しかし、[oqtopus-client の現行実装](https://github.com/oqtopus-team/oqtopus-client)（GitHub の `develop` および v1.0.1 以降）では `api_token` が `repr` の対象から除外されており、`repr(config)`/`str(config)` からトークンが漏れないよう修正されている。

```python
>>> config = OqtopusConfig(base_url="https://example.com", api_token="SECRET_TOKEN_ABC")
>>> "SECRET_TOKEN_ABC" in repr(config)
False
>>> "SECRET_TOKEN_ABC" in str(config)
False
```

したがって、この問題を理由に現行版の `OqtopusConfig` を print/log することを一律禁止する必要はない。ただし、古いバージョンを使っている環境では平文表示が起こりうるため、`oqtopus-client` を最新版へ更新すること。トークンを含む設定オブジェクトをログへ出さない運用も、追加の防御策としては引き続き有効。

## 4. QASM3出力の「未検証」だった点も確認できた

oqtopus-skills-design.md 78-79行目で「Qiskit の `qasm3.dumps()` が非標準ゲートに対して `gate ... { ... }` の定義ブロックを出すことがあるが、OQTOPUS側パーサが通すかは仕様書から読めない」とあった件。

ローカルで実験した結果:

- **標準ゲート（h, cx, measure）のみ** → `include "stdgates.inc"` の宣言だけで完結し、追加の `gate` 定義ブロックは出ない。
- **`rzz` を使うと `gate rzz(p0) _gate_q_0, _gate_q_1 { cx ...; rz ...; cx ...; }` という定義ブロックが自動生成される**（`ccx`, `swap`, パラメータ付き `rx`/`ry`/`u` は `stdgates.inc` の範囲内でそのまま展開され、定義ブロックは不要だった）。

```qasm3
OPENQASM 3.0;
include "stdgates.inc";
gate rzz(p0) _gate_q_0, _gate_q_1 {
  cx _gate_q_0, _gate_q_1;
  rz(p0) _gate_q_1;
  cx _gate_q_0, _gate_q_1;
}
...
```

→ **【解決】実クラウド（`qulacs`, demo環境）に実際に投げて確認した。`gate` 定義ブロック付きQASM3は問題なく `SUCCEEDED` した。**

```qasm3
OPENQASM 3.0;
include "stdgates.inc";
gate rzz(p0) _gate_q_0, _gate_q_1 {
  cx _gate_q_0, _gate_q_1;
  rz(p0) _gate_q_1;
  cx _gate_q_0, _gate_q_1;
}
bit[2] c;
qubit[2] q;
h q[0];
rzz(0.7) q[0], q[1];
c[0] = measure q[0];
c[1] = measure q[1];
```
→ `status: SUCCEEDED`, `counts: {'00': 535, '01': 465}`。

design memo の想定通り、**Qiskit で好きなゲートを使って組み、`qasm3.dumps()` してそのまま投げるだけでよい**ことが確定した。ゲートセットを `stdgates.inc` 収録分に絞る必要はない。サーバー側が `basis_gates`（後述、`qulacs` は `['sx','x','rz','cx']`）へのトランスパイルを担っているため、クライアント側でのトランスパイルは design memo の方針通り不要。

## 5. `OqtopusClient` の公開メソッド一覧（全35個）

ジョブ系:
`submit_job` / `run_job` / `run_sampling` / `run_estimation` / `run_multi_manual` / `run_sse` / `run_sse_file` / `wait` / `status` / `is_finished` / `result` / `get_job` / `get_job_status` / `get_job_result` / `get_sselog` / `cancel_job` / `delete_job` / `list_jobs`

バッチ/並列系:
`submit_jobs` / `submit_jobs_async` / `wait_for_job` / `wait_for_jobs` / `wait_for_jobs_async` / `run_jobs_batch`

デバイス系:
`list_devices` / `get_device`

アカウント/トークン系（design memoで「エージェントに触らせない」と明言されている）:
`get_api_token` / `get_api_token_status` / `create_api_token` / `delete_api_token`

その他: `get_announcement` / `get_announcements_list` / `refresh`

`run_job` 系はいずれも `interval` / `interval_backoff` / `max_interval` / `timeout` / `on_status`（ポーリング中のコールバック）を持つ。デフォルト `timeout=300.0`。**300秒ブロッキングしうる**という design memo の指摘（層2のMCPを非ブロッキングにすべき理由）はシグネチャからも裏付けられた。

## 6. `OqtopusJobSpec` のファクトリ

`sampling` / `estimation` / `multi_manual` / `sse` の4つの classmethod。どれも共通で `device_id`（必須, キーワード専用）, `program`（`str` 1本 or 複数プログラムの `Sequence[str]`）, `shots=1000`, `name`, `description`, `transpiler_info`, `simulator_info`, `mitigation_info` を取り、`sampling`/`estimation`/`multi_manual` はさらに `operator`（推定演算子）を取れる。`program` が単一 `str` だけでなく複数プログラムの `Sequence[str]` を受け付ける点は見落としやすい（マルチプログラムジョブの入口）。

## 7. 認証情報の取得・安全な設定方法

demo環境（`https://demo.oqtopus.io`）はアカウント登録さえすれば誰でも使える。API tokenの発行場所は **`https://demo.oqtopus.io/settings/security`**（ログイン後のアカウント設定内）。Browser Use系のツールがある環境ならブラウザ操作で自動取得してもよいが、**デフォルトはユーザー本人がブラウザで取得する**運用でよい（トークン発行はアカウントに紐づく操作なので、エージェントに自動でやらせる必然性が薄い）。

`config.ini` へ書き込む際、以下の点を守るとトークンが平文で残る経路を塞げる:

- **トークン文字列をAIとの会話に一切貼らせない。** チャット越しに渡すとログ・トランスクリプトに残る。
- `export OQTOPUS_API_TOKEN=xxxx` のような**コマンドライン引数形式は避ける**（シェル履歴・`ps`に残る）。
- 対話的な非エコー入力（`read -s`）でその場で `config.ini` に書き込ませるのが安全:

```bash
mkdir -p -m 700 ~/.config/oqtopus
: > ~/.config/oqtopus/config.ini
chmod 600 ~/.config/oqtopus/config.ini
read -r -s -p "OQTOPUS API token (非表示入力): " OQTOPUS_TOKEN; echo
printf '[default]\nbase_url = %s\napi_token = %s\n' "https://demo-api.oqtopus.io" "$OQTOPUS_TOKEN" > ~/.config/oqtopus/config.ini
unset OQTOPUS_TOKEN
```
  - `heredoc`（`<<EOF`）ではなく `printf` を使うのは、トークンに `$` や `` ` `` が含まれていた場合の意図しない展開・実行を避けるため。
  - AIエージェント（Bashツール経由の非対話実行）はこのステップを**代行できない**（`read`が入力を受け取れずハングする）。ユーザー自身の端末、または対話接続されたシェル（Claude Codeの `!` プレフィックス等）で実行する必要がある。
- **エージェント側は `config.ini` の中身を絶対に `cat`/`Read` しない。** 動作確認は SDK 越し（`OqtopusClient()` を呼んで `list_devices()` 等の非秘匿な結果だけを見る）で十分。パーミッションが `600` になっているかだけ `ls -l` で確認すればよい。

これで decision.ja.md / oqtopus-skills-design.md にある「`config.ini` を絶対に読ませない」「`print(config)` を禁止」の運用を、認証情報の**入力段階**から一貫させられる。

## 8. 実際にジョブを投げて分かったこと

### デバイス情報（`qulacs`、demo環境）

```python
client.get_device('qulacs')
# device_type: 'simulator'
# n_qubits: 16, status: 'available'
# basis_gates: ['sx', 'x', 'rz', 'cx']
# supported_instructions: ['measure', ' barrier']
# calibration_data: None   # シミュレータのため。実機デバイスでは値が入る可能性が高く未確認
```
`calibration_data` はこのシミュレータでは `None` だった。design memoが懸念していた「calibration_dataが巨大でSkillでの要約が必須」という点は、**実機デバイスでないと検証できない**（demo環境に実機があるかは未確認）。

### Bell状態を実際に投げた結果

```python
qc = QuantumCircuit(2, 2); qc.h(0); qc.cx(0, 1); qc.measure([0,1],[0,1])
req = OqtopusJobSpec.sampling(device_id='qulacs', shots=1000, program=qasm3.dumps(qc))
result = client.run_job(req, timeout=120.0)
# result.status: SUCCEEDED
# result.get_counts(): {'00': 501, '11': 499}   ← Bell状態として妥当
# result.execution_time: 0.021 (秒)
```

`result.transpile_result.transpiled_program` に、サーバー側で `basis_gates` へ変換された後のQASM3（`rz`/`sx`/`cx`のみ）と `stats`（transpile前後のqubit数・ゲート数・深さ）、`virtual_physical_mapping`（論理→物理qubit/bitの対応）が入っている。デバッグ用に十分な情報量。

### エラー挙動（2種類ある点に注意）

1. **`submit_job` 時点で即座に例外**: 存在しない `device_id` を指定すると `submit_job()` 呼び出し自体が `UserApiError: HTTP 400: Bad Request` を送出する（同期的・fail fast）。
2. **ジョブ自体は受理されるが非同期に失敗**: QASM3として不正な文字列（`"this is not qasm at all"`）を投げると、`submit_job`/`run_job` は例外を送出せず、ポーリング後に `result.status == FAILED` として返る。`result.message` はサーバー内部のパースエラーがそのまま出る（例: `"Expecting value: line 1 column 1 (char 0)"` — QASM3構文エラーというより内部のJSONパースエラーに見える文言で、ユーザー向けとしては分かりにくい）。

→ **Skill設計への示唆**: `run_job`/`wait` が例外を投げなかった場合でも、必ず `result.status` を見て `FAILED` を弾く必要がある（「例外が出なければ成功」という前提は誤り）。`message` はサーバー側の生文言なので、Skill側でも「QASM3構文を確認してください」等の補足を添えたほうが親切。

### `list_jobs()` で履歴が取れる

```python
client.list_jobs()  # 直近のジョブ一覧。job_id / status / name などが取れる
```
セッションをまたいだジョブ追跡（design memoの層2 MCP案 `list_jobs` の裏付け）に使える。

## 9. まだ確認できていないこと

- 実機デバイス（simulatorでなく実量子コンピュータ）での `calibration_data` の実際のスキーマ・サイズ感（Skillでの要約設計に影響）— demo環境に実機があるかどうかも含めて未確認
- 多qubit・多shotジョブでの `get_job_result()` の counts 辞書の実際のサイズ（design memoの「境界での切り詰め必須」判断の裏付け。今回は2qubitなので最大4パターンしかなく検証にならない）
- `submit_jobs` / `wait_for_jobs` 等の並列バッチ系ヘルパーの実地動作
- `run_sse` / `run_sse_file`（SSEコンテナ経由のジョブ）の実地動作
