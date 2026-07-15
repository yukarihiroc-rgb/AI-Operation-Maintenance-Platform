# Power Automate基本設計

> 本ファイルはMarkdownを自動結合した統合版です。

---


<!-- ========================================= -->
<!-- 00_文書概要.md -->
<!-- ========================================= -->

# 00_文書概要

# 1. 文書概要

## 1.1 目的

本書は、AI運用保守プラットフォームにおける Power Automate の基本設計を定義することを目的とする。

Power Automate を利用した業務フロー、自動処理、AI連携、通知処理、エラーハンドリング、ログ管理、再試行制御、セキュリティおよび共通設計を定義し、個別フロー設計（06_01～06_08）の上位設計書として位置付ける。

本設計書は、開発担当者が追加説明なしで実装できる品質を目標とする。

---

## 1.2 適用範囲

本設計書の対象範囲は以下とする。

- Power Automate 全体の共通設計
- フロー設計方針
- フロー一覧
- 命名規則
- コネクタ接続設計
- 共通変数・環境変数
- CorrelationId・FlowRunId設計
- エラーハンドリング
- 再試行設計
- ログ設計
- セキュリティ設計
- 性能設計
- 業務ルール

個別フローの処理内容および業務ロジックは、以下の設計書で定義する。

- 06_01 メール受付フロー
- 06_02 AI解析フロー
- 06_03 案件化フロー
- 06_04 SLA管理フロー
- 06_05 Teams通知フロー
- 06_06 AI再解析フロー
- 06_07 システムエラー登録フロー
- 06_08 共通Child Flow

---

## 1.3 関連設計書

本設計書は以下の設計書を前提とする。

|設計書|概要|
|-------|------|
|01_アーキテクチャ設計|システム全体アーキテクチャ|
|01_システム構成設計|システム構成・サービス構成|
|02_データ論理設計|論理データモデル|
|03_データ物理設計|Microsoft Lists列設計|
|04_画面設計|Power Apps画面設計|
|07_権限制御|権限設計|
|08_エラーハンドリング|システム全体のエラー設計|

---

## 1.4 設計方針

本設計書では以下の設計方針を採用する。

- Power Automate を業務処理の実行基盤とする。
- Power Apps は画面制御のみを担当し、業務処理は実施しない。
- Microsoft Lists を Phase1 の唯一のデータストアとする。
- Outlook はメール受信を担当する。
- AI Builder は AI 判定・要約・提案のみを担当し、業務データを直接更新しない。
- Teams は通知のみを担当する。
- SharePoint Online は添付ファイル保存のみを担当する。
- CorrelationId により Power Apps・Power Automate・SystemError を一意に追跡できる設計とする。
- FlowRunId により Power Automate の実行履歴を追跡できる設計とする。
- Scope、Configure run after、Terminate を活用し、保守性・可読性の高いフローを設計する。
- Retry Policy を活用し、一時的な障害は自動回復する。
- AI処理失敗によって受付登録・案件登録などの業務処理を停止させない。
- Microsoft Learn のベストプラクティスを優先し、約20名規模の社内システムに適したシンプルで保守性の高い設計とする。

---

## 1.5 対象利用者

|利用者|用途|
|------|------|
|システム設計者|Power Automate全体設計|
|Power Platform開発者|Power Automate実装|
|Power Apps開発者|Power Apps連携設計|
|運用管理者|運用・保守設計の確認|
|システム開発者|障害解析・保守対応|

---

## 1.6 ドキュメント管理

本設計書は章単位で管理する。

各章は個別の Markdown ファイルとして作成し、レビュー完了後に統合版「06_00_Power Automate基本設計_Ver1.0.md」を生成する。

改善事項および設計上の指摘事項はレビュー管理台帳で管理し、正式版へ直接反映しない。

---

## 1.7 変更履歴

|版数|日付|内容|
|---|---|---|
|Ver1.0|YYYY/MM/DD|新規作成|



<!-- ========================================= -->
<!-- 01_Power Automate設計方針_改訂版.md -->
<!-- ========================================= -->

# 01_Power Automate設計方針

# 2. Power Automate設計方針

## 2.1 基本方針

Power Automate は AI運用保守プラットフォームにおける業務処理基盤として位置付ける。

Power Apps は画面表示および利用者操作を担当し、業務ロジックは Power Automate が実施する。

Microsoft Lists を Phase1 の唯一のデータストアとし、Power Automate がデータ更新を担当する。

AI Builder は判定・要約・提案のみを担当し、業務データを直接更新しない。

各フローは Microsoft Learn のベストプラクティスに従い、可読性・保守性・障害解析性を重視して設計する。

---

## 2.2 Power Platform コンポーネント責務

|コンポーネント|責務|
|-------------|------------------------------|
|Power Apps|画面表示・入力・画面遷移|
|Power Automate|業務処理・データ更新・通知|
|Microsoft Lists|業務データ保存|
|AI Builder|AI解析・判定・要約・提案|
|Outlook|メール受信|
|Teams|通知|
|SharePoint Online|添付ファイル保存|

各コンポーネントは責務を越えた処理を実施しない。

---

## 2.3 フロー分類

|分類|概要|
|------|-------------------------|
|イベント駆動フロー|Outlookメール受信等を契機として自動起動するフロー|
|画面起動フロー|Power Appsから呼び出されるフロー|
|定期実行フロー|スケジュール実行されるフロー|
|Child Flow|共通処理として他フローから呼び出されるフロー|

---

## 2.4 フロー分割方針

- フローは業務単位で分割する。
- 1フローで複数業務を実施しない。
- 共通利用する処理は Child Flow として分離する。
- フロー間は疎結合とし、必要最小限のパラメータのみ受け渡す。

---

## 2.5 冪等性

- 同一要求が複数回実行された場合でも、業務データの整合性を維持する。
- 重複登録が発生する可能性がある処理では、既存データを確認した上で更新可否を判断する。
- 再実行時も業務影響が発生しないよう設計する。

---

## 2.6 フェイルセーフ

- AI処理失敗によって受付登録・案件登録などの業務処理を停止させない。
- 一時的な通信障害は Retry Policy により自動再試行する。
- 業務継続が困難な場合のみフローを終了し、SystemErrorへ登録する。

