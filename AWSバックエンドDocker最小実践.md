# AWS（エーダブリューエス） バックエンド（Backend） Docker（ドッカー） 最小実践

更新日：2026-10-27  
プロジェクト（Project）：`device-dashboard`  
技術 chain：React（リアクト） → Python（パイソン） / FastAPI（ファストエーピーアイ） → Docker（ドッカー） → AWS（エーダブリューエス）  
費用：学習用アカウント（Account）の Free plan（フリープラン） / credit（クレジット）；resource（リソース） は短時間のみ起動し、終了後に cleanup（クリーンアップ）

---

## 一、目標

手元で動作確認済みの Backend（バックエンド） Container（コンテナ） を AWS（エーダブリューエス） に接続し、既存 Frontend（フロントエンド） と読み取り可能なリクエスト（Request）経路をつなぎ、deploy（デプロイ） → rollback（ロールバック） → cleanup（クリーンアップ） を一通り実施する。

本ラウンド（round）のクラウド上の機能：機器一覧の read-only（リードオンリー）（`GET /devices`）。

完了時に説明できること：リクエスト（Request）が Browser（ブラウザー） からどこへ入るか、Container（コンテナ） がどこで動くか、データがどこに保存されるか、ログ（Log）の確認場所、版の更新と戻し方、練習用 resource（リソース） の片付け方。

---

## 二、全体構成（1 回のリクエスト（Request）の順）

```text
Browser
  → CloudFront + S3          Frontend（既存）
  → ALB                      API 入口
  → ECS（Fargate）Task       FastAPI Container を実行
  → RDS PostgreSQL           機器データを保存
  → 同じ経路で Browser へ戻る
```

| 順 | コンポーネント（Component） | 何か | 本ラウンド（round）の役割 |
|:---:|---|---|---|
| 1 | S3（エススリー） | オブジェクトストレージ（object storage） | React（リアクト） のビルド（Build）成果物を保存する |
| 2 | CloudFront（クラウドフロント） | CDN（シーディーエヌ） / 静的配信 | ページ（Page）を Browser（ブラウザー） へ届ける；既存 Distribution（ディストリビューション） を継続利用 |
| 3 | ALB（エーエルビー） | Application Load Balancer（アプリケーションロードバランサー） | API（エーピーアイ） への HTTP（エイチティーティーピー） リクエスト（Request）を受け、健全な Task（タスク） へ転送する；health check（ヘルスチェック） を行う |
| 4 | ECR（イーシーアール） | container registry（コンテナレジストリ） | version tag（バージョンタグ） 付き Docker（ドッカー） Image（イメージ） を保存する |
| 5 | ECS（イーシーエス）＋Fargate（ファーゲート） | コンテナ（Container）オーケストレーション（orchestration）＋サーバーレス（Serverless）実行 | Cluster（クラスター） / Service（サービス） が Task（タスク） を維持；Task（タスク） が ECR（イーシーアール） から Image（イメージ） を取得し FastAPI（ファストエーピーアイ） を実行 |
| 6 | RDS（アールディーエス） PostgreSQL（ポストグレスキューエル） | マネージド（Managed） PostgreSQL（ポストグレスキューエル） | 最小スペック（spec）・短時間；API（エーピーアイ） は connection string（コネクションストリング） で接続 |
| 7 | IAM（アイエーエム） Role（ロール） | 実行時の主体 | Task Execution Role（タスクエグゼキューションロール）：Image（イメージ） 取得・ログ（Log）書き込み；Task Role（タスクロール）：他の AWS（エーダブリューエス） resource（リソース） へのアクセス（Access）（必要時） |
| 8 | Security Group（セキュリティグループ） | ネットワーク（Network）のセキュリティグループ（Security Group） | ALB（エーエルビー）→ECS（イーシーエス）、ECS（イーシーエス）→RDS（アールディーエス） の接続可否を定める |
| 9 | environment variable（エンバイロメントバリアブル） | 実行時設定 | 起動時に connection string（コネクションストリング） などを Container（コンテナ） へ注入；secret（シークレット） は Image（イメージ） の外に置く |
| 10 | CloudWatch Logs（クラウドウォッチログス） | ログ（Log）サービス（Service） | Container（コンテナ） / API（エーピーアイ） の出力を確認し、障害調査に使う |

一連の関係は次のように覚える：

**Frontend（フロントエンド） は CloudFront（クラウドフロント）；Image（イメージ） は ECR（イーシーアール）；プロセス（Process）は ECS（イーシーエス）；トラフィック（Traffic）は ALB（エーエルビー）；データは RDS（アールディーエス）；権限は IAM（アイエーエム） Role（ロール）；ネットワーク（Network）は Security Group（セキュリティグループ）；設定は environment variable（エンバイロメントバリアブル）；観測は CloudWatch Logs（クラウドウォッチログス）。**

---

## 三、実施順序

0 → 7 の順で進める。各ステップを終えてから次へ進む。

### 0. 開始前の確認

`../AWS第一阶段费用与零支出原则.md` と照合し、本ラウンド（round）で作成する resource（ECR（イーシーアール） / ECS（イーシーエス） / ALB（エーエルビー） / RDS（アールディーエス） など）について、無料範囲・枠の消費・課金項目・想定時間・cleanup（クリーンアップ） 方法をまとめる。Region（リージョン） とアカウント（Account）側が短時間練習に適するかも示す。

**これはドキュメント（document）照会の作業であり、AI に任せればよい**（本ラウンドで何を起動するか／どのくらい練習するかを伝えれば足りる。費用ドキュメントを自分で通読して整理する必要はない）。本人がやるのは：AI の結論を確認し、コンソール（Console）で実際のプラン（plan）と残高を見ること。追加支出が必要な結論なら、続行するかどうかは自分で決める。

### 1. 手元の Image（イメージ）

既存 Backend（バックエンド） で `docker build` し、Image（イメージ） が起動すること、手元から API（エーピーアイ） を呼べることを確認する。必要な environment variable（エンバイロメントバリアブル） を確認する。

### 2. ECR（イーシーアール） へ push（プッシュ）

ECR（イーシーアール） repository（リポジトリ） を作成する。Image（イメージ） に version tag（バージョンタグ） を付けて ECR（イーシーアール） へ push（プッシュ） する。コンソール（Console）でその tag（タグ） が見えることを確認する。

### 3. ECS（イーシーエス） + ALB（エーエルビー） で起動

最小構成の ECS（イーシーエス） Cluster（クラスター）、Task Definition（タスクデフィニション）、Service（サービス）、および ALB（エーエルビー） を作成する。Task（タスク） は ECR（イーシーアール） から Image（イメージ） を取得する。Service（サービス） に IAM（アイエーエム） Role（ロール）、CloudWatch Logs（クラウドウォッチログス）、health check（ヘルスチェック） を設定する。ALB（エーエルビー） の URL（ユーアールエル） で直接 `GET /devices` を呼ぶ（この段階の設定準備に続き、Database（データベース） は次のステップで正式に接続する）。

### 4. RDS（アールディーエス） を接続

短時間・最小スペック（spec）の RDS（アールディーエス） PostgreSQL（ポストグレスキューエル） を作成する。Security Group（セキュリティグループ） で ECS（イーシーエス） → RDS（アールディーエス） を通す。connection string（コネクションストリング） を environment variable（エンバイロメントバリアブル） で Task（タスク） に注入する。API（エーピーアイ） が RDS（アールディーエス） 上のデータを読んでいることを確認する。

### 5. 既存 Frontend（フロントエンド） を接続

Frontend（フロントエンド） の API（エーピーアイ） URL（ユーアールエル） を ALB（エーエルビー） のアドレスに変え、S3（エススリー） / CloudFront（クラウドフロント） へ公開する。Browser（ブラウザー） で CloudFront（クラウドフロント） のページ（Page）を開き、一覧がクラウド上の API（エーピーアイ） 由来であることを確認する。CORS（コルス） と HTTPS（エイチティーティーピーエス） が正常であることを確認する。

### 6. deploy（デプロイ） と rollback（ロールバック）

API（エーピーアイ） response を一箇所変更 → 再 build / 新 tag（タグ） を push（プッシュ） → ECS（イーシーエス） Service（サービス） を更新 → Browser（ブラウザー） で新結果を確認 → 前の tag（タグ） に戻し、rollback（ロールバック） が効くことを確認する。

### 7. cleanup（クリーンアップ）

本ラウンド（round）で作成したものを一覧どおり停止・削除する：ALB（エーエルビー）、ECS（イーシーエス） Service（サービス） / Task（タスク）、RDS（snapshot の確認を含む）、ECR（イーシーアール） 上の本ラウンド（round）の旧 Image（イメージ）、長すぎる log（ログ） retention（リテンション）、残った public（パブリック） IP（アイピー）。既存の S3（エススリー） / CloudFront（クラウドフロント） Frontend（フロントエンド） はそのまま残す。

---

## 四、開始・終了チェック（check）

**開始前**

1. 費用ドキュメント（document）照合の結論は、AI が `../AWS第一阶段费用与零支出原则.md` を調べて書き済み（ドキュメント照会 → AI に任せる）  
2. その結論を確認し、コンソール（Console）で Free plan（フリープラン） / credit（クレジット） / Region（リージョン） / サービス（Service）の利用可否を確認済み  
3. Free plan（フリープラン） を維持；追加支出が必要な場合は、先に判断してから作成  

**終了後**

