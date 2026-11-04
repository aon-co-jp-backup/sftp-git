# sftp-git

📖 English: [README-English.md](README-English.md)

WEBサイト開発者向けに、**SFTPアップロードとGit管理**を統合し、
**バージョンレスAPI**と**バージョン管理**のハイブリッド運用で、
テストサーバーと本番環境の差分チェック・デプロイ作業の手間とミスを
減らすことを目指すツール。5つの中核機能(ドリフト検出・
VersionlessAPIハイブリッド・不要ファイルAI削除判定・DUAL DATABASE・
AI差分解析)のロジック実装+実サーバーでのE2E検証まで完了(2026-08-15)。
LSPサーバー化・VS Code拡張(Node製薄層)も実装・実機起動確認済み。

参考にした既存サービス: https://lp.smartrelease.cloud/

## 配布形態(ユーザー指定、2026-08-15)

- **VS Code拡張機能**として開発する。
- 加えて、**Windows/Mac/Linux**用のネイティブアプリと、
  **Android(スマホ・タブレット)/iPhone/iPad**用のモバイルアプリも
  開発する。
- 現時点では方針のみで、VS Code拡張機能とネイティブ/モバイルアプリの
  コードをどこまで共有するか(ロジックを共通ライブラリ化しUIだけ
  各プラットフォームで作る、等)は未設計。詳細は[DESIGN.md](DESIGN.md)
  「7. マルチプラットフォーム配布」節参照。
- **開発ツール連携(ユーザー指示、2026-08-15)**: LSPサーバーは標準の
  Language Server Protocolに準拠しているため、VS Code以外の任意の
  LSPクライアントからも同じRust製バイナリで連携可能(Claude Code等)。
- **複数AIプロバイダ対応(ユーザー指示、2026-08-15)**: AI差分解析の
  バックエンドとして、aruaru-llmに加えClaude(Anthropic)・
  ChatGPT(OpenAI)・Gemini(Google)・DeepSeek・Grok(xAI)を選択可能に
  した(`src/ai_providers.rs`)。各社APIキーは環境変数で設定
  (未設定時はそのプロバイダを使わず正直にエラーを返す)。

## 目指す機能(構想)

- **Git管理とSFTP UPLOADのハイブリッド**: Gitで変更履歴を追いつつ、
  従来のSFTP運用に慣れた開発者にも扱えるアップロード導線を両立する。
- **バージョンレス×バージョン管理のハイブリッド**: VersionlessAPIの
  身軽さと、厳密なバージョン管理の安全性の良いとこ取りを狙う。