---

## 2.7 保守性

- 全フローで共通命名規則を適用する。
- Scope を利用して処理を論理的に区分する。
- Configure run after により例外処理を統一する。
- 共通処理は Child Flow として実装し、重複実装を避ける。

---

## 2.8 Scope標準構成

全フローは以下の Scope 構成を標準とする。

|Scope|責務|
|------|------|
|Initialize|環境変数取得、共通変数初期化、CorrelationId・FlowRunId取得|
|Validate|入力チェック、必須チェック、重複チェック、事前条件確認|
|Business Process|業務ロジック実行、Microsoft Lists更新、AI Builder実行、SharePoint保存|
|Notification|Teams通知などの通知処理|
|Complete|正常終了処理、正常ログ出力|
|Catch|ExceptionJson生成、SystemError登録、エラー応答生成|
|Finally|終了ログ出力、共通後処理、リソース解放|

### 利用ルール

- Scope は上記の順序で配置する。
- Catch は Configure run after により前段 Scope の失敗・タイムアウト時に実行する。
- Finally は正常終了・異常終了を問わず必ず実行する。
- 各フローは本構成を基本とし、不要な Scope は省略可能とする。

---

## 2.9 フローライフサイクル

1. Trigger
2. Initialize
3. Validate
4. Business Process
5. Notification
6. Logging
7. Complete

正常終了・異常終了に関わらず Finally を実行する。

---

## 2.10 設計原則

- フローは単一責務とする。
- 業務ロジックを Power Apps に実装しない。
- Microsoft Lists を直接操作する利用者画面は作成しない。
- CorrelationId により業務全体を追跡可能とする。
- FlowRunId によりフロー実行履歴を追跡可能とする。
- エラー情報は SystemError を唯一の正（Single Source of Truth）として管理する。
- 一時エラーは自動再試行し、業務継続を優先する。
- Microsoft標準機能を優先し、不必要なカスタム実装は行わない。

---

## 2.11 設計レビュー

|観点|結果|
|------|------|
|要件定義整合性|適合|
|アーキテクチャ整合性|適合|
|システム構成整合性|適合|
|データ設計整合性|適合|
|Power Apps整合性|適合|
|Power Automateベストプラクティス|適合|
|保守性|適合|
|性能（20名規模）|適合|
|セキュリティ|適合|
|将来拡張性|適合|



<!-- ========================================= -->
<!-- 02_フロー一覧_改訂版.md -->
<!-- ========================================= -->

# 02_フロー一覧

# 3. フロー一覧

## 3.1 目的

本章では、AI運用保守プラットフォーム Phase1 において実装する Power Automate フローの一覧および責務を定義する。

各フローは単一責務を原則とし、画面制御は Power Apps、業務処理は Power Automate が担当する。

---

## 3.2 フロー一覧

|フローID|フロー名|分類|トリガー|概要|更新対象|呼出元|呼出先|
|---------|---------|------|---------|---------|----------------|----------------|----------------|
|FLOW-001|メール受付フロー|イベント駆動|Outlook メール受信|受付情報登録・添付保存・AI解析開始|IntakeInfo、SharePoint|Outlook|FLOW-002|
|FLOW-002|AI解析フロー|Child Flow|FLOW-001|AI要約・分類・対象システム判定・AI結果登録|IntakeInfo、IntakeTargetSystem|FLOW-001、FLOW-006|なし|
|FLOW-003|案件化フロー|画面起動|Power Apps|案件情報登録・担当登録・関連付け・通知|CaseInfo、CaseAssignee、IntakeCaseRelation|Power Apps|FLOW-005|
|FLOW-004|SLA管理フロー|定期実行|スケジュール|SLA期限監視・期限超過判定・通知|CaseInfo|スケジュール|FLOW-005|
|FLOW-005|Teams通知フロー|Child Flow|各フロー|Teams通知送信|なし|FLOW-003、FLOW-004|なし|
|FLOW-006|AI再解析フロー|画面起動|Power Apps|AI再解析実行|IntakeInfo、IntakeTargetSystem|Power Apps|FLOW-002|
|FLOW-007|システムエラー登録フロー|Child Flow|各フロー|SystemError登録|SystemError|全フロー|なし|
|FLOW-008|共通処理フロー|Child Flow|各フロー|共通処理（JSON生成・共通ログ等）|なし|全フロー|なし|

---

## 3.3 フロー関連図

```text
              Outlook
                  │
                  ▼
      FLOW-001 メール受付
                  │
                  ▼
        FLOW-002 AI解析
                  │
                  ▼
           Microsoft Lists
                  │
                  ▼
            Power Apps
                  │
      ┌───────────┴───────────┐
      ▼                       ▼
FLOW-003 案件化         FLOW-006 AI再解析
      │                       │
      ▼                       ▼
FLOW-005 Teams通知     FLOW-002 AI解析

Scheduler
    │
    ▼
FLOW-004 SLA管理
    │
    ▼
FLOW-005 Teams通知

全フロー共通
    ├────────────►FLOW-007 SystemError登録
    └────────────►FLOW-008 共通処理
```

---

## 3.4 フロー責務

### FLOW-001 メール受付フロー
- Outlookメール受信
- 受付情報登録
- 添付ファイル保存
- AI解析開始

### FLOW-002 AI解析フロー
- AI要約
- AI ToDo生成
- AI受付種別判定
- AI対象システム判定
- AI信頼度登録

### FLOW-003 案件化フロー
- 案件登録
- 担当登録
- Intakeとの関連付け
- Teams通知

### FLOW-004 SLA管理フロー
- SLA期限算出
- SLA超過判定
- 通知

### FLOW-005 Teams通知フロー
- Teams通知送信

### FLOW-006 AI再解析フロー
- AI再解析開始
- AI解析フロー呼出

### FLOW-007 システムエラー登録フロー
- SystemError登録
- CorrelationId保持
- FlowRunId保持

### FLOW-008 共通処理フロー
- 共通JSON生成
- 共通ログ生成
- 共通変数初期化
- 共通日時取得

---