1. 本ラウンド（round）の ALB（エーエルビー）、ECS（イーシーエス）、RDS（アールディーエス） を停止・削除済み；snapshot（スナップショット） を確認済み  
2. ECR（イーシーアール） の旧 Image（イメージ）、CloudWatch Logs（クラウドウォッチログス）、public（パブリック） IP（アイピー） / ネットワーク（Network）残留を確認済み  
3. 既存 Frontend（フロントエンド）を誤って削除していない  

---

## 五、進捗

- [x] 方針の確認（2026-10-27）
- [x] 0 費用とアカウント（Account）確認（ドキュメント照会は AI に任せる；2026-10-27：費用ドキュメントに基づき短時間練習可と判断）
- [x] 1 手元の Image（2026-10-27）
- [x] 2 ECR（イーシーアール） push（プッシュ）（2026-10-27：リポジトリ（repository）作成済み；tag（タグ） `v1` を push（プッシュ） 済み）
- [x] 3 ECS（イーシーエス） + ALB（エーエルビー）（2026-10-27：`curl ALB/` 成功；疎通後に B どおり課金項目を停止・削除）
- [x] 4 RDS（アールディーエス） PostgreSQL（ポストグレスキューエル）（2026-10-28：C.2～C.7 疎通。截図：`../2026-10-28-验通-经ALB能访问云上Backend容器.png`、`../2026-10-28-验通-云上API从RDS返回设备列表.png`）
- [x] 5 Frontend（フロントエンド） 接続（2026-10-28：手元→ALB（エーエルビー） 疎通；S3（エススリー） sync（シンク） + Invalidation（インバリデーション）；F.3 後 CloudFront（クラウドフロント） 同一オリジン（Origin） HTTPS（エイチティーティーピーエス） `/devices`→ALB（エーエルビー）→RDS（アールディーエス） **200**。截図：`../2026-10-28-验通-本地Frontend经ALB拉取云上设备列表.png`、`../2026-10-28-现象-CloudFront的HTTPS页请求HTTP的ALB被浏览器拦截.png`、`../2026-10-28-验通-CloudFront的HTTPS页经同域拉取RDS设备列表.png`）
- [x] 6 deploy（デプロイ） / rollback（ロールバック）（2026-10-28：`v1` 上書きのみ；Force new deployment（フォースニューデプロイメント） で発版を確認；Subnet（サブネット） は ALB（エーエルビー） の 1a/1c のみ）
- [x] 7 cleanup（2026-10-28：Desired（デザイアード）=0、ALB（エーエルビー）、TG（ティージー）、RDS（アールディーエス） 削除済み；Secrets Manager（シークレッツマネージャー） 一覧は空；S3（エススリー）/CloudFront（クラウドフロント） は未変更。Secrets Manager（シークレッツマネージャー） は別途学習予定）

関連：`../AWS第一阶段费用与零支出原则.md` · `機器管理システムの開発順序と学習手順.md`（第 4 段階）

---

## 六、実操記録（振り返り用）

記録日（きろくび）：2026-10-27 ～ 2026-10-28。実施（じっし）順序（じゅんじょ） 0 → 7 に沿（そ）い、本ラウンド（round）で実際（じっさい）に行（おこな）った操作（そうさ）と結果（けっか）を書（か）く（振（ふ）り返（かえ）り用）。コマンド（Command）はすべて 1 行（`\` による継続（けいぞく）は使（つか）わない）。

### 重要な判断（本ラウンドの構成）

- 手元の `docker compose` は **api** と **db** の 2 コンテナ（Container）を同時に起動し、開発を容易にする。
- 本ラウンド（round）のクラウド側：**api イメージ（Image）のみ ECR（イーシーアール） へ push（プッシュ）**；データベース（Database）はクラウド上で RDS（アールディーエス）を使う。手元の postgres（ポストグレス） コンテナ（Container）イメージ（Image）は push（プッシュ） しない。
- `docker compose build api` はバックエンド（Backend）イメージ（Image）のみ構築；`docker compose up -d` は 2 コンテナ（Container）の起動であり、「バックエンド（Backend）＋データベース（Database）」を 1 つのイメージ（Image）にまとめるわけではない。

### 本ラウンド（round）の AWS（エーダブリューエス） アカウント（Account）と Region（リージョン）

- 練習用アカウント（Account）：**`853900965180`**（コンソールで ECR（イーシーアール） 作成に使ったアカウント）。
- Region（リージョン）：**`ap-northeast-1`（東京）**。
- コンソール（Console） IAM（アイエーエム） 学習ユーザー（User）：**`touzui-learning`**（既存の `AdministratorAccess`）。
- 本機に出ていた誤った CLI（シーエルアイ） 主体：`393886308010` / `CLI_user_test`（`ecr:CreateRepository` なし、かつ練習用アカウント（Account）と別）→ 本ラウンド（round）の ECR（イーシーアール） の create/push（プッシュ） に **使えない**；`853900965180` 配下の `touzui-learning` の access（アクセス） key に切り替える必要がある。

### ステップ 0

- ステップ 0（ドキュメント照会 → AI に任せる）：`../AWS第一阶段费用与零支出原则.md` と照合した結果、本ラウンド（round）の ECR（イーシーアール） / ECS（イーシーエス） / ALB（エーエルビー） / RDS（アールディーエス） 等は短時間・小スペック（spec）・終了後 cleanup（クリーンアップ）；Paid plan（ペイドプラン） へはアップグレードしない。
- 「作成」を押す直前には、コンソール（Console）で Free plan（フリープラン） / credit（クレジット） を自分で再確認する。

### ステップ 1：手元の Image（完了）

**確認したファイル（設定は原則未変更）：**

- 必読：`Dockerfile`、`.dockerignore`、`requirements.txt`、`compose.yaml`、`.env`
- ざっと確認：`main.py`、`alembic.ini`、`migrations/`
- 本ステップでは精読不要：`requirements_for_python+sql.txt`、`.vene/`、`learning/`、`test/`、ノート（note） md、業務詳細ファイル（File）等

**要点：**

- `COPY . .`：左の `.` = ビルド（Build）コンテキスト（context）（本機の `backend`）；右の `.` = イメージ（Image）内のカレント（current）（`WORKDIR /app`）。
- `.dockerignore` は `.vene`、`.env`、`__pycache__` 等を除外；パスワード（Password）は compose / 環境変数で注入し、イメージ（Image）に入れない。
- `compose.yaml`：`api` は `build: .`；`db` は `image: postgres:17`。

**実行したコマンド（Command）（ディレクトリ（directory）：`device-dashboard/backend`）：**

```bash
docker compose build api
docker compose up -d
docker compose ps
docker images | grep -E 'device-dashboard|postgres'
curl http://127.0.0.1:8005/devices
```

**当時の結果要約：**

- `api` / `db` とも Up；`db` は healthy（ヘルシー）。
- イメージ（Image）：`device-dashboard-api:latest`（後で ECR（イーシーアール） へ push（プッシュ））；`postgres:17`（手元のみ）。
- `GET /devices` が機器一覧 JSON（ジェイソン） を返す（HTTP（エイチティーティーピー） 正常）。

### ステップ 2：ECR（完了）

**実施済み：コンソール（Console）で private（プライベート） repository（プライベートリポジトリ） を作成**

- サービス（Service）：Amazon ECR（イーシーアール） → Create（クリエイト） private（プライベート） repository（プライベートリポジトリ）
- Repository（リポジトリ） name：`device-dashboard-api`
- Image（イメージ） tag（タグ） mutability：**Mutable（ミュータブル）**（同一 tag（タグ） を上書き可；Immutable だと同名 tag（タグ） を上書き不可）
- Encryption（エンクリプション）：**AES-256（エーイーエスニゴーロク）**（デフォルト、追加の KMS 費用なし；本ラウンドは AWS（エーダブリューエス） KMS を選ばない）
- 作成後に Images（イメージズ） ページ（Page）へ：当時は **No active images（ノーアクティブイメージズ）**（まだ push（プッシュ） 前で正常）
- リポジトリ（repository） URI（ユーアールアイ） の形：`853900965180.dkr.ecr.ap-northeast-1.amazonaws.com/device-dashboard-api`

**ターミナル（Terminal）での作成は失敗（対照用に残す）：**

```bash
aws ecr create-repository --repository-name device-dashboard-api --region ap-northeast-1
```

- 失敗理由：`AccessDeniedException`；当時の identity（アイデンティティ） は `CLI_user_test@393886308010`、権限なしかつアカウント（Account）不一致。
- 注意：`--repository-name` はハイフン（hyphen） 2 つ；パラメータ（parameter）をターミナル（Terminal）の改行で分断しない。

**現在の CLI（シーエルアイ） 主体の確認：**

```bash
aws sts get-caller-identity
```

**CLI（シーエルアイ） の揃えと push（完了）：**

```bash
aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 853900965180.dkr.ecr.ap-northeast-1.amazonaws.com
docker tag device-dashboard-api:latest 853900965180.dkr.ecr.ap-northeast-1.amazonaws.com/device-dashboard-api:v1
docker push 853900965180.dkr.ecr.ap-northeast-1.amazonaws.com/device-dashboard-api:v1
```

**コマンド（Command）の意味メモ：**

- `get-login-password | docker login ...`：現在の AWS（エーダブリューエス） CLI（シーエルアイ） 主体で ECR（イーシーアール） の一時パスワード（Password）を取り、Docker（ドッカー） をそのアカウント（Account）の東京 ECR（イーシーアール） へログ（Log）インさせる。
- `docker tag ローカル名 ECR完全名:v1`：再 build せず、手元イメージ（Image）に ECR（イーシーアール） 向けの名前と version tag（バージョンタグ） を追加するだけ。
- `docker push ...:v1`：その tag（タグ） を ECR（イーシーアール） リポジトリ（repository）へアップロード（Upload）する。

**push（プッシュ） 結果要約（2026-10-27）：**

- 各レイヤーが `Pushed` と表示
- `v1: digest: sha256:e4c275f4f43dd64e87260d4b4adf445ff7d694b327494e8a2a6f5128002ba0da`
- コンソール（Console）確認：ECR（イーシーアール） → `device-dashboard-api` → Images（イメージズ）、更新後に tag（タグ） **`v1`** が見えること

---

### ステップ 3：ECS（イーシーエス） + ALB（コンソール手順）

**実行結果（2026-10-27）：**

- Cluster（クラスター）：`device-dashboard-cluster`；Service（サービス）：`device-dashboard-api-svc`；Task Definition（タスクデフィニション）：`device-dashboard-api:2`
- ALB（エーエルビー） DNS（ディーエヌエス）：`device-dashboard-alb-2041883229.ap-northeast-1.elb.amazonaws.com`
- 疎通：`curl http://...elb.amazonaws.com/` → `{"message":"backend in container"}`
- 失敗したこと：CloudWatch（クラウドウォッチ） ログ（Log） `exec /usr/local/bin/python: exec format error`（手元 ARM（アーム） イメージ vs Fargate（ファーゲート） x86（エックスハチロク））。修正：`docker build --platform linux/amd64 ...` 後に再 tag（タグ）/push（プッシュ） `v1`、さらに Update service（アップデートサービス） → Force new deployment（フォースニューデプロイメント）。
- 疎通後の費用対応（実施済み）：Desired（デザイアード）→0；ALB（エーエルビー）・TG（ティージー） を削除済み。空 Cluster（クラスター） / ECR（イーシーアール） `v1` / SG（エスジー） は残してよい。

