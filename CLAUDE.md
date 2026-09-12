# Claude Safety Rules

## 削除系コマンドの禁止（重要）

以下のルールはこのワークスペース内のすべての会話で絶対に守られる：

- Claude はファイルまたはディレクトリを削除するコマンドを一切生成してはならない。
  例：rm, rm -rf, rm *, rmdir, unlink, cache --delete,
      lftp mirror --delete, rsync --delete, git clean -df, find -delete 等。

- 削除が必要な場合でも、Claude は削除コマンドを提案せず、
  「手動で削除してください」といった説明に留めること。

- 削除の推奨・削除操作の自動判断も禁止。

- ssh / lftp / デプロイ系スクリプトを生成する場合でも、
  削除コマンドの生成は禁止。

これらはすべての会話・コード生成に適用される。

## シークレット管理（重要）

- `config/master.key` など機密ファイルを `git add` するコードを生成してはならない
- デプロイスクリプト・セットアップ手順でも同様
- シークレットは必ず環境変数（RAILS_MASTER_KEY 等）で渡すこと
- `.gitignore` への追加を確認する手順を必ずコードに含めること
- 初回コミット前に `git status` でステージング確認を促すこと

---

## プロジェクト概要

小さな島の電力司令室の運転員として、天気予報（確率で示される）を読んで次の時間帯の天候（空・風）を予測し、火力出力・蓄電池の充放電・需要調整であらかじめ電源計画を立てる、Unity WebGL のルールベース需給シミュレーションゲームのデモ版展示物。**仕様の正は [requirements.md](requirements.md)。** 本ファイルはその要約であり、数値・アルゴリズムの詳細（予報の混合比、係数表、精算の順序、定数値）は必ず requirements.md を参照する。

- サーバー・DB・外部通信・外部APIを一切持たない。ブラウザ内で完結する。天候・需要はシード付き乱数で生成し、実在の気象データ・気象APIは用いない。
- 認証なし。ブラウザ内で不透明なオーナーIDを生成し、PlayerPrefs（WebGLではIndexedDB経由）の全キーの接頭辞にする。
- 「おまかせ計画」は費用の安い順に電源を充てるルールベースの充当手順。学習・推論モデル・外部AI APIは使用しない。
- デプロイ先は Unity Play のみ。独自ドメイン・Vercel/Railway/Cloudflare 等の Web インフラは持たない。

## 開発コマンド

- **ビルド**（CLI・バッチモード）：
  ```
  Unity.exe -batchmode -nographics -executeMethod BuildScript.BuildWebGL -quit
  ```
  `BuildScript.cs` は `Assets/Editor/` に置く（未実装。requirements.md 13.5節）。`-quit` を付けないとメソッド完了後もUnityが終了せずバッチプロセスがハングし続ける。
- **テスト**：Unity Test Framework（NUnit）を使う。CLIでも実行できる（例：`Unity.exe -batchmode -nographics -projectPath . -runTests -testPlatform EditMode -testResults results.xml`）。単一テストのみ実行する場合は `-testFilter <クラス名または完全修飾テスト名>` を付ける。
- Unity Editor のライセンス有効化・実機ブラウザ確認の手順は Claude Desktop 側の `.claude/agents/unity-dev.md`（`20_開発`ワークスペース側）を参照。このリポジトリ単体には持たない。

## アーキテクチャ

すべて Unity WebGL・C# で完結し、シーン・UI・棒グラフ・ゲージはすべて `GameBootstrap` が起動時にコードで動的生成する（Prefab・シーンファイルの手作業編集を前提としない。requirements.md 13.3節）。

リポジトリ構成（未実装。requirements.md 13.5節が正）：

```
Assets/Scripts/
├── Core/      # Game・Grid・Plan・Weather・SlotForecast・Distribution・SlotRecord（データモデル）
├── Logic/     # ForecastBuilder・PlanValidator・BalanceProjector・AutoPlanner・Settler・Progression・GameFinalizer
├── Data/      # ForecastTable・定数（数値定数は5.11節の表に1箇所へ集約する）
├── Infra/     # Persistence（PlayerPrefs保存）・SplitRng（乱数系列分離）
├── UI/        # UIPresenter・各画面の動的生成
└── Bootstrap/ # GameBootstrap
Assets/Editor/BuildScript.cs  # CLIビルド用
```

### コアループ（関数A〜Jの責務。詳細は requirements.md 5章）