## 3.5 Child Flow インターフェース仕様

本システムで利用する Child Flow の入出力仕様を以下に定義する。

### FLOW-005 Teams通知フロー

#### Input

|項目|型|必須|説明|
|------|------|------|------|
|CorrelationId|String|○|業務追跡ID|
|NotificationType|String|○|通知種別|
|TeamsChannelId|String|○|通知先チャネルID|
|Message|String|○|通知本文|

#### Output

|項目|型|説明|
|------|------|------|
|Success|Boolean|通知成否|
|ErrorMessage|String|エラー内容|

---

### FLOW-007 システムエラー登録フロー

#### Input

|項目|型|必須|説明|
|------|------|------|------|
|CorrelationId|String|○|業務追跡ID|
|FlowRunId|String|○|Flow実行ID|
|ErrorGroupId|String|○|エラーグループID|
|ExceptionJson|Object|○|例外情報|

#### Output

|項目|型|説明|
|------|------|------|
|SystemErrorId|String|登録ID|
|Success|Boolean|登録成否|

---

### FLOW-008 共通処理フロー

#### Input

|項目|型|必須|説明|
|------|------|------|------|
|CorrelationId|String|○|業務追跡ID|
|RequestJson|Object|○|共通処理対象データ|

#### Output

|項目|型|説明|
|------|------|------|
|ResultJson|Object|処理結果|
|Success|Boolean|処理成否|

---

## 3.6 設計レビュー

|観点|結果|
|------|------|
|要件定義整合性|適合|
|アーキテクチャ整合性|適合|
|システム構成整合性|適合|
|データ設計整合性|適合|
|画面設計整合性|適合|
|Power Automateベストプラクティス|適合|
|保守性|適合|
|性能（20名規模）|適合|
|セキュリティ|適合|
|将来拡張性|適合|



<!-- ========================================= -->
<!-- 03_フロー命名規則.md -->
<!-- ========================================= -->

# 03_フロー命名規則

# 4. フロー命名規則

## 4.1 目的

本章では、Power Automate フローで使用する各種名称の命名規則を定義する。

すべてのフローで共通の命名規則を適用し、可読性・保守性・検索性を向上させる。

---

## 4.2 基本方針

命名は以下の方針とする。

- 日本語で統一する。
- 役割が分かる名称とする。
- 省略形は一般的なもの以外使用しない。
- 同一用途には同一名称を使用する。
- フロー間で命名規則を統一する。

---

## 4.3 フロー名

### 形式

【分類】処理名

### 例

|種類|名称|
|------|----------------------|
|イベント駆動|【イベント】メール受付|
|画面起動|【画面】案件化|
|定期実行|【定期】SLA監視|
|Child Flow|【共通】Teams通知|

---

## 4.4 Scope名

|Scope名|用途|
|---------|----------------|
|Initialize|初期化|
|Validate|入力チェック|
|Business Process|業務処理|
|Notification|通知処理|
|Complete|正常終了|
|Catch|例外処理|
|Finally|終了処理|

---

## 4.5 Action名

### 形式

【処理】対象

|Action|用途|
|---------|----------------|
|受付情報取得|Microsoft Lists取得|
|案件登録|案件登録|
|Teams通知送信|Teams通知|
|メール受信|Outlook取得|
|AI解析実行|AI Builder|

---

## 4.6 Compose名

|名称|用途|
|------|----------------|
|Compose_Request|リクエストJSON|
|Compose_Response|レスポンスJSON|
|Compose_Log|ログ生成|
|Compose_Error|エラー内容|

---

## 4.7 変数名

### 形式

var名称

|変数名|内容|
|---------|----------------|
|varCorrelationId|CorrelationId|
|varFlowRunId|FlowRunId|
|varCaseId|案件ID|
|varIntakeId|受付ID|
|varRetryCount|再試行回数|

---

## 4.8 配列変数

### 形式

arr名称

例

- arrAttachments
- arrTargetSystems
- arrTeamsUsers

---

## 4.9 オブジェクト変数

### 形式

obj名称

例

- objMail
- objAIResult
- objCase

---

## 4.10 ブール変数

### 形式

flg名称

例

- flgSuccess
- flgRetry
- flgNotify

---

## 4.11 Condition名

|名称|内容|
|------|----------------|
|AI解析成功？|AI結果判定|
|案件化対象？|案件化判定|
|通知必要？|通知判定|

---

## 4.12 Switch名

例

- Switch_受付種別
- Switch_案件状態
- Switch_通知区分

---

## 4.13 Apply to each名

### 形式

Loop_対象

例

- Loop_添付ファイル
- Loop_対象システム
- Loop_担当者

---

## 4.14 Do until名

### 形式

Until_条件

例

- Until_AI完了
- Until_通知完了

---

## 4.15 環境変数

### 形式

ENV_名称

例

- ENV_SharePointUrl
- ENV_TeamsChannelId
- ENV_AIModel
- ENV_TimeZone

---

## 4.16 接続名

- Outlook
- Microsoft Lists
- SharePoint
- Teams
- AI Builder

---

## 4.17 共通識別子

|名称|用途|
|------|----------------|
|CorrelationId|業務処理追跡|
|FlowRunId|Flow実行追跡|
|PromptVersion|AIプロンプト版管理|

---

## 4.18 命名禁止事項

- Action1
- Compose2
- Test
- New Flow
- Variable1
- 無意味な連番
- 日本語・英語混在（同一カテゴリ内）

---

## 4.19 設計レビュー

|観点|結果|
|------|------|
|要件定義整合性|適合|
|アーキテクチャ整合性|適合|
|Power Automateベストプラクティス|適合|
|保守性|適合|
|可読性|適合|
|将来拡張性|適合|



<!-- ========================================= -->
<!-- 04_接続先一覧.md -->
<!-- ========================================= -->

# 04_接続先一覧

# 5. 接続先一覧

## 5.1 目的

本章では、Power Automate が接続する外部サービスおよびコネクタの利用方針を定義する。

各サービスの責務を明確化し、Power Apps・Power Automate・Microsoft Lists・AI Builder・Outlook・Teams・SharePoint Online の責務を分離する。

---

## 5.2 基本方針