**学習メモ — 「効くまでの流れ」全体を掴む（下位の仕組みは後回し可）：**

本ステップの疎通時、1 リクエスト（Request）が実際に通る関係（まずこの経路を覚える；ENI／スケジューラ／コントロールプレーン等の細部は後で補う）：

```text
手元 curl
  → ALB（パブリック DNS:80、SG: alb-sg が 80 を許可）
  → Listener が HTTP を Target Group へ転送
  → TG がヘルスチェックで「healthy」なターゲット IP:8005 を選ぶ
  → トラフィックが ECS Task 内のコンテナ api（Fargate が動かしているプロセス）へ到達
  → コンテナ内 uvicorn が `/` に応答 → JSON が同じ経路で戻る
```

各部品の役割（本ラウンドで作った名前と対応）：

| 部品 | 本ラウンド（round）の名前 | 流れの中での役割 |
|---|---|---|
| ECR（イーシーアール） Image（イメージ） + tag（タグ） | `.../device-dashboard-api:v1` | コンテナ（Container）内で動くコード（Code）／依存の版 |
| Task Definition（タスクデフィニション） | `device-dashboard-api:2` | 説明書：イメージ（Image）、CPU（シーピーユー）/メモリ、ポート（Port）、環境変数、ログ（Log）、execution role |
| Task（タスク） | Service（サービス） が起動した 1 つ | 説明書の実際の実行インスタンス（Instance） |
| Service（サービス） | `device-dashboard-api-svc` | Desired（デザイアード） 個数を維持；落ちたら再起動；ALB（エーエルビー）/TG（ティージー） にぶら下げる |
| Cluster（クラスター） | `device-dashboard-cluster` | Task（タスク）/Service（サービス） の所属「場所」（Fargate（ファーゲート） ではマシンを自分で管理しない） |
| ALB（エーエルビー） | `device-dashboard-alb` | パブリック（public）入口、80 を受ける |
| Target Group（ターゲットグループ） | `device-dashboard-api-tg` | Task（タスク） IP（アイピー） を登録し、`/` でヘルスチェック（health check）、ALB（エーエルビー） 転送に使う |
| SG（エスジー） | `alb-sg` / `ecs-sg` | alb は外→80 を許可；ecs は alb→8005 のみ許可 |
| CloudWatch Logs（クラウドウォッチログス） | `/ecs/device-dashboard-api` | コンテナ（Container） stdout（スタンダードアウト）/stderr（スタンダードエラー）、障害調査用（例：exec format error） |

リリース時に動かす点（本ステップで一度実施済み）：イメージ（Image）変更 → 同 tag（タグ） または新 tag（タグ） を push（プッシュ） →（必要なら新 Task Definition（タスクデフィニション） revision）→ Service（サービス） 更新 / Force new deployment（フォースニューデプロイメント） → 新 Task（タスク） 起動、旧 Task（タスク） をドレイン。

**下位の仕組み（後回し可、本ステップで一度に掘り切らなくてよい）：** Fargate（ファーゲート） がどの下位ホスト（Host）へ載せるか、ENI（イーエヌアイ）／パブリック（public） IP（アイピー） の付け方、ALB（エーエルビー） ノードとターゲット登録プロトコル（Protocol）、ヘルスチェック（health check）失敗後の置換方針、イメージ（Image）取得・ログ（Log）書き込みへの IAM（アイエーエム） 資格情報の注入など——障害調査が必要になったときに公式ドキュメント（document）を参照する。

記入日：2026-10-27。  
費用の取り決め：疎通後、**存在しているだけで課金されるものを削除**（主に ALB（エーエルビー））；それ以外は停止／Desired（デザイアード）=0。空 Cluster（クラスター）、ECR（イーシーアール） `v1` は残してよい；既存の S3（エススリー） / CloudFront（クラウドフロント） は触らない。  
Region（リージョン） は右上で先に選択：**アジアパシフィック（東京）ap-northeast-1**。  
イメージ（Image）：`853900965180.dkr.ecr.ap-northeast-1.amazonaws.com/device-dashboard-api:v1`  
コンテナ（Container）ポート（Port）：`8005`；ヘルスチェック（health check）パス：`/`（`/devices` は使わない）。  
`/devices` はステップ 4 で RDS（アールディーエス） 接続後；本ステップの成功基準は `curl http://ALBのDNS/` が通ること。

#### A.1 ECS（イーシーエス） Cluster（クラスター） の作成

1. 上部検索で `ECS` → **Elastic Container Service（エラスティックコンテナサービス）** を開く
2. 左の **Clusters（クラスターズ）**
3. **Create cluster（クリエイトクラスター）**
4. **Cluster（クラスター） name**：`device-dashboard-cluster`
5. **Infrastructure** で **AWS（エーダブリューエス） Fargate（ファーゲート） (serverless)** にチェック（check）
6. 他はそのまま
7. 最下部の **Create（クリエイト）**
8. 一覧に当該 Cluster（クラスター） が出るまで待つ

#### A.2 Task Definition（タスクデフィニション） の作成

**概念メモ — Task Definition（タスクデフィニション） とは：**

- **Task Definition（タスクデフィニション）**：ECS（イーシーエス） 向けの「コンテナ（Container）の動かし方」の説明書。いま動いているプロセス（Process）そのものではない。
- 内容：どの Image（イメージ）、CPU（シーピーユー）/メモリ、ポート（Port）、環境変数、ログ（Log）の行き先、どの IAM（アイエーエム） Role（ロール） など。
- 実際に動いている 1 つを **Task（タスク）** と呼ぶ；**Service（サービス）** がこの定義に従って複数 Task（タスク） を維持する（落ちたら再起動など）。
- 設定変更は通常新しい **revision（リビジョン）** になる（例：`device-dashboard-api:1` → `:2`）；リリース／ロールバック（rollback）は Image（イメージ） tag（タグ） や revision（リビジョン） の切り替えで行うことが多い。
- 手元との比喩：クラウド上の「compose／実行仕様」に近い——動かし方を定義し、Cluster（クラスター）/Service（サービス） がどこで・何個維持するかを担う。

**2 つの Role（作成時に混同しない）：**

| 名称 | 意味 | 本ステップ |
|---|---|---|
| **Task（タスク） execution role** | ECS（イーシーエス） エージェント用：ECR（イーシーアール） からイメージ（Image）取得、CloudWatch Logs（クラウドウォッチログス） への書き込み | **選択／作成する**（よくある名前 `ecsTaskExecutionRole`） |
| **Task（タスク） role** | コンテナ（Container）内アプリが他の AWS（エーダブリューエス） サービス（例：S3（エススリー））へアクセス（Access）するとき用 | **本ステップは空のまま**；`s3:GetObject` 付きの作成ウィザードには進まない |

1. 左の **Task（タスク） definitions**
2. **Create（クリエイト） new task definition**
3. **Task（タスク） definition family（ファミリー）**：`device-dashboard-api`
4. Launch type（ローンチタイプ） / OS：**AWS（エーダブリューエス） Fargate（ファーゲート）**、OS は Linux
5. **CPU（シーピーユー）**：`0.25 vCPU`；**Memory**：`0.5 GB`
6. **Task（タスク） execution role**：`ecsTaskExecutionRole` があれば選択；なければ案内に従って作成してから選択。**Task（タスク） role** は本ステップ空でよい
7. **Container（コンテナ）**：
   - Name（ネーム）：`api`
   - Image（イメージ） URI（ユーアールアイ）：`853900965180.dkr.ecr.ap-northeast-1.amazonaws.com/device-dashboard-api:v1`
   - Essential：Yes
   - Container（コンテナ） port：`8005`、プロトコル（Protocol） TCP（ティーシーピー）
