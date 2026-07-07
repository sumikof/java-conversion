# Java バージョンアップ支援ツール 開発方針書 v2

v1 からの改訂版。v1 に対する設計レビュー(2026-07-04〜07 の対話)での全決定を反映。主要な変更点は末尾の「付録: 決定記録」を参照。

## 1. 目的

Java 7 で稼働する既存システムのソースコードを、生成 AI(LLM)を活用して Java 21(LTS)まで段階的にバージョンアップする**汎用ツール**を開発する。公開されている各バージョンの互換性情報(リリースノート、JEP、移行ガイド)を根拠として、必要最小限の修正を適用する。

## 2. スコープと前提

- **汎用ツール**である。ただしビルドシステムは **Gradle のみ**対応する
- マルチモジュール(マルチプロジェクト)構成は「**全サブプロジェクト一斉**」で扱う(サブプロジェクトごとの段階進行はしない)。初期実装のマイルストーンはシングルモジュールから刻む
- 対象プロジェクトの **Gradle 定義(build.gradle / settings.gradle)はツールが編集しない**。ビルド条件の差し替えはすべて init script オーバーレイで行う(8 章)
- 移行完了時点でリポジトリ内で変わっているのは **.java ファイルのみ**。ビルド定義の恒久変更は「ビルド変更レポート」として人間に引き渡す
- 最初のパイロット対象は、Gradle 9.2 + ECJ(互換レベル 7)でビルドされている実システム。ただしこの構成はツールの前提にしない

### 意図的スコープ外

| 項目 | 扱い |
|------|------|
| Maven / その他ビルドツール | 非対応 |
| モック自動生成による境界コードのテスト | 将来拡張。初期は保証対象外として可視化のみ |
| ライブラリの別製品への置き換え | エスカレーションして人間対応 |
| build.gradle の恒久的な書き換え | ビルド変更レポートで人間が実施 |
| ファイルキューの並列処理 | 直列で正しく動くまで見送り(コンパイルは全体単位なので利得は LLM 呼び出しのみ) |
| Web UI | CLI + Markdown + PR で開始。SQLite に全データがあるため後付け可能 |

## 3. 基本原則

| # | 原則 | 内容 |
|---|------|------|
| 1 | 振る舞い保証優先 | アップグレード開始前に Java 7 上で Characterization Test(Golden Master 方式)を作成・凍結し、全移行完了まで**同一のテストバイトコード**が通り続けることを合格基準とする。保証範囲は「カバーされた実行パス」であり、保証範囲率を第一級の報告値とする(15 章) |
| 2 | リファクタリング禁止 | 変更は「新バージョンでコンパイル・動作するために必要な修正」のみ。構文モダン化は行わない。機械的 AST 判定で強制する(7.3 節)。例外: 人間による手動修正は validate_diff の対象外(9.4 節) |
| 3 | 最小差分 | LLM の出力は SEARCH/REPLACE 形式で受け取り、適用後に AST 差分検証。無関係な差分は reject して再生成 |
| 4 | 根拠ベース | 修正は必ずナレッジベースの該当エントリを引用して行う。根拠のない修正は人間エスカレーション対象 |
| 5 | 再開可能 | 状態は SQLite 業務テーブル+Git(ブランチ・コミット・タグ)を正とし、任意時点で中断・再開できる。LangGraph チェックポイントは内側修正ループの一時状態のみに使う |
| 6 | 人間承認 | 各バージョンゲートで人間が全パッチを一括レビュー・承認してから次へ進む。「どのようなエラーが出てどう対処したか」を症状→根拠→対処→検証結果のセットで確認できることを最優先する |
| 7 | 凍結テストの唯一の例外 | 凍結テストの変更は原則禁止。唯一の例外は、人間が「非本質挙動の固定だった」と承認した場合の**再ベースライン**(6.5 節)。自動再ベースラインは禁止 |

## 4. バージョンアップ経路とゲートモデル

LTS のみを経由する: `Java 7 → 8 → 11 → 17 → 21`

変化を 1 つずつ入れるため、各ステップをサブゲートに分解する。失敗原因の切り分け(根拠ベース修正の前提)がゲート分解の目的である。