Power Automate は Microsoft 365 標準コネクタを利用する。

接続先は必要最小限とし、責務を越えた利用は行わない。

認証は Power Platform 接続（Connection）を利用し、接続アカウントは運用管理者が管理する。

---

## 5.3 接続先一覧

|接続先|コネクタ|認証方式|利用目的|更新可否|
|-------|---------|---------|------------------------------|----------|
|Outlook|Office 365 Outlook|Microsoft 365 接続|メール受信|参照のみ|
|Microsoft Lists|SharePoint|Microsoft 365 接続|業務データ登録・更新・取得|○|
|SharePoint Online|SharePoint|Microsoft 365 接続|添付ファイル保存・取得|○|
|Microsoft Teams|Microsoft Teams|Microsoft 365 接続|Teams通知|送信のみ|
|AI Builder|AI Builder|Power Platform 接続|AI要約・分類・判定・提案|参照のみ|

---

## 5.4 Outlook

### 責務

- メール受信
- メール情報取得
- 添付ファイル取得

### 利用ルール

- メール送信は行わない。
- Outlookを業務データ保存先として利用しない。
- Outlookデータは更新しない。

---

## 5.5 Microsoft Lists

### 責務

- 業務データ保存
- データ取得
- データ更新

### 利用ルール

- Phase1の唯一の業務データストアとする。
- Power AppsはPower Automate経由で更新する。
- データ整合性はPower Automateが保証する。

---

## 5.6 SharePoint Online

### 責務

- 添付ファイル保存
- 添付ファイル取得

### 利用ルール

- 添付ファイルのみ保存する。
- 保存先ライブラリは環境変数で管理する。

---

## 5.7 Microsoft Teams

### 責務

- Teams通知

### 利用ルール

- 通知のみ利用する。
- 業務データは保存しない。
- 通知処理はChild Flowへ集約する。

---

## 5.8 AI Builder

### 責務

- AI要約
- ToDo生成
- 種別判定
- 対象システム判定
- 優先度判定
- AI信頼度算出

### 利用ルール

- 業務データを直接更新しない。
- AI結果はPower AutomateがMicrosoft Listsへ反映する。
- AI失敗時も業務処理は継続する。

---

## 5.9 接続アカウント

|項目|内容|
|------|------|
|接続方式|Microsoft 365 Connection|
|接続アカウント|運用管理用サービスアカウント|
|認証方式|Microsoft Entra ID|
|管理者|運用管理者|

### 運用ルール

- 個人アカウントを利用しない。
- 接続情報はPower Platformで管理する。
- 接続変更時は影響確認を実施する。

---

## 5.10 環境変数

|環境変数|用途|
|----------|----------------|
|ENV_SharePointSiteUrl|SharePointサイト|
|ENV_DocumentLibrary|添付保存先|
|ENV_TeamsChannelId|通知先チャネル|
|ENV_AIModel|AIモデル|
|ENV_TimeZone|タイムゾーン|

---

## 5.11 設計レビュー

|観点|結果|
|------|------|
|要件定義整合性|適合|
|アーキテクチャ整合性|適合|
|システム構成整合性|適合|
|Power Platformベストプラクティス|適合|
|保守性|適合|
|セキュリティ|適合|
|将来拡張性|適合|



<!-- ========================================= -->
<!-- 05_共通設計.md -->
<!-- ========================================= -->

# 05_共通設計

# 6. 共通設計

## 6.1 目的

本章では、Power Automate 全体で共通利用する設計方針を定義する。

すべてのフローは本章の設計に従い、共通変数・識別子・環境変数・日時・JSON・ログ・AI共通情報を統一する。

---

## 6.2 基本方針

- フロー間で共通仕様を統一する。
- 共通情報は再利用可能な形式で保持する。
- CorrelationId により業務全体を追跡可能とする。
- FlowRunId により Flow 実行履歴を追跡する。
- 日時は UTC 保存・JST 表示とする。
- JSON形式を統一する。
- 環境依存値は環境変数で管理する。
- 定数はフロー内へハードコーディングしない。

---

## 6.3 共通変数

|変数名|型|用途|
|------|------|----------------|
|varCorrelationId|String|業務追跡ID|
|varFlowRunId|String|Flow実行ID|
|varCurrentDateTime|String|現在日時(UTC)|
|varCurrentUser|String|実行ユーザー|
|varRetryCount|Integer|再試行回数|
|varErrorMessage|String|エラーメッセージ|

---

## 6.4 CorrelationId

### 目的

Power Apps・Power Automate・SystemError を一意に関連付ける識別子として利用する。

### 利用ルール

- 業務開始時に生成する。
- 業務終了まで変更しない。
- 全フローへ引き継ぐ。
- SystemErrorへ保存する。
- ログへ保存する。

---

## 6.5 FlowRunId

### 目的

Power Automate の実行履歴を追跡する。

### 利用ルール

- Flow実行開始時に取得する。
- CorrelationIdとセットで保持する。
- SystemErrorへ保存する。
- 実行ログへ保存する。

---

## 6.6 日付時刻

|項目|方式|
|------|------|
|保存|UTC|
|内部処理|UTC|
|Power Apps表示|JST|
|Teams通知|JST|
|メール通知|JST|

### 利用ルール

- Power Automate内部ではUTCを利用する。
- 表示時のみJSTへ変換する。
- convertTimeZone() または Convert time zone アクションを利用する。

---

## 6.7 JSON設計

JSON構造は共通化する。

|名称|用途|
|------|----------------|
|Compose_Request|要求JSON|
|Compose_Response|応答JSON|
|Compose_Log|ログ生成|
|Compose_Error|例外情報|

JSONキーは camelCase を採用する。

---

## 6.8 環境変数

|環境変数|用途|
|---------|----------------|
|ENV_SharePointSiteUrl|SharePointサイト|
|ENV_DocumentLibrary|添付保存先|
|ENV_AIModel|AIモデル|
|ENV_TeamsChannelId|通知先|
|ENV_TimeZone|タイムゾーン|

---

## 6.9 定数

|名称|内容|
|------|----------------|
|STATUS_NEW|新規|
|STATUS_PROGRESS|対応中|
|STATUS_COMPLETE|完了|