8. **CloudWatch Logs（クラウドウォッチログス）** を有効化（デフォルトのロググループで可）
9. **Environment variable（エンバイロメントバリアブル）s（エンバイロメントバリアブルズ）** を追加：

| Key（キー） | Value（バリュー） |
|---|---|
| POSTGRES_USER | postgres（ポストグレス） |
| POSTGRES_PASSWORD | temporary-change-in-step4 |
| POSTGRES_DB（ディービー） | device_dashboard |
| POSTGRES_HOST | 127.0.0.1 |
| POSTGRES_PORT | 5432 |

10. Command（コマンド） は **カスタムしない**、alembic も **書かない**
11. **Create（クリエイト）** → `device-dashboard-api:1` などができることを確認
12. 作成後に環境変数が 0 なら：当該 revision（リビジョン） で **Create new revision（クリエイトニューリビジョン）**（フォーム版を選び、JSON（ジェイソン） は選ばない）→ コンテナ（Container） `api` → **Environment and secrets** → **Add environment variable（エンバイロメントバリアブル）** で 5 つの `POSTGRES_*` を補完 → Create（クリエイト） で `:2`。Service（サービス） は環境変数付きの最新 revision（リビジョン） を使う。

**構成メモ（本ラウンドでつまずいた点）：** Apple Silicon 手元の既定ビルド（Build）は ARM（アーム） イメージ（Image）；Fargate（ファーゲート） Task（タスク） が `X86_64` だと `exec format error` になる。クラウドへ上げる前に：

```bash
docker build --platform linux/amd64 -t device-dashboard-api:latest .
```

その後 tag（タグ） / push（プッシュ）；既にデプロイ（Deploy）済みなら Update service（アップデートサービス） で **Force new deployment（フォースニューデプロイメント）** にチェック（check）。

#### A.3 Target Group（ターゲットグループ） の作成

**概念メモ — Target type の 4 選択肢それぞれ：**

Target Group（ターゲットグループ） 作成時の **Target type** は「ALB（エーエルビー） がトラフィック（Traffic）をどの種類のターゲットへ送るか」。誤りだと後で ECS（イーシーエス）/Fargate（ファーゲート） をぶら下げられない。

| Target type | 意味 | 本ラウンド（round） |
|---|---|---|
| **Instances** | ターゲットは **EC2（イーシーツー） インスタンス（Instance）**（インスタンス単位で TG（ティージー） に登録） | 使わない。本ラウンド（round）は Fargate（ファーゲート） で、自分で管理する EC2（イーシーツー） がない |
| **IP（アイピー） addresses** | ターゲットは **IP（アイピー） アドレス**（コンテナタスクが得たプライベート IP（アイピー） 等） | **これを使う**。Fargate（ファーゲート） の Task（タスク） は IP（アイピー） 形式で TG（ティージー） に載る |
| **Lambda function** | ターゲットは **Lambda 関数**；ALB（エーエルビー） が直接 Lambda を呼ぶ | 使わない。本ラウンド（round）はコンテナ（Container） API（エーピーアイ） であり Lambda ではない |
| **Application Load Balancer（アプリケーションロードバランサー）** | ターゲットは **別の ALB（エーエルビー）**（LB がさらに下位の ALB（エーエルビー） へ） | 使わない。多段 ALB（エーエルビー） 構成向け；最小実践では不要 |

本ラウンド（round）の結論：**IP（アイピー） addresses** を選ぶ。

1. `EC2` を検索 → **EC2（イーシーツー）** を開く
2. 左 **Load Balancing** → **Target Group（ターゲットグループ）s** → **Create（クリエイト） target group**
3. **Target type**：**IP（アイピー） addresses**
4. Name（ネーム）：`device-dashboard-api-tg`
5. Protocol（プロトコル）：HTTP（エイチティーティーピー）；Port（ポート）：`8005`
6. VPC（ブイピーシー）：デフォルト（Default） VPC（または本ラウンドで統一して使う VPC（ブイピーシー））
7. Health check（ヘルスチェック） path：`/`；Success codes：`200`
8. Next → **Register targets** では IP（アイピー） を手動追加しない → **Create（クリエイト） target group**

#### A.4 ALB（エーエルビー） の作成

**概念メモ — 作成ページ（Page）の 3 種 Load Balancer の違いと関係：**

コンソール（Console） **Create（クリエイト） load balancer** によく出る 3 枚のカードは、いずれも「後ろのターゲットへ振り分ける」が、働くネットワーク（Network）層と典型用途が異なる：

| 種類 | 正式名称 | おおよその層 | 何を見て転送するか | 典型用途 |
|---|---|---|---|---|
| **Application Load Balancer（アプリケーションロードバランサー）** | アプリケーション負荷分散 | 第 7 層 | URL（ユーアールエル） パス、Host（ホスト） ヘッダ、HTTP（エイチティーティーピー） ヘッダなど | Web / API（エーピーアイ）（本ラウンド（round） `GET /devices`） |
| **Network（ネットワーク） Load Balancer（ネットワークロードバランサー）** | ネットワーク（Network）負荷分散 | 第 4 層 | IP（アイピー） + ポート（Port）など | 超高同時接続、極低遅延、非 HTTP（エイチティーティーピー）、他 LB／サービス（Service）の手前に置く場合 |
| **Gateway Load Balancer（ゲートウェイロードバランサー）** | ゲートウェイ負荷分散 | 第 3／ゲートウェイ寄り（VPC（ブイピーシー） Endpoint（エンドポイント） 等と併用） | トラフィック（Traffic）を「検査／セキュリティ機器」系ターゲットへ送る | ファイアウォール、侵入検知などの仮想アプライアンスを経路に挿入；通常の Web API（エーピーアイ） 入口ではない |

**関係（覚え方）：**

- 3 つとも **Elastic Load Balancing（エラスティックロードバランシング）** ファミリー（family）で、複数ターゲット・ヘルスチェック（health check）・AZ（エーゼット） またぎが可能。
- **どれを選ぶかはトラフィック（Traffic）種別**：ブラウザ／HTTP（エイチティーティーピー） API（エーピーアイ） → まず **ALB（エーエルビー）**；TCP（ティーシーピー） ポート（Port）転送のみ、または極限性能 → 多くは **NLB（エヌエルビー）**；セキュリティ機器チェーンへ通す → **GWLB**。
- 組み合わせも可能（発展）：例として外側 NLB（エヌエルビー）、内側 ALB（エーエルビー）；本ラウンド（round）の最小実践は **ALB（エーエルビー） のみ**作成し、NLB（エヌエルビー）/GWLB は作らない。

**本ラウンド（round）の結論：** **Application Load Balancer（アプリケーションロードバランサー）** カードの **Create（クリエイト）** を押す。

1. EC2（イーシーツー） → **Load Balancers** → **Create（クリエイト） load balancer** → **Application Load Balancer（アプリケーションロードバランサー）** → **Create（クリエイト）**
2. Name（ネーム）：`device-dashboard-alb`
3. Scheme：Internet-facing（インターネットフェイシング）；IP（アイピー）：IPv4
4. VPC（ブイピーシー） は TG（ティージー） と同じ；少なくとも **2 つの AZ（エーゼット）** にチェック（check）し、それぞれ **パブリック（public）サブネット（Subnet）** を選ぶ
5. Security group：新規 `device-dashboard-alb-sg`  
   - インバウンド：Type（タイプ） HTTP（エイチティーティーピー）、Port（ポート） `80`、Source（ソース） `0.0.0.0/0`  
   - ALB（エーエルビー） ウィザードに戻って当該 SG（エスジー） を選択
6. Listener（リスナー）：HTTP（エイチティーティーピー） `80` → Forward（フォワード） to `device-dashboard-api-tg`
7. **Create（クリエイト） load balancer**、**Active** になるまで待つ
8. ALB（エーエルビー） の **DNS（ディーエヌエス） name** をメモ帳へコピー

#### A.5 ECS（イーシーエス） 用 Security Group（セキュリティグループ） の作成

**概念メモ — Source（ソース） に Security group（`device-dashboard-alb-sg`）を選ぶ意味：**

- Security Group（セキュリティグループ） のインバウンド規則は **「誰が → 自分のどのポート（Port）へアクセス（Access）してよいか」** と読める。
- **Source（ソース）** が「誰」：
  - `0.0.0.0/0`：インターネット上の任意 IP（アイピー） がそのポート（Port）へ届く（ECS（イーシーエス） の `8005` には面が大きすぎて不安全；本ラウンド（round）ではしない）。
  - **別の Security Group（セキュリティグループ）**（ここでは `device-dashboard-alb-sg`）：**その SG（エスジー） が付いた NIC／リソース（resource）**からのトラフィック（Traffic）だけ許可——実質 **ALB（エーエルビー）** からのみ。
- 経路は：  
  `Browser → ALB:80（alb-sg が全世界→80 を許可）→ ECS Task:8005（ecs-sg が alb-sg のみ許可）`
- 外からは ALB（エーエルビー） の 80 にしか届かず、**コンテナ（Container） 8005 を直接スキャンできない**；ALB（エーエルビー） が ECS（イーシーエス） へ転送する。
- 「`device-dashboard-alb-sg` を選ぶ」は IP（アイピー） を書くのではなく、**その SG（エスジー） を使っている ALB（エーエルビー） の ENI（イーエヌアイ） をすべて信頼する**こと；ALB（エーエルビー） の拡大縮小や IP（アイピー） 変更でも規則を直さなくてよい。

