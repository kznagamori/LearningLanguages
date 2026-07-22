# LearningLanguages 生成パイプライン

このリポジトリは、`languages-curriculum.md` を最上流ソースとして、指定した 1 つのプログラミング言語に専用化した学習仕様、Learning MISRA 仕様、ハンズオン学習ドキュメント、実行可能なサンプルコード、CI/CD ワークフローを生成するためのものです。

生成物は、初心者が説明を読むだけで終わらず、コードを最初から段階的に実装し、各段階で動作を確認しながら、CLI ツール、デスクトップ GUI アプリ、サーバ / Web API、データ処理パイプラインなどのビジネスアプリケーションを自力で作成・拡張できるようになることを目的とします。

## 対象プラットフォーム

学習ドキュメント、サンプルプロジェクト、検証手順の対象配分は、次のとおりです。割合はプラットフォーム固有の説明、実装、トラブルシューティング、実行確認に割り当てる重点度を示し、言語共通の説明を無理に行数比で分割するものではありません。

- Windows x86_64: **70%**
- Linux x86_64: **10%**（正式対象は Ubuntu 24.04 LTS。WSL2 の Debian は開発環境として継続使用）
- Raspberry Pi 4 以降の ARM64: **20%**（Raspberry Pi OS 64-bit および Ubuntu 24.04 LTS 64-bit）

Raspberry Pi は GPIO、I²C、SPI、UART、PWM などを扱う組み込み機器教材ではなく、ARM64 Linux 上で汎用アプリケーションをビルド、実行、テスト、配布する対象として扱います。Windows 上の MobaXterm から SSH で接続し、教材で作成した単体 GUI アプリケーションは X11 フォワーディングで目視確認します。Raspberry Pi のデスクトップ全体の遠隔操作は対象外です。

正式ターゲットは、Windows x86_64、Ubuntu 24.04 LTS x86_64、Raspberry Pi OS 64-bit ARM64、Ubuntu 24.04 LTS ARM64 の 4 実行環境です。実行可能な項目は、プラットフォーム固有であることが学習目標の項目を除き、原則として 4 環境すべてでビルド・テスト・実行できることを完成条件とします。技術的に対応できない環境がある場合は黙って対象外にせず、理由、代替案、影響範囲を提示してユーザ確認を得たうえで、選択言語用の `<出力ディレクトリ>/master-spec.md` に例外を記録します。

GitHub Actions では、Raspberry Pi 4 以降の実機をセルフホステッドランナーとして使うネイティブビルド・テストを基本とし、CI での ARM64 クロスビルドも併記します。実機は 1 台とし、Raspberry Pi OS と Ubuntu 24.04 LTS を切り替えて順番に検証するため、両 OS の同時実行を前提にしません。2 OS の結果は同じコミット、ツールチェーン、テスト条件に対して取得します。クロスビルド成果物は GitHub Actions のアーティファクトとして保存し、対象 OS で起動したセルフホステッドランナーが取得して実機テストします。GitHub ホステッドランナーから Raspberry Pi への外部向け SSH 接続は必須にしません。クロスビルドの成功だけで Raspberry Pi 対応済みとはせず、成果物を実機へ配置して起動・テストするところまで確認します。GUI は CI の仮想ディスプレイ上で自動テストし、MobaXterm の X11 フォワーディングで実画面を補完確認します。

macOS arm64 は上記の配分外の補助サポートとし、CI ビルドと Mac 所有者向けのデバッグ手順を維持します。具体的なツール、バージョン、ターゲットトリプル、ランナーラベル、クロスコンパイル方法は、選択言語用の `<出力ディレクトリ>/master-spec.md` で確定します。

生成パイプラインは次の順序で進めます。

```text
languages-curriculum.md
  -> target_language を 1 言語確認
  -> <出力ディレクトリ>/output-spec.md
  -> <出力ディレクトリ>/master-spec.md
  -> <出力ディレクトリ>/misra-spec.md
  -> <出力ディレクトリ>/spec.md
     + prompts/generation/<target_language>/generate-item.md
  -> CLAUDE.md / AGENTS.md
  -> 項目単位の学習ドキュメント
     + サンプルコード / サンプルプロジェクト / CI/CD ワークフロー
```

`output-spec.md`、`master-spec.md`、`misra-spec.md`、`spec.md` は共通ファイルとしてリポジトリ直下に作成しません。すべて選択言語の出力ディレクトリ内に作成し、1 ファイルに複数言語の決定や例を混在させません。共通仕様雛形と共通の Learning MISRA 成果物も作成しません。

## プロンプト配置

プロンプトは用途別にディレクトリを分けます。

- `prompts/pipeline/`: 1 言語パイプラインを開始する前の制御・仕様派生・検証用プロンプト
- `prompts/generation/<target_language>/`: 選択言語専用の項目生成プロンプト