ハードコーディングは禁止する。

---

## 6.10 AIResultJson

AI Builder応答全文を保存する。

### 利用ルール

- 一覧画面では取得しない。
- 詳細画面・保守画面のみ取得する。
- 一般利用者へ公開しない。

---

## 6.11 ExceptionJson

例外情報をJSON形式で保持する。

### 利用ルール

- SystemErrorへ保存する。
- 保守担当のみ参照可能とする。

---

## 6.12 AIConfidence

|項目|内容|
|------|------|
|型|Decimal|
|精度|5,2|
|範囲|0.00～100.00|
|表示|小数点2桁|

AI判定値とセットで保持する。

---

## 6.13 ログ共通項目

|項目|用途|
|------|----------------|
|CorrelationId|業務追跡|
|FlowRunId|Flow追跡|
|FlowName|フロー識別|
|開始日時|開始時刻|
|終了日時|終了時刻|
|処理時間|性能分析|
|実行結果|Success / Failed|

---

## 6.14 共通設計ルール

- 共通情報は全フローで統一する。
- FlowRunIdは全フローで保持する。
- CorrelationIdは業務終了まで変更しない。
- AI情報はPower Automateが管理する。
- Power Appsで共通情報を生成しない。
- JSON構造をフロー毎に変更しない。
- 環境依存値は環境変数から取得する。

---

## 6.15 設計レビュー

|観点|結果|
|------|------|
|要件定義整合性|適合|
|アーキテクチャ整合性|適合|
|システム構成整合性|適合|
|データ設計整合性|適合|
|レビュー指摘事項反映|適合|
|Power Automateベストプラクティス|適合|
|保守性|適合|
|性能|適合|
|セキュリティ|適合|



<!-- ========================================= -->
<!-- 06_共通設計_Ver1.0.md -->
<!-- ========================================= -->

# 06_00_Power Automate基本設計_第6章_共通設計_Ver1.0

# 6. 共通設計

## 6.1 基本方針

本章では、Power Automate
の全フロー（06_01～06_08）で共通利用する変数、識別子、JSON構造、AI共通項目、環境変数および定数を定義する。

個別フロー固有の設定は各詳細設計書で定義し、本章では全フロー共通仕様のみを定義する。

### 設計方針

-   共通項目は全フローで統一する。
-   CorrelationId により一連の業務を追跡可能とする。
-   FlowRunId により個々のフロー実行を追跡可能とする。
-   ErrorGroupId により関連するエラーをグルーピングする。
-   日時は UTC で保存し、Power Apps 表示時に JST へ変換する。
-   AI共通項目は全AI処理で統一する。
-   JSONは可読性・保守性を考慮した共通構造を採用する。
-   環境依存値は環境変数として管理し、フローへ直接記述しない。

# 6.2 共通変数

  変数名             型        用途
  ------------------ --------- -------------------------
  varCorrelationId   String    業務処理全体識別子
  varFlowRunId       String    フロー実行ID
  varErrorGroupId    String    関連エラー識別子
  varCurrentUtc      String    現在UTC日時
  varCurrentJst      String    現在JST日時（表示用途）
  varFlowName        String    フロー名
  varResult          Object    処理結果
  varError           Object    例外情報
  varRetryCount      Integer   再試行回数

# 6.3 CorrelationId

-   業務開始時に生成し全フローで引き継ぐ。
-   Child Flow 呼び出し時も同一値を使用する。
-   SystemError を含む全関連データの検索キーとする。

# 6.4 FlowRunId

-   フロー開始時に取得する。
-   フロー実行履歴との照合に利用する。
-   CorrelationId と組み合わせて障害解析を行う。

# 6.5 ErrorGroupId

-   同一障害に起因する複数エラーをグルーピングする。
-   初回例外発生時に採番し、関連エラーで共有する。

# 6.6 タイムゾーン設計

  用途                  タイムゾーン
  --------------------- --------------
  Microsoft Lists保存   UTC
  SystemError           UTC
  監査ログ              UTC
  Power Apps表示        JST

# 6.7 JSON設計

-   camelCase を採用する。
-   Null を許容する。
-   配列は空配列を基本とする。
-   プロパティ名を変更しない。

# 6.8 環境変数

  環境変数              用途
  --------------------- ------------------
  ENV_SharePointSite    SharePointサイト
  ENV_DocumentLibrary   添付保存先
  ENV_TeamsChannelId    通知先
  ENV_AIModel           AIモデル
  ENV_TimeZone          既定タイムゾーン
  ENV_SystemMail        送信元メール
  ENV_AdminMail         管理者通知先

# 6.9 共通定数

  定数             内容
  ---------------- ---------------------
  TIMEZONE_UTC     UTC
  TIMEZONE_JST     Tokyo Standard Time
  STATUS_SUCCESS   正常終了
  STATUS_ERROR     異常終了
  STATUS_WARNING   警告
  TRUE             true
  FALSE            false

# 6.10 AIConfidence仕様

-   データ型：Decimal（5,2）
-   値範囲：0.00～100.00
-   AIResultJson とセットで保持する。
-   一覧画面では取得せず、詳細画面のみ取得する。

# 6.11 AIModel

AI解析に使用したモデル名を保持し、将来の比較分析に利用する。

# 6.12 PromptVersion

利用したプロンプトバージョンを保持する。

# 6.13 ProcessingTime

AI処理時間（ms）を保持し、性能分析・ボトルネック分析に利用する。

# 6.14 AIResultJson

AI応答全体をJSON形式で保持する。

-   一覧画面では取得しない。
-   詳細画面のみ取得する。
-   人が直接編集しない。
-   AI再解析時も過去データを保持する。

# 6.15 ExceptionJson

Power Automateの詳細例外情報をJSON形式で保持する。

保持対象例

-   エラーコード
-   エラーメッセージ
-   発生Scope
-   発生Action
-   HTTP Status
-   Inner Exception

保守用途のみ利用し、一般利用者には表示しない。