```
ゲート 0(最初に 1 回だけ: コンパイラ切替の隔離)
    javac(JDK 8)で -source 7 -target 7 コンパイル
    → 生成物を JDK 7 JVM で実行し凍結テスト全パス
    → ここで出るエラーは「コンパイラ差(ECJ vs javac)」。KB の gate=compiler_diff

各 LTS ステップ N→M は 2 サブゲート:
    ゲート A(ランタイムのみ上げる)
        レベル N でコンパイル済みの生成物を JDK M の JVM で実行し凍結テスト全パス
        → 失敗はランタイム挙動差・削除 API のリンクエラー。KB の gate=runtime
    ゲート B(言語レベルを上げる)
        --release M でコンパイル → JDK M で凍結テスト全パス
        → 失敗はコンパイル時非互換。KB の gate=compile
```

### 各ステップの重点領域(v1 から変更なし)

| ステップ | 主な対応事項 |
|---------|-------------|
| 7 → 8 | デフォルトメソッド衝突、バイトコード検証の厳格化、内部 API 変更 |
| 8 → 11 | **最大の難所**。Java EE モジュール削除(JAXB / JAX-WS / javax.annotation / CORBA)、JPMS による内部 API 制限、ツール削除 |
| 11 → 17 | 内部 API の強カプセル化、Nashorn 削除、SecurityManager 非推奨、RMI Activation 削除 |
| 17 → 21 | スレッド関連の挙動変更、非推奨 API 削除 |

### バージョン進行の管理単位

バージョンステップは**プロジェクト単位**の状態である(ファイル単位のバージョン到達管理は廃止)。「1 ファイルずつ」は修正の粒度(LLM コンテキストと diff 検証の単位)としてのみ維持する。

```
1. 現在ゲートの条件でプロジェクト全体をコンパイル(または実行)
2. エラーが出たファイルの一覧を抽出 → 作業キュー
3. キューのファイルを 1 つずつ修正(内側修正ループ、7.2 節)
4. キューが空になったら全体コンパイル → 全凍結テスト実行
5. 全件成功でゲート完了 → ステップ末尾なら人間のゲート承認(9.1 節)→ 次ステップへ
```

サブプロジェクト間で修正が連鎖するケース(core の API 置換が web を壊す)は、このループが自然に吸収する(次回の全体コンパイルで新たなエラーファイルがキューに載る)。

## 5. アーキテクチャ概要

```
┌─ Docker イメージ(linux/amd64 固定)──────────────────────────┐
│  Python ツール本体(CLI)                                      │
│    外側オーケストレーション: 素の Python + SQLite(正)         │
│    内側修正ループ: LangGraph + SqliteSaver                     │
│  JDK 7 / 8 / 11 / 17 / 21(ベンダー・パッチレベル固定)        │
│  JVM ヘルパー群:                                               │
│    - JUnitCore 直接 fork 実行系(テスト実行・Golden Master)   │
│    - JavaParser AST 差分サービス(validate_diff)              │
│    - SecurityManager サンドボックス(記録時の境界遮断)        │
│  JUnit 4.13.2 jar / ECJ(ゲート 0 以前のベースライン用)       │
└───────────────────────────────────────────────────────────────┘
      │                          │
   対象リポジトリ             LLM API(外向き許可はこれのみが理想)
   (Git: upgrade/ ブランチ)
```

- **JDK 調達は Docker 一択**。JDK 7 の入手性(Azul Zulu 等)と、Golden Master が JVM ビルドまで含めた挙動を記録することから、全員・全実行で JDK を完全一致させる。JDK 7 に arm64 ビルドは存在しないため `linux/amd64` 固定(Apple Silicon ではエミュレーション実行)
- runs テーブルにイメージタグを記録し、実行環境の指紋として監査を閉じる
- コンテナのファイルシステム隔離+SecurityManager の JVM 内遮断で防壁を二重化する

### テスト実行系

テスト実行は全ステップで Gradle を経由せず、ツールが `JAVA_HOME=<対象JDK>` の `java` プロセスを直接 fork して `org.junit.runner.JUnitCore` を実行する。理由:

- Gradle 9 は JDK 7 のテストワーカーを fork できない(toolchain 下限 8)
- 記録時と再生時で実行経路を同一にしないと、ハーネス差が挙動差として混入し原因切り分けが不能になる
- クラスパスはベースライン時の init script ダンプ(8 章)から組み立てる

### 制御の責務分割