- **テスト/本番差分のAIチェック**: 差分検知・レビューに
  [aruaru-llm](https://github.com/aon-co-jp/aruaru-llm)を利用する構想。
  差分解析そのものの推論を高速化するため、
  [open-cuda](https://github.com/aon-co-jp/open-cuda)(NVIDIA GPU)と
  DirectX互換の[open-directx](https://github.com/aon-co-jp/open-directx)
  (CUDA非対応環境向け)をaruaru-llmのHWアクセラレーションバックエンド
  として使い分ける。加えて、検出した差分について**日本語・英語
  両方**でAIによる改善アドバイスを生成する機能を持たせる。
- **不要ファイルのAI削除判定支援**: Git+VersionlessAPIで管理する
  古いソース・使われなくなったファイルについて、削除してよいか
  [aruaru-llm](https://github.com/aon-co-jp/aruaru-llm)が判断材料
  (参照されているか、最終更新からの経過、コミット履歴上の扱われ方等)
  を示して作業者の判断を支援する。**最終的な削除実行は作業者の承認を
  必須とし、AIが自動で削除する設計にはしない**(誤削除リスクのため)。
- **フルバックアップ**: 基本は[aruaru-db](https://github.com/aon-co-jp/aruaru-db)。
  希望に応じてPostgreSQLも追加し、**DUAL DATABASE構成**(片側に
  不具合が発生してももう片側がフル補完・サポートする冗長構成)を
  選択可能にする。Rust+[RPoem](https://github.com/aon-co-jp/RPoem)
  で機能追加。
- **ハードウェアアクセラレーション(用途別に使い分け)**:
  - 暗号化・圧縮・展開の高速化には
    [open-raid-z](https://github.com/aon-co-jp/open-raid-z)の
    `open_raid_z_core`(GPU/D3D12対応、
    [RS-SmartTCP](https://github.com/aon-co-jp/RS-SmartTCP)が実際に
    利用中)を使う。調査の結果、この用途には
    [open-directx](https://github.com/aon-co-jp/open-directx)・
    [open-cuda](https://github.com/aon-co-jp/open-cuda)は合わないと
    判明したため対象外。
  - AI差分解析・アドバイス生成(aruaru-llmの推論)の高速化には
    [open-cuda](https://github.com/aon-co-jp/open-cuda)(NVIDIA GPU)と
    [open-directx](https://github.com/aon-co-jp/open-directx)(DirectX
    互換、CUDA非対応環境向け)を使い分ける。こちらは元々LLM推論向けの
    実装であり、用途に合致する。

## 市場調査(2026-08-15)

日英で既存事例を調査した結果、以下が分かった。

- **SFTP×Git連携ツール**: DeployHQ・GitFTP-Deploy・Git FTP Sync
  (GitHub Actions)・AutoDeployToFTP・FTPbucket等、
  「git push→SFTP自動反映」という**片方向**の自動化ツールは複数存在する。
  しかし**SFTP側を手動編集した際にGitリポジトリとの乖離(ドリフト)を
  検出する逆方向の照合機能**を持つ専用ツールは見当たらなかった。
  → sftp-gitの差別化ポイントになり得る。
- **VersionlessAPI**: Stripeが2024-09以降採用している方式(アカウント
  単位の日付ベースAPIバージョン+リクエスト単位のヘッダー上書き、
  内部は現行実装+過去バージョンへの変換トランスフォーマー)が代表例。
  GitHub API(URI版+メディアタイプ併用)のような「破壊的変更のみ
  明示バージョン、それ以外は無破壊進化」というハイブリッド運用の
  先行事例はあるが、「versionless×versioned」を明示的に名付けた
  汎用設計パターンの文献は見当たらなかった。
- **ステージング/本番差分検出**: インフラのドリフト検出ツール
  `driftctl`(Snyk買収)は2023年にEOL済みで事実上放棄状態。
  AI活用のステージング/本番差分専用レビューツールは今回の調査では
  具体名を特定できなかった。

実現方式レベルの検討は[DESIGN.md](DESIGN.md)にまとめている。

## 現在の状態(2026-08-15時点)

**5機能のロジック実装+実サーバーでのE2E検証が完了**:

| 機能 | 実装 | 実サーバーE2E |
|---|---|---|
| SFTPドリフト検出 | `src/drift.rs`+`src/sftp_client.rs` | ✅ ローカルOpenSSH Serverで成功 |
| VersionlessAPIハイブリッド | `src/versionless_api.rs` | (ロジックのみ、RPoem実依存は未着手) |
| 不要ファイルAI削除判定支援 | `src/cleanup_advisor.rs` | (ロジックのみ、実削除はしない設計) |
| DUAL DATABASE | `src/dual_database.rs`+`src/dual_database_client.rs` | ✅ ローカルaruaru-serverで書き込み成功 |
| AI差分解析(日英) | `src/ai_diff_advisor.rs`+`src/aruaru_llm_client.rs` | ✅ 接続成功だがモデル品質は実用に耐えず(下記参照) |

**LSPサーバー化+VS Code拡張**: `src/bin/sftp_git_lsp.rs`(5機能全てを
カスタムリクエストとして公開)+`vscode-extension/`(Node製の薄い
接続層のみ、rust-analyzer方式)。**5機能全てにコマンドパレット経由の
UI導線を実装済み**(2026-08-15、`sftpGit.cleanupAdvice`/
`sftpGit.detectDrift`/`sftpGit.versionlessResolve`/`sftpGit.analyzeDiff`/
`sftpGit.dualDatabaseState`系3コマンド)。VS Code拡張ホストの実起動→LSP
サーバーの子プロセスspawn→正常終了まで確認済み。

**重要な発見(正直な開示)**: 小型GPT-2系モデル(`distilgpt2`/
`gpt2-medium`)は指示追従非対応のため、フォーマット指定プロンプトに
従えない。`ai_diff_advisor`は日英別プロンプトで2回呼び出す方式へ
設計変更済み。

**複数AIプロバイダ対応**: aruaru-llmに加え、Claude/ChatGPT/Gemini/
DeepSeek/Grokを選択可能にするクライアント(`src/ai_providers.rs`)を
実装。**実APIキーでの疎通確認は未実施**(このセッションにはいずれの
有効なキーも無かったため)。

詳細な実装ログ・実測結果はCLAUDE.mdのHANDOFF節を参照。

**RPoem実依存**: `src/versionless_api.rs`にRPoem
(`open-runo-versionless-api`、path依存)の`CompatibilityRule`/
`apply_compatibility`を使った`json_registry::JsonVersionRegistry`を
実装済み。

## 未着手・次の課題

- 複数AIプロバイダの実HTTP疎通確認(有効なAPIキーが必要)
- Windows/Mac/Linuxデスクトップアプリ・Android/iOSモバイルアプリ
  (着手順序: VS Code拡張→デスクトップ→モバイル、DESIGN.md参照)
- VS Code拡張のUI(現状は`showInputBox`によるJSON貼り付け形式、
  もう少しリッチなUI〈Webview等〉への発展は今後の検討課題)

## 関連リポジトリ

- [aruaru-llm](https://github.com/aon-co-jp/aruaru-llm) — AI差分チェック連携先候補
- [aruaru-db](https://github.com/aon-co-jp/aruaru-db) — フルバックアップ連携先候補
- [RPoem](https://github.com/aon-co-jp/RPoem) — Rust機能追加の連携先候補
- [open-raid-z](https://github.com/aon-co-jp/open-raid-z) — HWアクセラレーション暗号化・圧縮の本命候補(`open_raid_z_core`)
- 全体索引: [RUNO](https://github.com/aon-co-jp/RUNO)

## API 方針: REST 全廃・gRPC / Connect へ (2026-11-04)

### 日本語
- **REST は全廃する。新しい API は REST で作らない。** ブラウザや `curl` から呼ぶ API は **Connect の単純呼び出し**(`POST /<パッケージ>.<Service>/<Method>`、`content-type: application/json`、本文は JSON)、サービス間は **gRPC / gRPC-web**(RPoem の管理API `open_runo.admin.v1.Admin` と同じ流儀)。ストリーミングは、必要になってから。
- **既存の REST は、触るときに順に移す。** 移したら旧 REST の経路は消す。移行が終わるまで残るが、**新規には追加しない**。
- **JSON は RS-JSON に統一**する(読み込みは `from_str_secure` / `parse_secure` でサイズ・深さ・重複キーの上限つき、書き出しは `rust_json::fast`)。ブラウザの WASM に `prost` は入れない(サイズが増えるため)。
- **例外(自分たちの API ではなく、相手の規格が決めているもの)**: ACME、OAuth/OIDC のコールバック、第三者(GitHub など)からの Webhook、ヘルスチェック、静的ファイル配信、OTLP など。これらは規格どおりの HTTP を受ける。
- 参考実装: `RPoem`(`open_runo.admin.v1.Admin`)、`open-easy-web`(`server/src/connect.rs`、`proto/open_easy_web/admin/v1/admin.proto`)。
- ※ このリポジトリのコードが既に方針どおりかどうかは、各リポジトリの現状による。この節は方針の記録であり、コードの変更を意味しない。

### English
- **REST is being abolished. New APIs are not built as REST.** APIs called from browsers or `curl` use **Connect unary calls** (`POST /<package>.<Service>/<Method>`, `content-type: application/json`, JSON body); service-to-service calls use **gRPC / gRPC-web** (the same style as RPoem's admin API `open_runo.admin.v1.Admin`). Streaming only when it is actually needed.
- **Existing REST endpoints are migrated one by one, when they are touched.** The old REST route is removed once migrated. Until migration is finished they remain, but **no new ones are added**.
- **JSON is unified on RS-JSON**: reading with `from_str_secure` / `parse_secure` (size, depth and duplicate-key limits), writing with `rust_json::fast`. `prost` is not put into browser WASM (it would grow the binary).
- **Exceptions (shapes dictated by someone else's standard, not our own API)**: ACME, OAuth/OIDC callbacks, webhooks from third parties (e.g. GitHub), health checks, static file serving, OTLP, and so on. These accept plain HTTP as the standard requires.
- Reference implementations: `RPoem` (`open_runo.admin.v1.Admin`) and `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Note: whether the code in a given repository already follows this policy depends on that repository. This section records the policy; it does not mean the code has been changed.

### Deutsch (KI-Übersetzung, noch nicht von Muttersprachlern geprüft)
- **REST wird abgeschafft. Neue APIs werden nicht als REST gebaut.** APIs, die aus dem Browser oder mit `curl` aufgerufen werden, nutzen **Connect-Unary-Aufrufe** (`POST /<Paket>.<Service>/<Methode>`, `content-type: application/json`, JSON-Body); zwischen Diensten gilt **gRPC / gRPC-web** (wie bei der Admin-API von RPoem, `open_runo.admin.v1.Admin`). Streaming erst, wenn es wirklich nötig ist.
- **Bestehende REST-Endpunkte werden einzeln migriert, sobald man sie anfasst.** Nach der Migration wird die alte REST-Route entfernt. Bis dahin bleiben sie bestehen, es werden aber **keine neuen hinzugefügt**.
- **JSON wird auf RS-JSON vereinheitlicht**: Lesen mit `from_str_secure` / `parse_secure` (Grenzen für Größe, Tiefe und doppelte Schlüssel), Schreiben mit `rust_json::fast`. `prost` kommt nicht in Browser-WASM (die Datei würde größer).
- **Ausnahmen (Formate, die ein fremder Standard vorgibt, nicht unsere eigene API)**: ACME, OAuth/OIDC-Callbacks, Webhooks von Dritten (z. B. GitHub), Health-Checks, statische Dateien, OTLP usw. Diese nehmen schlichtes HTTP entgegen, wie es der Standard verlangt.
- Referenzimplementierungen: `RPoem` (`open_runo.admin.v1.Admin`) und `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Hinweis: Ob der Code eines Repositorys diese Richtlinie schon befolgt, hängt vom jeweiligen Repository ab. Dieser Abschnitt hält die Richtlinie fest; er bedeutet nicht, dass der Code bereits geändert wurde.

### Русский (машинный перевод, носителями языка ещё не проверен)
- **REST полностью отменяется. Новые API не создаются как REST.** API, вызываемые из браузера или через `curl`, используют **унарные вызовы Connect** (`POST /<пакет>.<Service>/<Method>`, `content-type: application/json`, тело — JSON); между сервисами — **gRPC / gRPC-web** (так же, как административный API RPoem, `open_runo.admin.v1.Admin`). Потоковая передача — только когда она действительно нужна.
- **Существующие REST-эндпоинты переносятся по одному, когда до них доходят руки.** После переноса старый REST-маршрут удаляется. До завершения переноса они остаются, но **новые не добавляются**.
- **JSON унифицируется на RS-JSON**: чтение через `from_str_secure` / `parse_secure` (ограничения на размер, глубину и повторяющиеся ключи), запись через `rust_json::fast`. `prost` не добавляется в браузерный WASM (размер вырос бы).
- **Исключения (форматы, заданные чужим стандартом, а не наш собственный API)**: ACME, обратные вызовы OAuth/OIDC, вебхуки сторонних сервисов (например, GitHub), проверки работоспособности, раздача статических файлов, OTLP и т. п. Они принимают обычный HTTP, как требует стандарт.
- Эталонные реализации: `RPoem` (`open_runo.admin.v1.Admin`) и `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Примечание: соответствует ли код конкретного репозитория этой политике, зависит от самого репозитория. Этот раздел фиксирует политику и не означает, что код уже изменён.