1. EC2（イーシーツー） → **Security Group（セキュリティグループ）s** → **Create security group（クリエイトセキュリティグループ）**
2. Name（ネーム）：`device-dashboard-ecs-sg`；同一 VPC（ブイピーシー）
3. インバウンド：Custom TCP（ティーシーピー）、Port（ポート） `8005`、Source（ソース） にセキュリティグループ（Security Group） `device-dashboard-alb-sg`（全世界へ 8005 を開けない）
4. 作成

#### A.6 ECS（イーシーエス） Service（サービス） の作成

1. ECS（イーシーエス） → Clusters（クラスターズ） → `device-dashboard-cluster` → **Services** → **Create（クリエイト）**
2. Compute：Fargate（ファーゲート）
3. Task（タスク） definition：`device-dashboard-api`（最新 revision）
4. Service（サービス） name：`device-dashboard-api-svc`；Desired（デザイアード） tasks：`1`
5. Networking（ネットワーキング）：同一 VPC（ブイピーシー）；**パブリック（public）サブネット（Subnet）**；SG（エスジー） は `device-dashboard-ecs-sg`；**Public IP（パブリックアイピー）：Turned on**
6. Load balancing（ロードバランシング）：既存 ALB（エーエルビー） `device-dashboard-alb`；コンテナ（Container） `api` ポート（Port） 8005；Listener（リスナー） HTTP（エイチティーティーピー） 80；TG（ティージー） は `device-dashboard-api-tg`
7. Create（クリエイト） → Task（タスク） が **Running（ランニング）**；TG（ティージー） Targets が **healthy（ヘルシー）** になるまで待つ
8. 失敗時：ECS（イーシーエス） → 当該 Task（タスク） → Logs（ログス） でエラーを残す

#### A.7 疎通確認

```bash
curl http://あなたのALBのDNS/
```

期待例：`{"message":"backend in container"}`

#### B. 疎通後の費用対応

##### B.1 Desired（デザイアード） を 0 にする

1. ECS（イーシーエス） → Clusters（クラスターズ） → `device-dashboard-cluster` → Services → `device-dashboard-api-svc`
2. **Update service（アップデートサービス）** → Desired（デザイアード） tasks を `0` → 保存
3. Running（ランニング） tasks = 0 になるまで待つ

##### B.2 ALB（エーエルビー） の削除（存在しているだけで課金）

1. EC2（イーシーツー） → Load Balancers → `device-dashboard-alb` を選択
2. **Actions** → **Delete（デリート） load balancer** → 確認
3. 一覧から消えるまで待つ

##### B.3 Target Group（ターゲットグループ） の削除

1. EC2（イーシーツー） → Target Group（ターゲットグループ）s → `device-dashboard-api-tg` を選択
2. **Actions** → **Delete（デリート）**（まだ参照されている場合は ALB（エーエルビー） 削除後に再試行）

##### B.4 残す／触らない

- 残してよい：空 Cluster（クラスター）、Desired（デザイアード）=0 の Service（または Service（サービス） を削除して次回再作成）、ECR（イーシーアール） `v1`、2 つの SG（エスジー）
- 触らない：既存の S3（エススリー） / CloudFront（クラウドフロント） フロントエンド（Frontend）
- 次回ステップ 4/5 で ALB（エーエルビー） を再作成し Service（サービス） を起動する

---

### ステップ 4：RDS（アールディーエス） 接続（コンソール手順）

記入日：2026-10-27。  
前提：ステップ 3 は疎通済みで ALB（エーエルビー）/TG（ティージー） 削除済み、Service（サービス） Desired（デザイアード）=0；Cluster（クラスター） / Task Definition（タスクデフィニション） / ECR（イーシーアール） `v1` / `device-dashboard-ecs-sg` は残っている。  
費用：RDS（アールディーエス） は **存在しているだけで課金**（コンピュート＋ストレージ）；ALB（エーエルビー） 再作成後も同様。疎通後は本節 **E** で停止・削除。  
注意：RDS（アールディーエス） 作成時に **Manage master credentials in AWS（エーダブリューエス） Secrets（シークレッツ） Manager（シークレッツマネージャー）** が出たら **自己管理パスワード（Password） / Self managed（セルフマネージド）** を選び、Secrets（シークレッツ） Manager（シークレッツマネージャー） の追加月額を避ける。

> **学習メモ（ユーザー）**：AWS（エーダブリューエス） Secrets（シークレッツ） Manager（シークレッツマネージャー） はまだ体系的に学んでいない。本ラウンド（round）の cleanup（クリーンアップ） で初めてコンソール（Console）を開いた。今後学ぶこと：何を保管するか、RDS（アールディーエス） の Self managed（セルフマネージド） パスワード（Password）との違い、月額が発生しうる理由、コンソール（Console）で secret（シークレット） を見つけて削除する方法。2026-10-28 東京 Region（リージョン） の一覧は空（本ラウンドの残留なし）。

成功基準：`curl http://ALBのDNS/devices` が機器一覧 JSON（ジェイソン） を返す（データは RDS（アールディーエス） 由来）。

#### C.1 RDS（アールディーエス） 用 Security Group（セキュリティグループ） の作成

1. EC2（イーシーツー） または VPC（ブイピーシー） → **Security Group（セキュリティグループ）s** → **Create security group（クリエイトセキュリティグループ）**
2. Name（ネーム）：`device-dashboard-rds-sg`；VPC（ブイピーシー）：ECS（イーシーエス） と同じ
3. インバウンド：Type（タイプ） **PostgreSQL（ポストグレスキューエル）** または Custom TCP（ティーシーピー）、Port（ポート） **`5432`**  
   Source（ソース）：セキュリティグループ（Security Group） **`device-dashboard-ecs-sg`**（`0.0.0.0/0` は使わない）
4. 作成

#### C.2 RDS（アールディーエス） PostgreSQL（ポストグレスキューエル） の作成（短時間・最小）

1. **RDS（アールディーエス）** を検索 → **Databases** → **Create database（クリエイトデータベース）**
2. **Standard create（スタンダードクリエイト）** を選ぶ
3. Engine（エンジン）：**PostgreSQL（ポストグレスキューエル）**（バージョンはデフォルトで可）
4. Templates（テンプレートズ）：優先 **Free tier（フリーティア）**（アカウントに表示があれば）；なければ Dev/Test、できるだけ最小インスタンス（Instance）
5. Settings（セッティングス）：
   - DB（ディービー） instance identifier（ディービーインスタンスアイデンティファイア）：`device-dashboard-db`
   - Master username（マスターユーザーネーム）：`postgres`
   - パスワード（Password）：学習用を自分で決めて手元に控える（**git に書かない／チャットに送らない**）
   - 認証情報管理：**Self managed（セルフマネージド）**（Secrets Manager（シークレッツマネージャー） は使わない）
6. Instance（インスタンス）：できるだけ小さく（例：`db.t4g.micro` / `db.t3.micro`）
7. Storage（ストレージ）：デフォルト（Default） 20 GiB で可；自動拡張の上限を大きくしない（Storage autoscaling をオフ、またはごく小さく）
8. Connectivity（コネクティビティ）：
   - VPC（ブイピーシー）：ECS（イーシーエス） と同じデフォルト（Default） VPC（ブイピーシー）
   - Public access（パブリックアクセス）：**No**
   - VPC（ブイピーシー） security group：**`device-dashboard-rds-sg`**（可能なら余分な default を外す）
9. Additional（アディショナル）：Initial database name（イニシャルデータベースネーム） に **`device_dashboard`**
10. 監視／拡張などオフにできるものはデフォルト（Default）のままかオフにして費用を抑える
11. **Create database（クリエイトデータベース）** → Status（ステータス） = **Available（アベイラブル）** まで待つ（多くの場合 5〜10 分）
12. インスタンス（Instance）を開き **Endpoint（エンドポイント）** をコピー（形：`device-dashboard-db.xxxx.ap-northeast-1.rds.amazonaws.com`）

#### C.3 Target Group（ターゲットグループ） + ALB（エーエルビー） の再作成

ステップ 3 の A.3 / A.4 と同じ（前回削除済みのため再作成）：

1. TG（ティージー）：`device-dashboard-api-tg`；Target type **IP（アイピー）**；Port（ポート） **8005**；Health check（ヘルスチェック） **`/`**；IP（アイピー） は手動登録しない
2. ALB（エーエルビー）：`device-dashboard-alb`；Internet-facing（インターネットフェイシング）；パブリック（public）サブネット（Subnet） 2 つ；SG（エスジー） は既存 `device-dashboard-alb-sg`（残っていれば）または A.4 どおり再作成；Listener（リスナー） HTTP（エイチティーティーピー） 80 → 当該 TG（ティージー）
3. 新しい ALB（エーエルビー） DNS（ディーエヌエス） をコピー

#### C.4 新しい Task Definition（タスクデフィニション） revision（実 DB（ディービー） 接続 + 起動時マイグレーション）

前提：family（ファミリー） `device-dashboard-api` と revision（リビジョン） `:1` / `:2` は **既存**（ステップ 3 で残したもの）。本ステップは family（ファミリー） の新規作成ではなく、最新（記録上 `:2`）に対し **Create new revision（クリエイトニューリビジョン）** し、接続情報と起動コマンド（Command）を変えて `:3` などの新番号を得る。Task Definition（タスクデフィニション） / revision（リビジョン） 自体は「説明書を増やす」ことでは課金されない。