- **外側(工程管理)= 素の Python**: ステップ×ゲート進行・作業キュー・ゲート承認待ちは決定的な進行であり、状態は SQLite 業務テーブル+Git に全部入っている。再開は「業務テーブル+Git から現在位置を再導出」で行う。承認待ちは `awaiting_approval` ステータスで表現(interrupt 不要)
- **内側(1 ファイルの修正)= LangGraph + SqliteSaver**: LLM 呼び出し・条件分岐・リトライ・人間入力待ち(interrupt)が絡む本来のエージェントループ。thread_id = 「ファイル×ステップ×ゲート」のキュー項目単位
- 正が 2 つになることを避ける: LangGraph チェックポイントは内側ループの進行中状態のみを持ち、完了した事実はすべて業務テーブルへ書く

## 6. フェーズ 1: Characterization Test 生成(Java 7 上)

### 6.1 Golden Master 方式

期待値の出所は**実行結果**であり、LLM の推測ではない。

1. **LLM の役割は入力生成のみ**: 対象メソッドの呼び出しコード(引数のバリエーション・境界値・例外を誘発しそうな値)を生成する
2. ハーネスがそれをベースライン JVM(= 本番稼働 JVM のメジャーバージョン。設定項目)上で実行し、戻り値・例外・必要なら引数オブジェクトの事後状態を観測して記録する
3. 記録値から assert を機械的に生成してテストコードを確定する

テストは **JUnit 4(4.13.2)** で作成する(Java 7〜21 を同一テストで通すため)。

### 6.2 呼び出しグラフと構築順序

静的解析でプロジェクトの呼び出しグラフを構築し、**末端ノード(プロジェクト内の他メソッドに依存しないメソッド)から**テストを構築する。上位メソッドのテストは検証済みの下位実装をそのまま使う。

### 6.3 境界コードの扱い: パス単位判定

外部リソース(DB・ネットワーク・ファイル I/O・サーブレット API 等)に触れるコードは実行して記録できない。**メソッド単位の二値ラベルではなく、入力(テストケース)単位で判定**する:

1. 静的解析ラベルは優先順位付けにのみ使う: 推移的に境界 API(JDBC / `java.net` / `java.io` 書き込み系 / JNDI / servlet API 等のデナイリスト)へ一切到達しないメソッド = 全パス安全
2. Golden Master の記録実行は **SecurityManager サンドボックス内**で行い、境界操作を横取りして例外化する。境界に触れずに完走したケースだけを記録・テスト化し、触れたケースは破棄(そのパスは保証対象外)
3. 結果として JaCoCo の行・分岐カバレッジがそのまま「保証範囲の可視化」になる

サンドボックスは安全装置としても必須である(static イニシャライザ等が本番 DB 接続・ファイル削除・メール送信を実行しうる)。モック自動生成は行わない(モック応答値は LLM の推測であり、質問 3 で排除した推測が裏口から戻るため)。

### 6.4 非本質挙動の除外(層 1: 予防)

Java バージョン間で変わることが公式に許容されている挙動を固定しないよう、記録・assert 生成に規則を組み込む:

- `HashSet`/`HashMap` 由来のコレクションは順序不問比較(ソートして比較)。`List` は順序込み
- 例外は**型のみ** assert し、メッセージ文字列は記録しない
- 日付・ロケール依存文字列はそのまま assert せず、構造化値(エポックミリ秒等)に分解して記録
- 記録・再生の両方で JVM 環境を固定: `-Duser.timezone=UTC -Duser.language=en -Dfile.encoding=UTF-8` 等を fork 時に常時付与

典型例: HashMap イテレーション順序(JDK 8 ツリー化)、CLDR デフォルト化(JDK 9)による日付フォーマット変化、Helpful NPE のメッセージ変化。

### 6.5 再ベースライン(層 2: 例外規定)

それでも非本質挙動の固定が漏れてテストが落ちた場合の唯一の出口:

- **人間が「非本質挙動の固定だった」と承認した場合のみ**、該当 assert を新バージョンで再記録できる
- DB に `rebaseline` として記録し、KB に「この挙動はバージョン間で非保証」エントリを追加(次プロジェクトでは層 1 で予防される)
- ツール・LLM による自動再ベースラインは**禁止**(原則 1 の抜け道になるため)

### 6.6 凍結の実体: バイトコード

凍結テストは「ソース+コンパイル済み jar(ハッシュ付き)」の組とする。ゲート 0 で javac(`-source 7 -target 7`)により一度だけコンパイルし、以後 Java 21 まで**同一バイトコード**(バージョン 51.0、全後継 JVM で動く)を全 JVM で実行する。各ステップでの再コンパイルはコンパイラ差をテスト側の挙動差として混入させるため行わない。

テスト資産は対象ブランチの専用ディレクトリ(`characterization-tests/`)に隔離し、凍結後のコミットで touch されたら機械チェックで検出する。