# 6.16 設計レビュー結果

  観点                               結果
  ---------------------------------- ------
  要件定義整合性                     適合
  アーキテクチャ整合性               適合
  システム構成整合性                 適合
  データ設計整合性                   適合
  Power Apps整合性                   適合
  Power Automateベストプラクティス   適合
  保守性                             適合
  性能（20名規模）                   適合
  セキュリティ                       適合
  将来拡張性                         適合



<!-- ========================================= -->
<!-- 07_エラーハンドリング_Ver1.0.md -->
<!-- ========================================= -->

# 06_00_Power Automate基本設計_第7章_エラーハンドリング_Ver1.0

# 7. エラーハンドリング

## 7.1 基本方針

Power Automate のエラーハンドリングは、業務継続を最優先とする。

一時的な障害は自動回復を試み、回復不能な障害のみ SystemError
リストへ登録する。

利用者へは必要最小限のメッセージのみ表示し、詳細な例外情報は保守用途として管理する。

### 設計方針

-   Scope を標準採用する。
-   Configure run after を利用して例外処理を統一する。
-   Terminate により終了状態を明示する。
-   SystemError をエラー情報の Single Source of Truth とする。
-   AI処理失敗でも受付登録・案件登録は継続する。
-   CorrelationId・FlowRunId・ErrorGroupId により障害追跡を可能とする。
-   利用者向けメッセージと保守用例外情報を分離する。

## 7.2 Scope標準構成

``` text
Initialize
    ↓
Validate
    ↓
Business Process
    ↓
Notification
    ↓
Complete
    ↓
Catch
    ↓
Finally
```

  Scope              責務
  ------------------ --------------------
  Initialize         変数初期化
  Validate           入力チェック
  Business Process   業務処理
  Notification       通知処理
  Complete           正常終了処理
  Catch              例外処理
  Finally            終了処理・ログ出力

## 7.3 Configure run after

Catch Scope は Failed、Timed Out、Skipped を対象とする。

Finally Scope は Succeeded、Failed、Timed Out、Skipped
のすべてを対象とし、終了処理を必ず実行する。

## 7.4 Terminate設計

  状態        用途
  ----------- --------------------
  Succeeded   正常終了
  Failed      業務継続不能
  Cancelled   利用者キャンセル等

Terminate は Catch または Complete のみで使用する。

## 7.5 例外処理

### 一時障害

-   通信エラー
-   タイムアウト
-   一時的なAPIエラー

対応：Retry Policy により自動回復する。

### 業務エラー

-   必須データ不足
-   案件不存在
-   マスタ未登録

対応：SystemError 登録後、必要に応じてフロー終了する。

### AI処理エラー

-   AI Builder タイムアウト
-   AI応答取得失敗

対応：

-   AI項目のみ未更新
-   受付登録・案件登録は継続
-   SystemErrorへ登録

### 通知エラー

Teams通知失敗時も業務処理は継続し、SystemErrorへ登録する。

## 7.6 Retry連携

Retry Policy により回復不能となった場合のみ Catch Scope を実行する。

Catch では以下を実施する。

-   ErrorGroupId生成
-   ExceptionJson生成
-   SystemError登録
-   利用者向けメッセージ生成

## 7.7 SystemError登録

重大エラーは Child Flow（FLOW-007）で登録する。

  項目            内容
  --------------- ----------------------
  CorrelationId   業務追跡
  FlowRunId       実行履歴
  ErrorGroupId    関連エラー
  FlowName        フロー名
  ScopeName       Scope名
  ActionName      Action名
  ErrorCode       エラーコード
  ErrorMessage    利用者向けメッセージ
  ExceptionJson   詳細例外
  OccurredAt      発生日時（UTC）

## 7.8 SLA計算責務

-   SLA期限計算は Power Automate が担当する。
-   Power Apps は表示のみ担当する。
-   更新失敗時は既存値を保持し、SystemErrorへ登録する。

## 7.9 利用者向けメッセージ

  事象           表示メッセージ
  -------------- --------------------------------------------------
  受付登録失敗   受付登録に失敗しました。
  案件作成失敗   案件作成に失敗しました。
  AI解析失敗     AI解析に失敗しました。受付処理は継続しています。
  通信エラー     通信エラーが発生しました。再度実行してください。
  権限不足       操作権限がありません。

## 7.10 設計レビュー結果

  観点                               結果
  ---------------------------------- ------
  要件定義整合性                     適合
  アーキテクチャ整合性               適合
  システム構成整合性                 適合
  データ設計整合性                   適合
  Power Apps整合性                   適合
  Power Automateベストプラクティス   適合
  保守性                             適合
  性能（20名規模）                   適合
  セキュリティ                       適合
  将来拡張性                         適合



<!-- ========================================= -->
<!-- 08_再試行設計_Ver1.0.md -->
<!-- ========================================= -->

# 06_00_Power Automate基本設計_第8章_再試行設計_Ver1.0

# 8. 再試行設計

## 8.1 基本方針

Power Automate
の再試行（Retry）は、一時的な障害から自動復旧することを目的として利用する。

Microsoft Learn のベストプラクティスに従い、コネクタ標準の Retry Policy
を優先し、不要な独自ループや Do until による再試行は実装しない。

### 設計方針

-   Retry Policy を標準採用する。
-   一時的障害のみ再試行対象とする。
-   業務エラーは再試行しない。
-   Retry 回数超過後は Catch Scope を実行する。
-   Retry の成否に関わらず CorrelationId を維持する。
-   Retry に失敗した場合は SystemError を登録する。

## 8.2 Retry Policy

  設定項目         方針
  ---------------- --------------------------------------
  Retry方式        Exponential Interval
  初期待機時間     Power Automate標準設定
  最大再試行回数   コネクタ標準設定を基本とする
  再試行対象       通信エラー・タイムアウト等の一時障害

## 8.3 Timeout設計

Timeout発生時は以下の順で処理する。

``` text
Timeout
 ↓
Retry Policy
 ↓
Catch Scope
 ↓
SystemError登録
 ↓
Finally
```

## 8.4 Outlook

対象：メール受信、添付ファイル取得

再試行対象：Exchange Online一時障害、通信タイムアウト

失敗時：SystemError登録

## 8.5 Microsoft Lists

対象：作成・更新・取得