1. **`initGame`（関数A）**：シードから気圧配置・空・風の系列W、需要のぶれの系列D、予備の系列R（デモ版では未使用）を独立に生成する（`SplitRng`）。系列を分けることで、プレイヤーの操作順が天候・需要の結果に影響しない。
2. **`buildForecast`（関数B, `ForecastBuilder`）**：気圧配置と時間帯から日次予報の分布を取り出し、そこから実況を抽選する。直前予報は「日次予報×0.5＋実況に全確率×0.5」を10%単位に丸めて作る（最大剰余法）。**日次予報の生成と実況の抽選は同じ分布表を参照する**——予報は実況の分布を誇張・過小にしない（requirements.md 13.1節）。
3. **`validatePlan`（関数C, `PlanValidator`）**：火力の変化幅（直前出力±40）・蓄電池の残量／空き容量・需要調整の残回数を超える入力を、黙って補正せず理由付きで補正する。UIの範囲制限が一次防御、確定時の検証が最終防衛という二段構え（requirements.md 5.3節・13.1節）。
4. **`projectBalance`（関数D, `BalanceProjector`）**：宣言した天候での供給内訳・需要側内訳・計画インバランスを即時計算する。**計画バランス（予測）と需給精算（実績）は宣言した天候か実況かの違いのみで同じ計算式を使う**——恣意的に別ロジックを作らない。
5. **`autoPlan`（関数E, `AutoPlanner`）**：宣言した天候で計画インバランスが許容幅（±5MW）に収まるよう、**火力→放電→需要調整**の順（供給不足時）／**火力下限→充電**の順（供給過多時）で費用の安い電源から充てる。生成した計画は必ず `validatePlan` を通す。
6. **`settleSlot`（関数F, `Settler`）**：実況・需要のぶれを取り、供給・実需要・インバランスを計算する。精算は**許容幅→緊急電源→停電**（不足側）、**許容幅→出力抑制→過剰供給**（余剰側）の固定順で行う。出力抑制の上限は太陽光＋風力の実出力までとする（requirements.md 5.6節）。
7. **`advance`（関数G, `Progression`）**・**`finalizeGame`（関数H, `GameFinalizer`）**・**`persist`/`resetIfNeeded`（関数I, `Persistence`）**：それぞれ時間帯・日の進行と日次まとめ、総費用・ランク判定・的中率集計、PlayerPrefs保存とJST 03:00境界での日次リセットを担う。
8. **`onPlanChanged`（関数J）**：計画パネル・宣言パネルの操作ごとに `validatePlan` → `projectBalance` を再実行し、バランス表示を即時更新する。

### 設計上の不変条件（requirements.md 13.1節。実装・レビュー時に必ず確認する）

- 実況・直前予報・需要のぶれは初期化時に20時間帯分をまとめて確定し、プレイ中に乱数を消費しない（同一シード＝同一系列の再現）。
- 日次予報の分布表がそのまま抽選の分布であり、確率0の要素は直前予報でも0のまま。
- 計画の補正は必ず理由を表示し、黙って値を変えない。
- 精算は「許容幅→緊急電源→停電」「許容幅→出力抑制→過剰供給」の固定順。出力抑制の上限は再生可能電源の実出力。
- 確定・精算・状態更新は1操作内で連続して行い、途中の状態を保存しない。保存は時間帯の境界のみ。
- おまかせ計画は火力→放電→需要調整の順に充て、生成した計画は必ず計画検証を通す。

状態遷移（ゲーム全体）は `LOADING → TITLE → PLANNING ⇄（宣言・計画変更）→ SETTLED → PLANNING`（朝・昼・夕の後）または `→ DAY_SUMMARY`（夜の後）`→ PLANNING`（4日目以前）または `→ RESULT`（5日目）。詳細と時間帯・蓄電池・火力個別の状態遷移図は requirements.md 11章。

## 開発フロー

- ブランチ運用：`Assets/**` の変更は必ずブランチを切って PR を作成する（main への直接 push 禁止）。`Assets/**` 以外（ドキュメント・`SPEC/`・`TASKS/`等）は main への直接 push を許可する。
- 1 issue のワンショット実装（requirements.md 1.4節）。
- リリースフロー（デモ版）：`issue → setting & coding → security review → add, commit, push → reviewer & pr-checker → merge →（Unity Playへの手動アップロード）→ user test`。code-review・audit・security-gate・正式release・reportは省略する。Unity WebGL は PR の merge では自動デプロイされないため、merge の直後に手動アップロード工程を挟む。
- コミット前にセキュリティレビューを行う。マージ前に reviewer・pr-checker を実行する。
- バージョン番号は `メジャー2桁.マイナー2桁.デバッグ2桁`（`01.01.00` が初期値）。タグは注釈付き（`git tag -a`）。