## 7. フェーズ 2: アップグレードループ

### 7.1 外側ループ(素の Python)

4 章のゲートモデルに従って進行。ゲート内の作業キュー処理・全体検証・ゲート承認待ちを制御する。

### 7.2 内側修正ループ(LangGraph、1 ファイル単位)

```
[retrieve_knowledge] → [generate_patch] → [validate_diff]
      → NG: reject 理由を文脈に加えて generate_patch へ(リトライ上限 3)
      → OK: [apply_patch(コミット)] → 外側へ返し再コンパイルで検証
リトライ上限/トークン予算超過/KB 該当なし → [escalate](interrupt で人間待ち)
```

- State はファイル修正ループの状態のみ(from/to_version・gate はキュー項目から与えられる定数)
- リトライ時は「前回の失敗パッチ+reject/失敗理由」を文脈に注入し、同じ失敗の繰り返しを防ぐ

### 7.3 validate_diff(機械判定主・LLM 従)

1. **AST 差分抽出は JavaParser の JVM ヘルパー**で行う(javalang はメンテ停止・Java 8 構文までのため不採用)。「変更前後のソース → 構造化された変更リスト(JSON)」を返す
2. **機械判定(主)**: KB エントリの `allowed_changes`(構造化された許可変更リスト。例: `replace_import: javax.xml.bind.* → jakarta.xml.bind.*`)に AST 差分の全項目が該当するかをコードで照合。禁止パターン(識別子リネーム・メソッド本体の構文書換・コメント/フォーマットのみの差分・knowledge_id を提示できない変更)は即 reject
3. **LLM 判定(従)**: 許可リスト外だが禁止でもないグレーゾーンのみ LLM に回す。LLM が「許容」でも自動適用せず、**灰色フラグ付きでゲートレビューに載せる**。疑わしきは通さない

### 7.4 パッチ形式

LLM 出力は **SEARCH/REPLACE ブロック形式**(unified diff の直接出力は行番号ずれで壊れやすく、全文再出力は無関係差分が混入しやすい)。ツールが適用 → AST 差分検証 → patches テーブルには適用後に生成した正規の unified diff を保存する。

## 8. ビルド構成の扱い: init script オーバーレイ

プロジェクトのビルドファイルは一切編集せず、Gradle の `--init-script` で外部から差し替える:

1. **コンパイラと言語レベル**: ステップ/ゲートごとに `options.release`、toolchain、ECJ→javac 切替を init script 側で構成
2. **依存の追加**(JAXB → `jakarta.xml.bind` 等、JDK 削除モジュールの外部依存化): 移行作業中は init script で注入。ツールが Maven Central から取得。javax → jakarta のパッケージ名変更は影響が大きいため、可能な限り javax 互換版(jaxb-api 2.x 系)を選択する
3. **クラスパスダンプ**: ベースライン時に init script でカスタムタスクを注入し、ソースセット・コンパイル/テストクラスパスを抽出。以後のテスト実行系(5 章)はこれを使う

移行完了時に「**ビルド変更レポート**」(注入していた内容と等価な恒久変更点+根拠 knowledge_id)を出力し、人間が build.gradle へ反映する。依存ポリシーは「**注入は自動、恒久化は人間**」。

- 既存ライブラリのバージョンアップ(新 JDK 非対応のため): **人間承認制**。ツールは必要性検出と候補提示まで
- ライブラリの別製品への置き換え: **禁止**(スコープ外、エスカレーション)
- annotation processor・コード生成(ANTLR 等)を伴うビルドはベースラインダンプ時に検出してエスカレーション

利点: 移行中いつでも「元の構成でビルドすれば現行システムがそのまま出る」ことが保たれる。

## 9. 人間の関与モデル

### 9.1 ゲート承認(正常系の主経路)

各バージョンステップ完了時に停止し、人間がそのステップの全パッチを一括レビューする。レビュー対象は Git のステップ差分(前ゲートタグ → 現在)を **PR** として提示し、ツールが修正ごとの説明コメントを生成する:

> **症状**(コンパイルエラー/テスト失敗ログ)→ **根拠**(引用 KB エントリと出典 URL)→ **対処**(diff)→ **検証結果**(コンパイル・テスト通過)

- 承認は patches テーブルに `approved_by` / `approved_at` を記録
- 灰色フラグ付きパッチ(7.3 節)はレビューで重点確認
- 棄却されたパッチは revert コミットで巻き戻し(`reverted` 記録)、該当ファイルをステップ内で再処理