`prompts/pipeline/` のプロンプト自体は複数言語から共通利用しますが、1 回の実行では必ず `target_language` を 1 言語に固定します。上流の `README.md` と `languages-curriculum.md` は共通入力として参照し、言語別仕様・docs・projects は選択言語の出力ディレクトリだけへ生成します。定義済みの生成用プロンプトとリポジトリ直下のエージェントコンテキストを除き、別言語の出力ディレクトリを読み書きしません。

`README.md` は操作説明に限定し、AI コードエージェントへ入力する実プロンプトは `prompts/` 配下のファイルとして管理します。

## 初回ブートストラップ

初期状態で `prompts/pipeline/` が存在しない場合は、通常パイプラインを開始する前に、AI コードエージェントへ `README.md` と `languages-curriculum.md` を入力し、本 README の手順と `languages-curriculum.md` の「パイプラインプロンプトの初回ブートストラップ契約」に従って次の 9 ファイルだけを生成させます。

- `prompts/pipeline/00-common-control.md`
- `prompts/pipeline/01-derive-output-spec.md`
- `prompts/pipeline/02-derive-master-spec.md`
- `prompts/pipeline/03-derive-misra-spec.md`
- `prompts/pipeline/04-run-language-pipeline.md`
- `prompts/pipeline/05-generate-agent-context.md`
- `prompts/pipeline/80-enable-language.md`
- `prompts/pipeline/90-audit.md`
- `prompts/pipeline/91-verify.md`

ブートストラップでは、言語別仕様、学習ドキュメント、サンプルプロジェクト、`CLAUDE.md`、`AGENTS.md` を生成しません。上記 9 ファイルがすでに存在する場合は、欠落・名称・責務・入出力契約を監査し、上流ルールが変わったファイルだけを更新します。ブートストラップ完了後に、次の通常実行へ進みます。

## 実行順

AI コードエージェントをリポジトリルートで起動し、次の順にプロンプトを入力してください。対象言語表で状態が `対応中` の言語を生成する通常実行を示します。

1. `prompts/pipeline/00-common-control.md` を入力し、`target_language` を 1 言語確認する
2. `prompts/pipeline/01-derive-output-spec.md` で `<出力ディレクトリ>/output-spec.md` を作成する
3. `prompts/pipeline/02-derive-master-spec.md` で `<出力ディレクトリ>/master-spec.md` を作成する
4. `prompts/pipeline/03-derive-misra-spec.md` で `<出力ディレクトリ>/misra-spec.md` を作成する
5. `prompts/pipeline/04-run-language-pipeline.md` で `<出力ディレクトリ>/spec.md` と `prompts/generation/<target_language>/generate-item.md` を作成する
6. `prompts/pipeline/05-generate-agent-context.md` でリポジトリ直下の `CLAUDE.md` と `AGENTS.md` を作成する
7. `prompts/generation/<target_language>/generate-item.md` を、1 項目について入力する
8. ユーザが生成された学習ドキュメントとプロジェクトを確認し、承認または修正指示を行う
9. 承認後、`prompts/pipeline/91-verify.md` を同じ項目 ID に対して実行し、最終プロジェクトだけでなく、学習ドキュメントの手順を空の検証用作業場所から順番に再現できることを確認する
10. 必要に応じて `prompts/pipeline/90-audit.md` を同じ項目 ID に対して実行する

手順 7〜10 は、`<出力ディレクトリ>/output-spec.md` と `<出力ディレクトリ>/spec.md` に定義された各項目について繰り返します。ユーザ未承認、`91-verify.md` 不合格、または `90-audit.md` の未解決指摘がある場合は、次項目へ進まず同じ項目を `generate-item.md` で再生成します。

`output-spec.md` には、全項目の ID、docs/projects パス、前提項目、生成順、対象プラットフォーム、検証方式、DoD、章別・種別の正確な件数を持つ生成対象台帳を作成します。アルゴリズムのカテゴリ名や未確定項目が残る場合、第 8 / 10 / 11 章の全 34 アプリが揃わない場合、または第 12 章の Raspberry Pi CI/CD 追加 4 項目が揃わない場合は、`spec.md` と `generate-item.md` の生成へ進みません。

言語非依存の共通仕様雛形を生成する段階は設けません。言語別 `spec.md` は、`languages-curriculum.md` と選択言語用の `output-spec.md`、`master-spec.md`、`misra-spec.md` から直接派生します。`misra-spec.md` は、`master-spec.md` で確定したツールチェーン、Linter、Formatter、静的解析ツールを参照して検出方法を具体化し、同じ設定を重複決定しません。