再試行対象：

-   SharePoint一時障害
-   スロットリング
-   通信エラー

再試行対象外：

-   必須項目不足
-   Lookupエラー
-   業務整合性エラー

## 8.6 SharePoint Online

対象：添付ファイル保存・取得

再試行失敗時は SystemError に登録する。

## 8.7 AI Builder

対象：

-   AI要約
-   AI分類
-   AI判定
-   AI ToDo生成

再試行失敗時：

-   AI項目のみ未更新
-   業務処理は継続
-   SystemError登録

## 8.8 Teams

Teams通知失敗時も業務処理は継続し、SystemErrorへ登録する。

## 8.9 Retry対象外

  事象             理由
  ---------------- ------------------------
  入力値不正       再実行しても改善しない
  権限不足         設定変更が必要
  マスタ未登録     業務データ不備
  Lookup不整合     業務エラー
  AI判定結果不正   人による確認が必要
  重複登録エラー   冪等性で制御する

## 8.10 Retryとログ

-   CorrelationIdは変更しない。
-   FlowRunIdは変更しない。
-   Retry回数は varRetryCount に保持できるものとする。

## 8.11 業務ルール

-   Retryは一時障害のみ対象とする。
-   業務エラーではRetryしない。
-   AI処理・通知処理失敗でも業務処理は継続する。
-   標準Retry Policyを採用する。

## 8.12 設計レビュー結果

  観点                               結果
  ---------------------------------- ------
  要件定義整合性                     適合
  アーキテクチャ整合性               適合
  システム構成整合性                 適合
  データ設計整合性                   適合
  Power Apps整合性                   適合
  Power Automateベストプラクティス   適合
  保守性                             適合
  性能（20名規模）                   適合
  セキュリティ                       適合
  将来拡張性                         適合



<!-- ========================================= -->
<!-- 09_ログ設計_Ver1.0.md -->
<!-- ========================================= -->

# 06_00_Power Automate基本設計_第9章_ログ設計_Ver1.0

# 9. ログ設計

## 9.1 基本方針

Power Automate のログは、障害解析・監査・運用保守を目的として記録する。

ログは CorrelationId を中心に管理し、FlowRunId および ErrorGroupId
と組み合わせて業務全体を追跡可能とする。

### 設計方針

-   CorrelationId を全フローで共通利用する。
-   FlowRunId によりフロー実行単位を識別する。
-   ErrorGroupId により関連エラーをグルーピングする。
-   SystemError をエラー情報の Single Source of Truth とする。
-   業務データとログデータを分離する。
-   個人情報・機密情報は必要最小限のみ記録する。

## 9.2 ログ分類

  ログ種別         用途             保存先
  ---------------- ---------------- -------------------------
  実行ログ         フロー実行状況   Power Automate 実行履歴
  システムエラー   障害解析         SystemError
  監査ログ         業務更新履歴     各業務リスト変更履歴

## 9.3 CorrelationId

-   一連の業務処理を識別する共通キー
-   Child Flow へ引き継ぐ
-   障害解析時の検索キーとする

## 9.4 FlowRunId

-   フロー実行単位の識別子
-   Power Automate 実行履歴との照合に利用する

## 9.5 ErrorGroupId

-   同一障害に起因する複数エラーを関連付ける
-   初回エラー発生時に生成する

## 9.6 SystemError

記録項目

  項目            内容
  --------------- ----------------------
  CorrelationId   業務識別
  FlowRunId       実行履歴
  ErrorGroupId    関連エラー
  FlowName        フロー名
  ScopeName       Scope名
  ActionName      Action名
  ErrorCode       エラーコード
  ErrorMessage    利用者向けメッセージ
  ExceptionJson   詳細例外
  OccurredAt      発生日時(UTC)

## 9.7 監査ログ

Microsoft Lists の監査列と変更履歴を利用する。

Power Automate は監査情報を改ざんしない。

## 9.8 ログ出力ルール

-   正常終了時は Power Automate 実行履歴を利用する。
-   異常終了時は SystemError を登録する。
-   AIResultJson・ExceptionJson は一覧画面で取得しない。
-   UTCで保存し、Power Apps表示時にJSTへ変換する。

## 9.9 セキュリティ

-   ExceptionJson は保守担当のみ参照可能とする。
-   AIResultJson は権限制御対象とする。
-   ログへ認証情報・アクセストークンを出力しない。

## 9.10 業務ルール

-   CorrelationId は変更しない。
-   FlowRunId はフロー実行中変更しない。
-   SystemError は唯一の正とする。
-   利用者向けメッセージと詳細例外は分離する。

## 9.11 設計レビュー結果

  観点                               結果
  ---------------------------------- ------
  要件定義整合性                     適合
  アーキテクチャ整合性               適合
  システム構成整合性                 適合
  データ設計整合性                   適合
  Power Apps整合性                   適合
  Power Automateベストプラクティス   適合
  保守性                             適合
  性能（20名規模）                   適合
  セキュリティ                       適合
  将来拡張性                         適合



<!-- ========================================= -->
<!-- 10_セキュリティ_Ver1.0.md -->
<!-- ========================================= -->

# 06_00_Power Automate基本設計_第10章_セキュリティ_Ver1.0

# 10. セキュリティ

## 10.1 基本方針

Power Automate は最小権限の原則に基づき設計する。

Power Apps・Power Automate・Microsoft Lists・SharePoint Online
の責務を分離し、接続アカウントは共通管理する。

## 10.2 接続アカウント

  項目                 方針
  -------------------- --------------------
  接続方式             Microsoft 365 接続
  管理主体             運用管理者
  最小権限             必要最小限
  個人アカウント利用   禁止

## 10.3 コネクタ

利用する標準コネクタ

-   Office 365 Outlook
-   SharePoint（Microsoft Lists含む）
-   Microsoft Teams
-   AI Builder

カスタムコネクタは Phase1 では採用しない。

## 10.4 権限制御

-   Power Apps が画面権限を制御する。
-   Power Automate は業務処理のみ担当する。
-   Microsoft Lists は直接利用させない。

## 10.5 DLP