### 9.2 エスカレーション

発生条件: リトライ 3 回超過 / KB 該当なし / トークン予算超過 / 依存ライブラリ変更が必要 / ビルド構成の特殊性検出。

CLI でエスカレーションレポート(Markdown: 症状・試行パッチ・reject 理由・引用 KB)を確認し、対応は 2 択:

- `upgrade resolve <id> --hint "..."`: 修正方針テキストを内側ループの interrupt に流し込んで再試行
- `upgrade resolve <id> --manual`: 人間がソースを直接編集・コミット(`escalation_id` 参照)し、検証キューに戻す

### 9.3 再ベースライン承認

6.5 節。escalations の 1 類型として扱う。

### 9.4 手動修正と validate_diff

人間のコミットは **validate_diff を通さない**(人間の判断は原則 2 の埒外)。ただしコンパイル・テスト検証は同じゲートを通る。責任は承認した人間にある。

### 9.5 操作インターフェース(CLI ファースト)

```
upgrade init / baseline / run / status      # 工程の起動・監視
upgrade approve-gate                        # PR 承認の取り込み → 次ステップへ
upgrade escalations / resolve <id> ...      # エスカレーション一覧・解決
```

独自 Web UI は作らない。「エディタと Git と PR」という既存の仕事場に情報を届ける。

## 10. ナレッジベース

### 10.1 正本と実行時インデックス

- エントリの正本は **1 件 1 YAML ファイル**としてツールのリポジトリに置き、Git でレビュー・版管理する(KB はツールの資産)
- ツール起動時に SQLite へロードし embedding を計算・キャッシュ。knowledge テーブルは実行時インデックス

### 10.2 エントリ構造(v1 からの補強)

- `error_pattern`: 機械照合用の正規表現(例: `package javax\.xml\.bind does not exist`)。検索 1 段目に使う
- `gate`: `compiler_diff` / `runtime` / `compile`(4 章のゲート分解に対応)
- `category`: removed_api / behavior_change / module / tool / **incidental_behavior**(再ベースライン由来の非保証挙動)
- `allowed_changes`: validate_diff の機械判定に使う構造化された許可変更リスト
- `symptom` / `fix_guide` / `source_url`(v1 と同じ)

### 10.3 作成と保守

- **半自動ハーベスト**: 公式移行ガイド・リリースノート・JEP を LLM に読ませてエントリ草案を抽出し、**人間が 1 件ずつレビューして正本化**。出典 URL の実在・該当記述は機械チェック(捏造検出)
- エスカレーション解決時、内容を**エントリ草案として自動起票**(承認は人間)。escalations → knowledge のフィードバックループ
- 検索は 2 段構え: ① `error_pattern` の正規表現マッチ(高速・高精度)→ ② ヒットしない場合にベクトル検索
- 機械変換可能なパターンは将来的に OpenRewrite レシピへ切り出すハイブリッド構成に拡張可能な設計とする

## 11. LLM 構成

### 11.1 役割別モデルティア

| 用途 | 要求 | ティア |
|------|------|--------|
| テスト入力生成 | 多様性重視。誤りは Golden Master 側で弾かれる | 中位 |
| エラー分析+パッチ生成 | 最重要。精度がリトライ回数=コストに直結 | 上位 |
| グレーゾーン審査 | 保守的判断。結論は人間レビュー行き | 中位 |
| KB ハーベスト | オフライン一括、人間レビュー前提 | 中位 |

基準構成(リファレンス)は Anthropic API とし、LangChain の抽象でプロバイダ差し替え可能にする。チューニングは基準構成で行う。

### 11.2 再現性

- `temperature 0`
- 全呼び出しのプロンプト全文・応答全文・モデル ID・パラメータを **llm_calls テーブル**に保存(「なぜこのパッチが出たか」の追跡がゲートレビューの生命線)

### 11.3 コスト制御

- ファイル×ステップ単位のトークン予算。超過はリトライ閾値と同様にエスカレーション
- 成功指標としてファイルあたりトークンコストを計測(15 章)

## 12. SQLite スキーマ v2

v1 からの主変更: `project` 追加(ステップ管理のプロジェクト単位化)、`files.current_ver` 廃止、`llm_calls` 追加、`knowledge` 補強、承認・ゲート・イメージタグの記録。