1. ECS（イーシーエス） → Task（タスク） definitions → `device-dashboard-api` → 最新（例：`:2`）→ **Create new revision（クリエイトニューリビジョン）**
2. コンテナ（Container） `api` → **Environment variable（エンバイロメントバリアブル）s（エンバイロメントバリアブルズ）** を次のように変更（HOST は自分の Endpoint（エンドポイント））：

| Key（キー） | Value（バリュー） |
|---|---|
| POSTGRES_USER | postgres（ポストグレス） |
| POSTGRES_PASSWORD | （下記「どのパスワードか」） |
| POSTGRES_DB（ディービー） | device_dashboard |
| POSTGRES_HOST | （RDS（アールディーエス） Endpoint（エンドポイント）。`http://` もポート（Port）も付けない） |
| POSTGRES_PORT | 5432 |

**どのパスワード（Password）か：**

- RDS（アールディーエス） 作成時 **Credentials（クレデンシャルズ）** で自分で設定した **Master password（マスターパスワード）**（Master username（マスターユーザーネーム） `postgres` と対）。
- AWS（エーダブリューエス） ログ（Log）インパスワード（Password）でも、IAM（アイエーエム） ユーザー（User）パスワード（Password）でも、昨日のプレースホルダ `temporary-change-in-step4` でも **ない**。
- コンソール（Console）は当該パスワード（Password）を再表示しない。忘れたら作成時にメモしたものを使う。無ければ RDS（アールディーエス） で **Reset master password（リセットマスターパスワード）** してから新 revision（リビジョン） に入れる（短時間練習では作成時メモが普通）。
- 実パスワード（Password）を git に書かない。チャットにも送らない。

3. コンテナ（Container）内の **Docker（ドッカー） configuration - optional** を展開（よく折りたたまれている。ページ最下部の Storage / Tags ではない）。下表どおり **Entry point（エントリーポイント）** / **Command（コマンド）** / **Working directory（ワーキングディレクトリ）** を入れる。

**概念メモ — Entry point（エントリーポイント）・Command（コマンド）・Working directory（コンソール三欄。後で読む用）：**

| コンソール（Console）欄 | Docker（ドッカー） / イメージ（Image）側 | 役割（本ラウンド） |
|---|---|---|
| **Entry point（エントリーポイント）** | **ENTRYPOINT** | コンテナ（Container）が先に必ず実行するプログ（Log）ラム（と固定引数）。本ラウンド（round）は `sh,-c`＝shell（シェル） の `-c` で後ろのスクリプト（script）全体を実行する。 |
| **Command（コマンド）** | **CMD** | Entry point（エントリーポイント） に渡す引数。Entry point（エントリーポイント） が既に `sh` + `-c` なら、ここは**実行するスクリプト（script）文字列だけ**。`sh -c` を重ねて書かない（起動コマンドが壊れる）。 |
| **Working directory（ワーキングディレクトリ）** | **WORKDIR** | コマンド（Command）のカレント（current）ディレクトリ（directory）。本プロジェクト（Project） Dockerfile（ドッカーファイル） は `WORKDIR /app`。`/app` にするか空欄でイメージ（Image）既定。`/usr/app` は誤り。 |

効き方の覚え方：Working directory（ワーキングディレクトリ） の下で概ね `ENTRYPOINT` + `CMD` → 本ラウンド（round）は  
`sh -c "python -m alembic upgrade head && python -m uvicorn ..."`。  
未変更ならイメージ（Image）元の `CMD`（uvicorn のみ、alembic なし）。本ステップで変えるのは、起動時に先にマイグレーション（migration）してから API（エーピーアイ） を上げるため。

コンソール（Console）は **Comma delimited（コンマデリミテッド）**。推奨：

| 欄 | 値 |
|---|---|
| **Entry point（エントリーポイント）** | `sh,-c` |
| **Command（コマンド）** | `python -m alembic upgrade head && python -m uvicorn main:app --host 0.0.0.0 --port 8005`（一塊。先頭に `sh -c` を付けない） |
| **Working directory（ワーキングディレクトリ）** | `/app`（または空） |

保存後 JSON（ジェイソン） が分かるなら、おおよそ `"entryPoint": ["sh","-c"]`、`"command": ["python -m alembic upgrade head && ..."]`。

4. Image（イメージ） は引き続き `.../device-dashboard-api:v1`（**linux/amd64** 版であること）
5. **Create（クリエイト）** → 新 revision（リビジョン）（例：`:3`）

#### C.5 Service（サービス） 更新：ALB（エーエルビー） を再接続、Desired（デザイアード）=1

1. ECS（イーシーエス） → `device-dashboard-api-svc` → **Update service（アップデートサービス）**
2. Task（タスク） definition は最新 revision（RDS（アールディーエス） 環境変数付き）
3. Desired（デザイアード） tasks：`1`
4. Networking（ネットワーキング） 確認：パブリック（public）サブネット（Subnet）、SG（エスジー）=`device-dashboard-ecs-sg`、Public IP（パブリックアイピー） on
5. Load balancing（ロードバランシング）：再作成した ALB（エーエルビー） + TG（ステップ 3 と同じ付け方）
6. **Force new deployment（フォースニューデプロイメント）** にチェック（check） → Update（アップデート）
7. Task（タスク） Running（ランニング）、TG（ティージー） **1 Healthy（ヘルシー）** まで待つ
8. 再び Failed（フェイルド） なら CloudWatch（クラウドウォッチ） `/ecs/device-dashboard-api` のログ（Log）を見る

#### C.6 サンプルデータの投入

一覧が要るので、一度 `seed.py` を実行する（イメージ内にファイルあり）。

1. ECS（イーシーエス） → Clusters（クラスターズ） → `device-dashboard-cluster` → **Tasks（タスクズ）** タブ（Tab） → **Run new task（ランニュータスク）**（または Run task）
2. Launch type（ローンチタイプ）：Fargate（ファーゲート）；Task（タスク） definition：Service（サービス） と同じ最新 revision（リビジョン）
3. Networking（ネットワーキング）：同一 VPC（ブイピーシー）、パブリック（public）サブネット（Subnet）、`device-dashboard-ecs-sg`、Public IP（パブリックアイピー） on
4. コンテナ（Container）コマンド（Command）を seed（シード） のみに上書き（例）：

```text
python seed.py
```

5. 実行し、Task（タスク） が止まるまで待つ；ログ（Log）に `Sample device data is ready.` が出るはず
6. （一回限りの Task（タスク） なので Stopped で正常）

#### C.7 疎通確認

```bash
curl http://あなたのALBのDNS/
curl http://あなたのALBのDNS/devices
```

期待：`/` に message（メッセージ） あり；`/devices` が機器入りの JSON（ジェイソン） 配列を返す。

#### E. ステップ 4 疎通後の費用対応

| 操作 | 対象 |
|---|---|
| Desired（デザイアード） → 0 | `device-dashboard-api-svc` |
| 削除 | ALB（エーエルビー）、Target Group（ターゲットグループ） |
| 削除 | RDS（アールディーエス） インスタンス（Instance） `device-dashboard-db`；作成時に最終スナップショット（Snapshot）保持を聞かれたら **No**；事後 RDS（アールディーエス） → Snapshots（スナップショットズ） で余分なスナップショット（Snapshot）がないことを確認 |
| 確認 | Secrets（シークレッツ） Manager（シークレッツマネージャー） secret（シークレット） を誤って開いていないか → あれば削除 |
| 残してよい | Cluster（クラスター）、ECR（イーシーアール） `v1`、SG（エスジー）、Task Definition（タスクデフィニション） |

RDS（アールディーエス） は **Stop（ストップ） しただけで片付け完了としない**（停止中もストレージ等の費用が残ることがある）；本ラウンド（round）終了時は **Delete（デリート）** する。

---

### ステップ 5：Frontend（フロントエンド） 接続（公開前の CLI（シーエルアイ） 確認を含む）

記入日：2026-10-28。  
本ラウンド（round）既存：S3（エススリー） Bucket（バケット） `my-aws-learning-853900965180-ap-northeast-1-an`；CloudFront（クラウドフロント） Distribution（ディストリビューション） ID（アイディー） `E1JYIKKU2FABXH`、ドメイン（Domain） `d2femt4u2ijui6.cloudfront.net`。  
`Frontend/src/api.ts` の `API_BASE_URL` は ALB（エーエルビー）：`http://device-dashboard-alb-580734253.ap-northeast-1.elb.amazonaws.com`。  
Backend（バックエンド） `allow_origins` に `https://d2femt4u2ijui6.cloudfront.net` を含め、Image（イメージ） を再 push（プッシュ） / Force new deployment（フォースニューデプロイメント） 済みであること。

#### F.1 公開前：CLI（シーエルアイ） で自分で確認（操作練習）

手元の端末で順に実行：

```bash
# 1. 今の CLI がどのアカウントか
aws sts get-caller-identity

# 2. S3 にどの Bucket があるか
aws s3 ls

# 3. 対象 Bucket の中身
aws s3 ls s3://my-aws-learning-853900965180-ap-northeast-1-an/

# 4. CloudFront Distribution が合っているか、Origin がこの Bucket か
aws cloudfront get-distribution --id E1JYIKKU2FABXH --query 'Distribution.{Id:Id,Domain:DomainName,Status:Status,Origin:DistributionConfig.Origins.Items[0].DomainName}' --output json

# 5. Frontend の API アドレス（ALB であること）
cat "/Users/mi/DESK/GL/AWS Learning/device-dashboard/Frontend/src/api.ts"
```