`CLAUDE.md` と `AGENTS.md` はリポジトリ直下に置きます。両ファイルは `README.md` と `languages-curriculum.md` の横断ルールから生成し、特定言語の設定や進捗を固定しません。実行時の `target_language` と言語別仕様ファイルのパスは `generate-item.md` から受け取ります。別言語のパイプラインを実行しただけでは両ファイルに差分を発生させません。

生成用プロンプトの出力先は次の形式です。

```text
prompts/generation/<target_language>/generate-item.md
```

例:

```text
prompts/generation/rust/generate-item.md
```

## 生成される学習ドキュメントの必須方針

すべてのハンズオン学習項目は、完成コードを先に提示するだけの説明ではなく、学習者が空の作業ディレクトリまたは直前段階から実装を積み上げる形式にします。各実装ステップには、目的、開始状態、作業ディレクトリ、対象 OS / シェル、操作、変更するファイルの編集後の完全な内容、コードの説明、必要な Mermaid 図、動作確認、期待結果、確認ポイント、失敗例を含めます。

動作の流れ、データフロー、呼び出し順序、状態遷移、非同期処理、エラー経路など、動作理解に関わる説明には Mermaid 図を必須とし、図の直後に図とコードの対応を文章で説明します。

アルゴリズム章は、各アルゴリズムまたはデータ構造を独立した学習項目にします。アルゴリズム自体の考え方、適用場面、図解、疑似コード、手計算トレース、変数遷移、計算量、実装コードとの対応、段階実装、テストを揃え、言語とアルゴリズムを同時に学べる内容にします。

サンプルアプリケーションは、第 8 章の 4 件、第 10 章の 12 件、第 11 章の 18 件（簡易タスク管理を 3 フレームワークで実装する 3 件を含む）の合計 34 件をすべて個別に生成します。各アプリは生成前に全機能、利用者の操作または入力、期待結果、受入条件、必要なテストを機能台帳へ確定します。ドキュメントでは要件・画面または入出力・データ・処理・コード・テストの対応を説明し、機能ごとに段階実装します。完成後には、学習者が機能追加や置き換えを行う拡張演習も含めます。

## 未対応言語の対応中化手順

対象言語表で状態が `対応中` 以外の言語を生成する場合は、通常実行の前に `prompts/pipeline/80-enable-language.md` を実行します。

1. `prompts/pipeline/00-common-control.md` を入力し、`target_language` を 1 言語確認する
2. 対象が `将来予定` または `要追加定義` なら、`prompts/pipeline/80-enable-language.md` を実行する
3. AI が不足している言語固有要求と決定候補を依存関係順に提示し、ユーザに 1 件ずつ確認する
4. 確定内容を `languages-curriculum.md` に反映し、未確定マーカーがないことを確認して対象言語を `対応中` にする
5. §「実行順」の手順 1 から通常実行を開始する

`80-enable-language.md` は、共通仕様ファイルや共通 MISRA ルールセットを作成しません。言語の対応中化に必要な最上流要求だけを `languages-curriculum.md` に反映します。対象言語用の `output-spec.md`、`master-spec.md`、`misra-spec.md` は通常実行で作成します。

## 未決定事項の扱い

AI コードエージェントは、ライブラリ、パッケージ、バージョン、API、OS 要件、テスト方針などの未決定事項を推測で確定してはいけません。

未決定事項がある場合は、候補一覧、推奨候補、推奨理由、影響範囲を提示し、ユーザに 1 件ずつ確認します。ユーザの回答を得てから次の質問へ進みます。

## 対応言語

対象 `target_language` の一覧、出力ディレクトリ、状態（`対応中` / `将来予定` / `要追加定義`）は、`languages-curriculum.md` §「対象言語の指定」の対象言語表を単一ソースとして参照します。本 README は重複した状態リストを保持しません。

## 出力先

主な出力先は次の通りです。

- 言語別出力仕様: `<出力ディレクトリ>/output-spec.md`
- 言語別決定事項: `<出力ディレクトリ>/master-spec.md`
- 言語別 Learning MISRA 仕様: `<出力ディレクトリ>/misra-spec.md`
- 言語別詳細仕様: `<出力ディレクトリ>/spec.md`
- 大規模生成用コンテキスト: リポジトリ直下の `CLAUDE.md` / `AGENTS.md`
- 学習ドキュメント: `<出力ディレクトリ>/docs/`
- サンプルコード / サンプルプロジェクト / CI/CD ワークフロー: `<出力ディレクトリ>/projects/`

`docs/` と `projects/` の内部構造は、選択言語用の `<出力ディレクトリ>/output-spec.md` を単一ソースとします。各章ディレクトリのトップとなる索引ドキュメントは `README.md` とし、`index.md` は生成しません。

`prompts/pipeline/` と `prompts/generation/` はプロンプト管理用であり、最終学習成果物の出力先ではありません。