```sql
-- プロジェクト状態(バージョン進行の管理単位)
CREATE TABLE project (
    project_id   INTEGER PRIMARY KEY,
    repo_path    TEXT NOT NULL,
    baseline_jvm TEXT NOT NULL,             -- 本番稼働 JVM(記録 JVM)
    current_step TEXT NOT NULL,             -- '7->8' 等
    current_gate TEXT NOT NULL,             -- '0' / 'A' / 'B'
    status       TEXT NOT NULL,             -- baselining / upgrading / awaiting_approval / done
    image_tag    TEXT NOT NULL,             -- 実行環境の指紋
    created_at   TEXT DEFAULT (datetime('now')),
    updated_at   TEXT DEFAULT (datetime('now'))
);

-- 対象ファイル台帳(ステップ内の作業状態のみ。到達バージョンは持たない)
CREATE TABLE files (
    file_id      INTEGER PRIMARY KEY,
    project_id   INTEGER NOT NULL REFERENCES project(project_id),
    path         TEXT NOT NULL,
    subproject   TEXT,                      -- マルチモジュール時のグルーピング
    testability  TEXT,                      -- all_safe / partial / boundary(静的ラベル)
    status       TEXT NOT NULL DEFAULT 'pending',
                 -- pending / fixing / fixed / escalated(現在ゲート内での状態)
    UNIQUE(project_id, path)
);

-- 修正履歴(全 diff を保存し監査可能にする)
CREATE TABLE patches (
    patch_id     INTEGER PRIMARY KEY,
    file_id      INTEGER NOT NULL REFERENCES files(file_id),
    step         TEXT NOT NULL,             -- '8->11' 等
    gate         TEXT NOT NULL,
    diff         TEXT NOT NULL,             -- 適用後に生成した正規 unified diff
    knowledge_id INTEGER REFERENCES knowledge(knowledge_id),
    gray_flag    INTEGER NOT NULL DEFAULT 0, -- LLM 従判定で通過(要重点レビュー)
    git_commit   TEXT,                      -- 1 パッチ 1 コミット
    result       TEXT NOT NULL,             -- applied / rejected / reverted
    reject_reason TEXT,
    approved_by  TEXT,                      -- ゲート承認の記録
    approved_at  TEXT,
    created_at   TEXT DEFAULT (datetime('now'))
);

-- コンパイル・テスト実行結果
CREATE TABLE runs (
    run_id       INTEGER PRIMARY KEY,
    project_id   INTEGER NOT NULL REFERENCES project(project_id),
    step         TEXT NOT NULL,
    gate         TEXT NOT NULL,
    run_type     TEXT NOT NULL,             -- compile / test / record(Golden Master 記録)
    jvm          TEXT NOT NULL,
    image_tag    TEXT NOT NULL,
    success      INTEGER NOT NULL,
    log          TEXT,
    created_at   TEXT DEFAULT (datetime('now'))
);

-- ナレッジ実行時インデックス(正本は YAML)
CREATE TABLE knowledge (
    knowledge_id INTEGER PRIMARY KEY,
    yaml_id      TEXT NOT NULL UNIQUE,      -- 正本ファイルへの参照
    step         TEXT NOT NULL,
    gate         TEXT NOT NULL,             -- compiler_diff / runtime / compile
    category     TEXT NOT NULL,             -- removed_api / behavior_change / module / tool / incidental_behavior
    error_pattern TEXT,                     -- 1 段目検索用の正規表現
    symptom      TEXT NOT NULL,
    fix_guide    TEXT NOT NULL,
    allowed_changes TEXT NOT NULL,          -- validate_diff 機械判定用(JSON)
    source_url   TEXT NOT NULL,
    embedding    BLOB
);

-- LLM 呼び出しの完全記録
CREATE TABLE llm_calls (
    call_id      INTEGER PRIMARY KEY,
    purpose      TEXT NOT NULL,             -- input_gen / patch_gen / gray_review / harvest
    patch_id     INTEGER REFERENCES patches(patch_id),
    model_id     TEXT NOT NULL,
    params       TEXT NOT NULL,             -- JSON(temperature 等)
    prompt       TEXT NOT NULL,
    response     TEXT NOT NULL,
    tokens_in    INTEGER,
    tokens_out   INTEGER,
    created_at   TEXT DEFAULT (datetime('now'))
);

-- 人間エスカレーション記録
CREATE TABLE escalations (
    escalation_id INTEGER PRIMARY KEY,
    file_id      INTEGER REFERENCES files(file_id),
    step         TEXT NOT NULL,
    gate         TEXT NOT NULL,
    type         TEXT NOT NULL,             -- retry_exceeded / no_knowledge / budget /
                                            -- dependency / build_config / rebaseline
    reason       TEXT NOT NULL,
    context      TEXT,                      -- エラーログ・試行 diff
    resolution   TEXT,                      -- hint / manual / rebaseline 承認内容
    kb_draft_id  TEXT,                      -- 解決から自動起票された KB 草案
    status       TEXT NOT NULL DEFAULT 'open',
    created_at   TEXT DEFAULT (datetime('now'))
);
```