確認基準：Account（アカウント） `853900965180` / `touzui-learning`；Bucket（バケット） 名は上表；CloudFront（クラウドフロント） Domain（ドメイン） は `d2femt4u2ijui6.cloudfront.net` かつ Origin（オリジン） がその Bucket（バケット）；`api.ts` は ALB（エーエルビー） アドレス。

##### コマンド（Command） 4 の分段説明（`aws cloudfront get-distribution ...`）

全文：

```bash
aws cloudfront get-distribution --id E1JYIKKU2FABXH --query 'Distribution.{Id:Id,Domain:DomainName,Status:Status,Origin:DistributionConfig.Origins.Items[0].DomainName}' --output json
```

| 断片 | 何か | どう効くか |
|---|---|---|
| `aws` | AWS（エーダブリューエス） CLI（シーエルアイ） | 手元の資格情報で AWS（エーダブリューエス） に API（エーピーアイ） を送る |
| `cloudfront` | サービス（Service）名 | 今回呼ぶのは **CloudFront（クラウドフロント）**（S3（エススリー）/ECS（イーシーエス） ではない） |
| `get-distribution` | サブコマンド（Command） / API（エーピーアイ） | ID（アイディー） 指定で **1 つ**の Distribution（ディストリビューション） の設定と状態を取得（一覧ではない） |
| `--id E1JYIKKU2FABXH` | Distribution（ディストリビューション） ID（アイディー） | どの配信設定を見るかを指定する |
| `--query '...'` | JMESPath（ジェイメスパス） フィルタ | 生の応答は大きい。`--query` は**手元**で必要なフィールドだけ抜き出す。中身は次表 |
| `--output json` | 出力形式 | 結果を JSON（ジェイソン） で表示する |

`--query` の中：

| 断片 | 意味 |
|---|---|
| `Distribution` | 戻り JSON（ジェイソン） の `Distribution` オブジェクトに入る |
| `{Id:Id, Domain:DomainName, ...}` | 新しいオブジェクトを組む。左が出力名、右が元データのパス |
| `Id` | Distribution（ディストリビューション） ID（アイディー）（`E1JYIKKU2FABXH` と一致するはず） |
| `DomainName` → 出力の `Domain` | Browser（ブラウザー） が開くドメイン（Domain）（例：`d2femt4u2ijui6.cloudfront.net`） |
| `Status` | 例：`Deployed`（設定がエッジ側に行き渡ったか） |
| `DistributionConfig.Origins.Items[0].DomainName` → 出力の `Origin` | **最初の Origin（オリジン）** のドメイン（Domain）。本ラウンド（round）では当該 S3（エススリー） Bucket（バケット） のドメイン（Domain）であるべき |

一言：コマンド（Command） 4 = **指定 Distribution（ディストリビューション） の詳細を取り、ID（アイディー） / 公開ドメイン（Domain） / 状態 / 最初の Origin（オリジン） だけ表示**し、「Frontend（フロントエンド） をどこへ出し、ユーザー（User）がどの URL（ユーアールエル） を開くか」を確認する。

#### F.2 確認後：ビルド（Build）してアップロード（Upload）

```bash
cd "/Users/mi/DESK/GL/AWS Learning/device-dashboard/Frontend"
yarn build
aws s3 sync dist/ s3://my-aws-learning-853900965180-ap-northeast-1-an/ --delete
aws cloudfront create-invalidation --distribution-id E1JYIKKU2FABXH --paths "/*"
```

`create-invalidation` 成功時は JSON（ジェイソン） が返り、**Invalidation（インバリデーション） Id（アイディー）**（例：`I8UOJJ47XWERL3Q9J547YW2XUW`）と **Status（ステータス）**（作成直後は多くの場合 `InProgress`）が含まれる。次で `Completed` になったか確認する（`--id` は自分の戻り値の Id（アイディー） に差し替える）：

```bash
aws cloudfront get-invalidation --distribution-id E1JYIKKU2FABXH --id I8UOJJ47XWERL3Q9J547YW2XUW --query 'Invalidation.Status' --output text
```

期待：一行で `Completed`。まだ `InProgress` なら少し待って再実行。

##### Invalidation（インバリデーション） 状態照会：コマンド（Command）分段説明

全文：

```bash
aws cloudfront get-invalidation --distribution-id E1JYIKKU2FABXH --id I8UOJJ47XWERL3Q9J547YW2XUW --query 'Invalidation.Status' --output text
```

| 断片 | 何か | どう効くか |
|---|---|---|
| `aws` | AWS（エーダブリューエス） CLI（シーエルアイ） | 手元の資格情報で AWS（エーダブリューエス） にリクエスト（Request）する |
| `cloudfront` | サービス（Service）名 | 今回は CloudFront（クラウドフロント） を呼ぶ |
| `get-invalidation` | サブコマンド（Command） | **ある 1 回**の Invalidation（キャッシュ無効化）の詳細を取得する。新規作成ではない |
| `--distribution-id E1JYIKKU2FABXH` | どの Distribution（ディストリビューション） か | 無効化は特定の配信設定に紐づくため必須 |
| `--id I8UOJJ47XWERL3Q9J547YW2XUW` | どの Invalidation（インバリデーション） か | `create-invalidation` が返した Id（アイディー）。毎回違うので自分の値に置き換える |
| `--query 'Invalidation.Status'` | JMESPath（ジェイメスパス） | 戻り JSON（ジェイソン） から状態だけ取り出す（`InProgress` / `Completed` など） |
| `--output text` | 出力形式 | 状態を 1 行のテキストで出す |

一言：このコマンド（Command） = **CloudFront（クラウドフロント） に「さっき出した `/*` のキャッシュ（Cache）無効化はどこまで進んだ？」と聞く。**

`Completed` のあと Browser（ブラウザー）：`https://d2femt4u2ijui6.cloudfront.net`。  

**Failed（フェイルド） to load devices** かつ Network（ネットワーク） の Request（リクエスト） URL（ユーアールエル） がまだ `http://...elb.amazonaws.com/devices` の場合は **mixed content（ミックスドコンテンツ）**（HTTPS（エイチティーティーピーエス） ページから HTTP（エイチティーティーピー） API（エーピーアイ） 不可）。截図：`../2026-10-28-现象-CloudFront的HTTPS页请求HTTP的ALB被浏览器拦截.png`。**F.3** で API（エーピーアイ） も同じ CloudFront（クラウドフロント） 経由にする。

#### F.3 CloudFront（クラウドフロント） ページ（Page）から API（エーピーアイ） を完全に通す（mixed content（ミックスドコンテンツ） の解消（かいしょう））

考（かんが）え方（かた）：Browser（ブラウザー） → **だけ** `https://d2femt4u2ijui6.cloudfront.net/...` を要求（ようきゅう／Request（リクエスト））；CloudFront（クラウドフロント） が `/devices*` を ALB（エーエルビー） へ転送（てんそう／Forward）する（CloudFront（クラウドフロント）→ALB（エーエルビー） は HTTP（エイチティーティーピー） で可（か））。ページ（Page）と API（エーピーアイ） が同（おな）じドメイン（Domain）になると、クロスオリジン（cross-origin／CORS）にも頼（たよ）らなくなる。

##### F.3-1 コンソール（Console）：Distribution（ディストリビューション） に ALB（エーエルビー） Origin（オリジン） を追加（ついか）

1. CloudFront（クラウドフロント） → Distributions（ディストリビューションズ） → **`E1JYIKKU2FABXH`** を開（ひら）く
2. **Origins（オリジンズ）** タブ（Tab）→ **Create origin（クリエイトオリジン）**
3. 入力（にゅうりょく）：
   - **Origin domain（オリジンドメイン）**：`device-dashboard-alb-580734253.ap-northeast-1.elb.amazonaws.com`（一覧（いちらん）の ALB（エーエルビー） を選（えら）ぶか、DNS（ディーエヌエス） を貼（は）る）
   - **Protocol（プロトコル）**：**HTTP only（エイチティーティーピーオンリー）**
   - **HTTP（エイチティーティーピー） port**：`80`
   - **Name（ネーム）**：例（れい）`device-dashboard-alb`（既定（きてい）のままでも可（か））
4. **Create origin（クリエイトオリジン）**
5. Distribution（ディストリビューション） の **Last modified（ラストモディファイド） / Status（ステータス）** が **Deployed（デプロイド）** に戻（もど）るまで待（ま）つ（数分（すうふん）かかることがある）

##### F.3-2 コンソール（Console）：Behavior（ビヘイビア） を追加（ついか）し、`/devices*` を ALB（エーエルビー） へ向（む）ける