-   Power Platform DLP ポリシーを適用する。
-   承認済みコネクタのみ利用する。
-   外部サービスへのデータ送信は禁止する。

## 10.6 AIResultJson

-   詳細画面のみ取得可能
-   一覧画面では取得しない
-   保守担当者・システム開発者のみ閲覧可能

## 10.7 ExceptionJson

-   保守用途のみ利用する。
-   一般利用者へ表示しない。
-   SystemError からのみ参照する。

## 10.8 FlowRunId

-   Power Automate 実行履歴との照合に利用する。
-   一般利用者へ表示しない。

## 10.9 機密情報

以下はログ・通知へ出力しない。

-   パスワード
-   アクセストークン
-   接続情報
-   シークレット
-   認証ヘッダー

## 10.10 セキュリティ監査

-   CorrelationId による追跡
-   Microsoft 365 標準監査ログを利用
-   SystemError による障害管理

## 10.11 業務ルール

-   最小権限を徹底する。
-   標準コネクタを優先する。
-   DLP ポリシーを遵守する。
-   AIResultJson・ExceptionJson は権限制御対象とする。

## 10.12 設計レビュー結果

  観点                               結果
  ---------------------------------- ------
  要件定義整合性                     適合
  アーキテクチャ整合性               適合
  システム構成整合性                 適合
  データ設計整合性                   適合
  Power Apps整合性                   適合
  Power Automateベストプラクティス   適合
  保守性                             適合
  性能（20名規模）                   適合
  セキュリティ                       適合
  将来拡張性                         適合



<!-- ========================================= -->
<!-- 11_性能設計_Ver1.0.md -->
<!-- ========================================= -->

# 06_00_Power Automate基本設計_第11章_性能設計_Ver1.0

# 11. 性能設計

## 11.1 基本方針

Power Automate は約20名規模の社内利用を前提とし、Microsoft Learn
のベストプラクティスに従ってシンプルかつ保守性の高い構成とする。

## 11.2 Concurrency

-   不要な並列実行は行わない。
-   同時更新による競合を避けるため、業務処理は必要に応じて直列実行とする。
-   並列実行が必要な場合のみ個別フローで設定する。

## 11.3 Pagination

-   一覧取得時は Pagination を利用する。
-   必要最小限のデータのみ取得する。
-   大量データの一括取得は行わない。

## 11.4 Apply to each

-   ネストを最小限とする。
-   不要なループを実装しない。
-   Select や Filter Array を優先して利用する。

## 11.5 API呼び出し

-   API呼び出し回数を最小化する。
-   同一データの重複取得を避ける。
-   必要列のみ取得する。

## 11.6 AI呼び出し

-   必要時のみ AI Builder を実行する。
-   AI解析結果は再利用可能な項目を保存する。
-   不要な再解析は行わない。

## 11.7 Select・Filter Array

-   Apply to each より優先して利用する。
-   データ加工は可能な限り標準アクションで実施する。

## 11.8 業務ルール

-   必要最小限のデータ取得を行う。
-   長時間実行フローを避ける。
-   標準機能を優先する。
-   AI処理は必要最小限とする。

## 11.9 設計レビュー結果

  観点                               結果
  ---------------------------------- ------
  要件定義整合性                     適合
  アーキテクチャ整合性               適合
  システム構成整合性                 適合
  データ設計整合性                   適合
  Power Apps整合性                   適合
  Power Automateベストプラクティス   適合
  保守性                             適合
  性能（20名規模）                   適合
  セキュリティ                       適合
  将来拡張性                         適合



<!-- ========================================= -->
<!-- 12_業務ルール_Ver1.0.md -->
<!-- ========================================= -->

# 06_00_Power Automate基本設計_第12章_業務ルール_Ver1.0

# 12. 業務ルール

## 12.1 基本方針

Power Automate は
AI運用保守プラットフォームの業務処理基盤として動作し、Power Apps
は画面制御のみを担当する。

## 12.2 共通業務ルール

-   業務処理は Power Automate が担当する。
-   Power Apps は画面表示・入力・画面遷移のみ担当する。
-   Microsoft Lists を唯一の業務データストアとする。
-   Outlook はメール受信のみ担当する。
-   SharePoint Online は添付ファイル保存のみ担当する。
-   Teams は通知のみ担当する。
-   AI Builder は AI
    判定・要約・提案のみ担当し、人が確定した値を更新しない。

## 12.3 フロー設計ルール

-   フローは単一責務とする。
-   共通処理は Child Flow 化する。
-   Scope・Configure run after・Terminate を標準構成とする。
-   CorrelationId・FlowRunId・ErrorGroupId を全フローで利用する。

## 12.4 データ更新ルール

-   更新は Power Automate が実施する。
-   Power Apps から Microsoft Lists を直接更新しない。
-   UTC保存・JST表示を標準とする。

## 12.5 AI業務ルール

-   AI結果は参考情報とする。
-   AI処理失敗でも受付登録・案件登録は継続する。
-   AIResultJson を保持する。
-   AIConfidence を保持する。
-   PromptVersion を保持する。

## 12.6 エラー処理ルール

-   SystemError を唯一の正とする。
-   一時障害は Retry Policy を利用する。
-   利用者向けメッセージと詳細例外を分離する。

## 12.7 セキュリティルール

-   最小権限を適用する。
-   DLP ポリシーを遵守する。
-   AIResultJson・ExceptionJson は権限制御対象とする。

## 12.8 性能ルール

-   必要最小限のデータ取得とする。
-   標準コネクタを利用する。
-   不要な AI 呼び出しを行わない。
-   長時間実行フローを避ける。

## 12.9 保守ルール

-   命名規則を統一する。
-   環境変数を利用する。
-   ハードコードを禁止する。
-   共通処理の重複実装を避ける。

## 12.10 設計レビュー結果

  観点                               結果
  ---------------------------------- ------
  要件定義整合性                     適合
  アーキテクチャ整合性               適合
  システム構成整合性                 適合
  データ設計整合性                   適合
  Power Apps整合性                   適合
  Power Automateベストプラクティス   適合
  保守性                             適合
  性能（20名規模）                   適合
  セキュリティ                       適合
  将来拡張性                         適合