LangGraph の SqliteSaver チェックポイントは内側修正ループ専用(thread_id = ファイル×ステップ×ゲート)。完了した事実は業務テーブルが正。

## 13. Git 運用

1. ツール起動時に対象リポジトリの指定コミットから**ツール専用ブランチ**(`upgrade/7-to-21`)を切る
2. **1 適用パッチ = 1 コミット**。コミットメッセージに patch_id / knowledge_id / ステップ・ゲートを構造化して埋め込む(SQLite と Git の相互参照)
3. **ゲート承認ごとにタグ**(`upgrade/java8-approved` 等)。棄却は revert コミット
4. 凍結テストは `characterization-tests/` に隔離し、凍結後の変更を機械検出
5. ステップ差分(前ゲートタグ → 現在)がそのままゲートレビューの PR になる

## 14. 開発フェーズ計画

| フェーズ | 内容 | 完了条件 |
|---------|------|---------|
| P1 基盤 | Docker イメージ(5 JDK 固定・JUnit 4・JavaParser ヘルパー)、SQLite スキーマ v2、init script オーバーレイ+クラスパスダンプ、JUnitCore 直接 fork 実行系、CLI 骨格 | サンプル Gradle プロジェクトが全ゲート構成(ECJ 互換レベル 7 → javac 各バージョン)でコンパイルでき、手書きテストが 5 JDK 全てで直接 fork 実行できる |
| P2 テスト生成 | 呼び出しグラフ、境界パス判定+SecurityManager サンドボックス、LLM 入力生成 → Golden Master 記録、非本質挙動除外規則、バイトコード凍結 | サンプルに対し末端から Characterization Test が自動生成され、保証範囲(パス単位カバレッジ)がレポートされる |
| P3 ナレッジ基盤(P2 と並行可) | YAML 正本+ローダ+embedding、2 段検索、半自動ハーベスト、8→11 区間の先行整備(30〜50 件) | 8→11 の代表的エラーが 1 段目検索でヒットする |
| P4 修正ループ | 内側 LangGraph、機械判定 validate_diff、SEARCH/REPLACE 適用、1 パッチ 1 コミット、リトライ/トークン予算/エスカレーション | 単一ファイルの典型的非互換(JAXB 等)が自動修正され、根拠付きコミットが残る |
| P5 工程制御 | 外側オーケストレーション、ゲート PR 生成+承認フロー、エスカレーション CLI、再ベースライン承認、中断再開 | サンプルプロジェクトが 7→21 を E2E 完走(人為的エスカレーション・ゲート承認を挟んで) |
| P6 パイロット | 実システム(Gradle 9.2 + ECJ 構成)への適用、残り区間の KB 整備(7→8 / 11→17 / 17→21 / compiler_diff)、精度・コストチューニング | 実システムで最初のゲート(ゲート 0: ECJ→javac)を人間承認込みで通過 |

## 15. 成功指標

- **必達**: 凍結した JUnit テスト(同一バイトコード)が Java 21 上で全件パスすること。保証は**カバーされたパスの範囲**に対するものであることを明記する
- **保証範囲率**(第一級の報告値): auto-testable と判定されたパスの行・分岐カバレッジ(JaCoCo)
- 自動完走率(エスカレーションなしで 7→21 到達したファイルの割合)
- validate_diff の reject 率(LLM の過剰修正の監視指標)
- **再ベースライン件数**(層 1 除外規則の品質監視。増えるなら 6.4 節の規則を強化する)
- 1 ファイルあたりの平均処理時間・LLM トークンコスト

## 16. リスクと対策