1. 同（おな）じ Distribution（ディストリビューション） → **Behaviors（ビヘイビアズ）** → **Create behavior（クリエイトビヘイビア）**
2. 入力（にゅうりょく）：
   - **Path pattern（パスパターン）**：`/devices*`
   - **Origin（オリジン） and origin groups**：いま作（つく）った **`device-dashboard-alb`** を選（えら）ぶ
   - **Viewer protocol policy（ビューワープロトコルポリシー）**：**Redirect HTTP to HTTPS（リダイレクトエイチティーティーピーツーエイチティーティーピーエス）**
   - **Allowed HTTP methods（アラウドエイチティーティーピーメソッド）**：**GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE** を含（ふく）むようにチェック（本ラウンドは少なくとも GET / POST）
   - **Cache policy（キャッシュポリシー）**：**CachingDisabled（キャッシングディセイブルド）**（マネージド方針（ほうしん）；API（エーピーアイ） はキャッシュしない）
   - **Origin request policy（オリジンリクエストポリシー）**：**AllViewerExceptHostHeader（オールビューワーエクセプトホストヘッダー）**（あれば選（えら）ぶ；無（な）ければ Legacy 等で必要（ひつよう）なヘッダーを転送（てんそう））
3. **Create behavior（クリエイトビヘイビア）**
4. 再（ふたた）び Distribution（ディストリビューション） が **Deployed（デプロイド）** になるまで待（ま）つ

デフォルト（Default）Behavior（ビヘイビア） は引（ひ）き続（つづ）き S3（エススリー）（静的（せいてき）ページ）。`/devices*` だけが ALB（エーエルビー） へ行（い）く。

##### F.3-3 Frontend（フロントエンド） を変更（へんこう）：API（エーピーアイ） を同（おな）じ CloudFront（クラウドフロント） ドメイン（Domain）にする

`Frontend/src/api.ts`：

```ts
const API_BASE_URL = 'https://d2femt4u2ijui6.cloudfront.net'
```

##### F.3-4 再（さい）ビルド（Build）、同期（どうき／sync（シンク））、キャッシュ（Cache）無効化（むこうか／Invalidation（インバリデーション））

```bash
cd "/Users/mi/DESK/GL/AWS Learning/device-dashboard/Frontend"
yarn build
aws s3 sync dist/ s3://my-aws-learning-853900965180-ap-northeast-1-an/ --delete
aws cloudfront create-invalidation --distribution-id E1JYIKKU2FABXH --paths "/*"
```

返（かえ）ってきた Invalidation（インバリデーション） Id（アイディー） で **Completed（コンプリーテッド）** を確認（かくにん）したあと：

1. Browser（ブラウザー） で `https://d2femt4u2ijui6.cloudfront.net` を開（ひら）く（シークレット／強（つよ）い再読（さいよ）みが安心（あんしん））
2. Network（ネットワーク）：`/devices` の Request（リクエスト） URL（ユーアールエル） は **`https://d2femt4u2ijui6.cloudfront.net/devices`**、Status（ステータス） **200**、一覧（いちらん）に機器（きき）が 5 台（だい）
3. 截図（せつず）は「验通-…」形式（けいしき）でプロジェクト（Project）ルート（Root）に保存（ほぞん）

---

### ステップ 6：deploy（デプロイ） と rollback（一連（いちれん）の操作（そうさ））

記入日（きにゅうび）：2026-10-28。  
約束（やくそく）：**tag（タグ） は `v1` のみ**（ユーザー（User）は `v2` を削除（さくじょ）済（ず）み。本ラウンド（round）では `v2` を作（つく）らない・使（つか）わない）。  
確認（かくにん）は ALB（エーエルビー） の `/`（CloudFront（クラウドフロント） のルートは静的（せいてき）ページなので、API（エーピーアイ） の message 変化（へんか）は見（み）えない）。  
Task Definition（タスクデフィニション） の Image（イメージ） は `.../device-dashboard-api:v1` のまま；発版（はっぱん）は **`v1` の上書（うわが）き push（プッシュ） + Force new deployment（フォースニューデプロイメント）**。tag（タグ） 変更（へんこう）のために新（あたら）しい revision（リビジョン） を作（つく）る必要（ひつよう）はない。

#### G.1 deploy（デプロイ）（`v1` を上書（うわが）き）

**1）コード（Code）を変更（へんこう）** `backend/main.py`：

```python
@app.get("/")
async def root():
  return {"message": "backend updated"}
```

**2）手元（てもと）で構築（こうちく／Build（ビルド））し、`v1` を上書（うわが）き push（プッシュ）：**

```bash
cd "/Users/mi/DESK/GL/AWS Learning/device-dashboard/backend"
aws sts get-caller-identity
docker build --platform linux/amd64 -t device-dashboard-api:latest .
aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 853900965180.dkr.ecr.ap-northeast-1.amazonaws.com
docker tag device-dashboard-api:latest 853900965180.dkr.ecr.ap-northeast-1.amazonaws.com/device-dashboard-api:v1
docker push 853900965180.dkr.ecr.ap-northeast-1.amazonaws.com/device-dashboard-api:v1
```

**3）Update Service（アップデートサービス）（Image（イメージ） はまだ `v1`。Task Definition（タスクデフィニション） は変（か）えない）：**  
`device-dashboard-api-svc` → Update（アップデート） → Desired（デザイアード） **1** → **Force new deployment（フォースニューデプロイメント）** にチェック（check） → Update（アップデート） → Task（タスク） が **Running（ランニング）** になるまで待（ま）つ。

**4）deploy（デプロイ） の確認（かくにん）：**

```bash
curl http://device-dashboard-alb-580734253.ap-northeast-1.elb.amazonaws.com/
```

期待（きたい）：`{"message":"backend updated"}`

#### G.2 rollback（ロールバック）（コード（Code）を旧（きゅう）文（ぶん）に戻（もど）し、再（ふたた）び `v1` を上書（うわが）き）

`v1` だけのとき、ロールバック（rollback）＝コード（Code）を旧（きゅう）の返（かえ）り値（ち）に戻（もど）す → 再（さい）build / `v1` 上書（うわが）き push（プッシュ） → Force new deployment（フォースニューデプロイメント）。

**1）`main.py` を戻（もど）す：**

```python
@app.get("/")
async def root():
  return {"message": "backend in container"}
```

**2）G.1 と同（おな）じ build / tag（タグ） / push（プッシュ） `v1` の 4 本（ほん）のコマンド（Command）を再実行（さいじっこう）。**

**3）Service（サービス） を再（ふたた）び Update（アップデート） → Desired（デザイアード） **1** → **Force new deployment（フォースニューデプロイメント）** → Running（ランニング）。**

**4）rollback（ロールバック） の確認（かくにん）：**

```bash
curl http://device-dashboard-alb-580734253.ap-northeast-1.elb.amazonaws.com/
```

期待（きたい）：`{"message":"backend in container"}`

ステップ 6 の完了（かんりょう）目印（めじるし）：先（さき）に `backend updated` を見（み）て、rollback（ロールバック） のあと `backend in container` を見（み）る。

---

### ステップ 7：cleanup（一連（いちれん））

記入日（きにゅうび）：2026-10-28。  
**削除（さくじょ）しないもの：** 既存（きそん）の S3（エススリー） Bucket（バケット）、CloudFront（クラウドフロント） Distribution（静的（せいてき）Frontend（フロントエンド））。  
ALB（エーエルビー） を消（け）すと、CloudFront（クラウドフロント） 上（じょう）の `/devices*` は一時的（いちじてき）に通（とお）らなくなる。次（つぎ）の練習（れんしゅう）で ALB（エーエルビー） を再（さい）接続（せつぞく）すればよい。

順（じゅん）に行（おこな）う：

1. **ECS（イーシーエス） Service（サービス）** `device-dashboard-api-svc` → Update（アップデート） → Desired（デザイアード） **`0`** → Update（アップデート） → Running（ランニング） = 0 まで待（ま）つ  
2. **ALB（エーエルビー） を削除（さくじょ）** `device-dashboard-alb`（EC2（イーシーツー） → Load Balancers）  
3. **Target Group（ターゲットグループ） を削除（さくじょ）** `device-dashboard-api-tg`  
4. **RDS（アールディーエス） を削除（さくじょ）** `device-dashboard-db`：Delete（デリート）；最終（さいしゅう）スナップショット（Snapshot）は **No**；RDS（アールディーエス） → Snapshots（スナップショットズ） で余分（よぶん）なスナップショット（Snapshot）が無（な）いことを確認（かくにん）  
5. **Secrets（シークレッツ） Manager（シークレッツマネージャー） を確認（かくにん）**：本ラウンド（round）で誤（あやま）って secret（シークレット） を開（ひら）いていたら削除（さくじょ）  
   - 入口（いりぐち）：コンソール（Console）で **Secrets（シークレッツ） Manager（シークレッツマネージャー）** を検索（けんさく）→ **Secrets（シークレッツ）**；Region（リージョン） は本ラウンド（round）の **`ap-northeast-1`**  
   - 2026-10-28 確認（かくにん）：一覧（いちらん）は空（から）。本ラウンド（round）の残留（ざんりゅう）なし  
   - **学習（がくしゅう）メモ（ユーザー）**：Secrets（シークレッツ） Manager（シークレッツマネージャー） は初（はじ）めて使（つか）った。学（まな）ぶ内容（ないよう）は上文（じょうぶん）ステップ 4 の RDS（アールディーエス） 作成（さくせい）と同じ（おなじ）注記（ちゅうき）。「一覧に項目（こうもく）があるか」だけをチェック（check）して終（お）わらせない  
6. **残（のこ）してよいもの：** Cluster（クラスター）、Service（Desired（デザイアード）=0）、ECR（イーシーアール） `v1`、Security Group（セキュリティグループ）s、Task Definition（タスクデフィニション）  

RDS（アールディーエス） は Stop（ストップ） だけで終（お）わらせない。本ラウンド（round）終了（しゅうりょう）時は **Delete（デリート）** する。