| リスク | 対策 |
|--------|------|
| テストが振る舞いを十分にカバーできない | パス単位カバレッジ(JaCoCo)を計測し保証範囲として常時可視化。閾値未満は入力生成の追加または人間確認 |
| LLM がテストを修正して「通す」 | テストはバイトコード凍結+`characterization-tests/` 隔離+変更の機械検出。再ベースラインは人間承認限定 |
| Golden Master が非本質挙動を固定する | 層 1(除外規則+JVM 環境固定)+層 2(人間承認の再ベースライン+KB フィードバック) |
| レガシーコードの実行が危険な副作用を起こす | SecurityManager サンドボックス+Docker のFS/ネットワーク隔離の二重防壁 |
| ファイル単位で解決しない横断的変更(モジュール化等) | エスカレーションで人間対応。ツールのスコープは「ファイル単位で閉じる修正」 |
| 実行時のみ顕在化する挙動差 | ゲート A(ランタイム先行)で分離して検出。runs にログ全保存、analyze 時にエラーログを RAG クエリへ |
| ECJ と javac のコンパイラ差が Java 非互換と混同される | ゲート 0 で隔離し、KB の gate=compiler_diff として別分類 |
| LLM 審査の甘さが validate_diff の抜け穴になる | 機械判定を主、LLM は従。グレーは自動適用せず灰色フラグでゲートレビューへ |
| 再生成ループの暴走によるコスト超過 | ファイル×ステップのトークン予算+超過エスカレーション |
| 特殊なビルド(独自コンパイルタスク・APT・コード生成) | ベースライン検証で検出してエスカレーション |

## 付録: 決定記録(v1 からの主要変更)

| # | 決定 | 理由の要点 |
|---|------|-----------|
| 1 | 汎用ツール、ただし Gradle のみ | スコープの確定 |
| 2 | バージョン進行はプロジェクト単位、ファイル単位の到達バージョン管理は廃止 | Java のコンパイル単位はプロジェクトであり、ファイル別バージョンは検証可能な実行状態として存在しない |
| 3 | Golden Master 方式(LLM は入力生成のみ、期待値は実行結果) | LLM 推測の期待値はリトライの収束先が結局実測値になる。Characterization Test の定義は「現状の挙動の固定」 |
| 4 | 呼び出しグラフを構築し末端からテスト構築 | 上位テストが検証済み下位実装を使える |
| 5 | 境界コードはパス単位判定+SecurityManager サンドボックス | メソッド単位二値ではカバレッジが壊滅。サンドボックスは安全装置としても必須 |
| 6 | ベースラインのみ既存 Gradle 9.2+ECJ、8 以降は OpenJDK javac | 対象システムの現構成は維持しつつ、以後は標準ビルドへ |
| 7 | ステップをゲート 0 / A / B に分解 | コンパイラ差・ランタイム差・コンパイル非互換を 1 つずつ顕在化させ、根拠ベース修正を成立させる |
| 8 | init script オーバーレイ、プロジェクトのビルドファイル不変更 | 汎用性と「いつでも元の構成でビルドできる」安全性。恒久化はビルド変更レポートで人間が実施 |
| 9 | 非本質挙動の 2 層対策+人間承認限定の再ベースライン | 凍結原則の唯一の例外を正式なワークフローとして定義 |
| 10 | ゲート単位の人間一括承認(PR 形式) | 症状→根拠→対処→検証結果をステップの文脈のまとまりでレビューできる |
| 11 | マルチモジュールは全サブプロジェクト一斉 | 実行 JVM は結局 1 つ。段階進行は制約管理コストだけが乗る |
| 12 | KB は YAML 正本+SQLite インデックス、半自動ハーベスト+人間レビュー | KB はツールの資産。レビュー・版管理・共有可能に |
| 13 | validate_diff は JavaParser ヘルパー+機械判定主・LLM 従、SEARCH/REPLACE 形式 | javalang はメンテ停止。LLM 同士の審査は抜け穴になる |
| 14 | モデルティア+temperature 0+llm_calls 全記録+トークン予算 | 監査可能性とコスト暴走防止 |
| 15 | Docker 単一イメージ(5 JDK 固定、linux/amd64) | JDK 7 の入手性と JVM ビルドレベルまでの再現性。arm64 ビルドは存在しない |
| 16 | Git ブランチ+1 パッチ 1 コミット+ゲートタグ、テストはバイトコード凍結 | 巻き戻しは Git に任せる。ゲートレビューが普通の PR になる |
| 17 | LangGraph は内側修正ループのみ、外側は素の Python+SQLite が正 | 正を 2 つにしない。決定的な工程管理にエージェント基盤は不要 |
| 18 | CLI ファースト、独自 UI なし。手動修正は validate_diff 免除 | 既存の仕事場(エディタ・Git・PR)に情報を届ける。人間の判断は原則 2 の埒外 |
