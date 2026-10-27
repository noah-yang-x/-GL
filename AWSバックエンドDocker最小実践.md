# AWS バックエンド Docker 最小実践

更新日：2026-10-27  
プロジェクト：`device-dashboard`  
技術 chain：React → Python / FastAPI → Docker → AWS  
費用：学習用アカウントの Free plan / credit；resource は短時間のみ起動し、終了後に cleanup

---

## 一、目標

手元で動作確認済みの Backend Container を AWS に接続し、既存 Frontend と読み取り可能なリクエスト経路をつなぎ、deploy → rollback → cleanup を一通り実施する。

本ラウンドのクラウド上の機能：機器一覧の read-only（`GET /devices`）。

完了時に説明できること：リクエストが Browser からどこへ入るか、Container がどこで動くか、データがどこに保存されるか、ログの確認場所、版の更新と戻し方、練習用 resource の片付け方。

---

## 二、全体構成（1 回のリクエストの順）

```text
Browser
  → CloudFront + S3          Frontend（既存）
  → ALB                      API 入口
  → ECS（Fargate）Task       FastAPI Container を実行
  → RDS PostgreSQL           機器データを保存
  → 同じ経路で Browser へ戻る
```

| 順 | コンポーネント | 何か | 本ラウンドの役割 |
|:---:|---|---|---|
| 1 | S3 | オブジェクトストレージ | React のビルド成果物を保存する |
| 2 | CloudFront | CDN / 静的配信 | ページを Browser へ届ける；既存 Distribution を継続利用 |
| 3 | ALB | Application Load Balancer | API への HTTP リクエストを受け、健全な Task へ転送する；health check を行う |
| 4 | ECR | container registry | version tag 付き Docker Image を保存する |
| 5 | ECS（Fargate） | コンテナオーケストレーション＋サーバーレス実行 | Cluster / Service が Task を維持；Task が ECR から Image を取得し FastAPI を実行 |
| 6 | RDS PostgreSQL | マネージド PostgreSQL | 最小スペック・短時間；API は connection string で接続 |
| 7 | IAM Role | 実行時の主体 | Task Execution Role：Image 取得・ログ書き込み；Task Role：他の AWS resource へのアクセス（必要時） |
| 8 | Security Group | ネットワークのセキュリティグループ | ALB→ECS、ECS→RDS の接続可否を定める |
| 9 | environment variable | 実行時設定 | 起動時に connection string などを Container へ注入；secret は Image の外に置く |
| 10 | CloudWatch Logs | ログサービス | Container / API の出力を確認し、障害調査に使う |

一連の関係は次のように覚える：

**Frontend は CloudFront；Image は ECR；プロセスは ECS；トラフィックは ALB；データは RDS；権限は IAM Role；ネットワークは Security Group；設定は environment variable；観測は CloudWatch Logs。**

---

## 三、実施順序

0 → 7 の順で進める。各ステップを終えてから次へ進む。

### 0. 開始前の確認

`../AWS第一阶段费用与零支出原则.md` を読む。Free plan、credit、Region、本ラウンドで使うサービスが利用可能かを確認する。作成する resource、練習時間、cleanup 方法を明記する。

### 1. 手元の Image

既存 Backend で `docker build` し、Image が起動すること、手元から API を呼べることを確認する。必要な environment variable を確認する。

### 2. ECR へ push

ECR repository を作成する。Image に version tag を付けて ECR へ push する。コンソールでその tag が見えることを確認する。

### 3. ECS + ALB で起動

最小構成の ECS Cluster（Fargate）、Task Definition、Service、および ALB を作成する。Task は ECR から Image を取得する。Service に IAM Role、CloudWatch Logs、health check を設定する。ALB の URL で直接 `GET /devices` を呼ぶ（この段階の設定準備に続き、Database は次のステップで正式に接続する）。

### 4. RDS を接続

短時間・最小スペックの RDS PostgreSQL を作成する。Security Group で ECS → RDS を通す。connection string を environment variable で Task に注入する。API が RDS 上のデータを読んでいることを確認する。

### 5. 既存 Frontend を接続

Frontend の API URL を ALB のアドレスに変え、S3 / CloudFront へ公開する。Browser で CloudFront のページを開き、一覧がクラウド上の API 由来であることを確認する。CORS と HTTPS が正常であることを確認する。

### 6. deploy と rollback

API response を一箇所変更 → 再 build / 新 tag を push → ECS Service を更新 → Browser で新結果を確認 → 前の tag に戻し、rollback が効くことを確認する。

### 7. cleanup

本ラウンドで作成したものを一覧どおり停止・削除する：ALB、ECS Service / Task、RDS（snapshot の確認を含む）、ECR 上の本ラウンドの旧 Image、長すぎる log retention、残った public IP。既存の S3 / CloudFront Frontend はそのまま残す。

---

## 四、開始・終了チェック

**開始前**

1. Free plan / credit / Region / サービスの利用可否を確認済み  
2. 本ラウンドの resource 一覧、所要時間、cleanup 方法を明記済み  
3. Free plan を維持；追加支出が必要な場合は、先に判断してから作成  

**終了後**

1. 本ラウンドの ALB、ECS、RDS を停止・削除済み；snapshot を確認済み  
2. ECR の旧 Image、CloudWatch Logs、public IP / ネットワーク残留を確認済み  
3. 既存 Frontend（S3 / CloudFront）を誤って削除していない  

---

## 五、進捗

- [x] 方針の確認（2026-10-27）
- [x] 0 費用とアカウント確認（2026-10-27：費用ドキュメントに基づき短時間練習可と判断）
- [x] 1 手元の Image（2026-10-27）
- [x] 2 ECR push（2026-10-27：リポジトリ作成済み；tag `v1` を push 済み）
- [x] 3 ECS + ALB（2026-10-27：`curl ALB/` 成功；疎通後に B どおり課金項目を停止・削除）
- [ ] 4 RDS PostgreSQL（進行中：C.1/C.2 を一巡済み；2026-10-27 一時停止時に RDS インスタンス削除済み；次回は C.2 で再作成 + C.3 から）
- [ ] 5 Frontend 接続
- [ ] 6 deploy / rollback
- [ ] 7 cleanup

関連：`../AWS第一阶段费用与零支出原则.md` · `機器管理システムの開発順序と学習手順.md`（第 4 段階）

---

## 六、実操記録（振り返り用）

記録日：2026-10-27。以下は実際に実施した手順。コマンドはすべて 1 行（`\` による継続は使わない）。

### 重要な判断（本ラウンドの構成）

- 手元の `docker compose` は **api** と **db** の 2 コンテナを同時に起動し、開発を容易にする。
- 本ラウンドのクラウド側：**api イメージのみ ECR へ push**；データベースはクラウド上で RDS（ステップ 4）を使う。手元の postgres コンテナイメージは push しない。
- `docker compose build api` はバックエンドイメージのみ構築；`docker compose up -d` は 2 コンテナの起動であり、「バックエンド＋データベース」を 1 つのイメージにまとめるわけではない。

### 本ラウンドの AWS アカウントと Region

- 練習用アカウント：**`853900965180`**（コンソールで ECR 作成に使ったアカウント）。
- Region：**`ap-northeast-1`（東京）**。
- コンソール IAM 学習ユーザー：**`touzui-learning`**（既存の `AdministratorAccess`）。
- 本機に出ていた誤った CLI 主体：`393886308010` / `CLI_user_test`（`ecr:CreateRepository` なし、かつ練習用アカウントと別）→ 本ラウンドの ECR の create/push に **使えない**；`853900965180` 配下の `touzui-learning` の access key に切り替える必要がある。

### ステップ 0

- `../AWS第一阶段费用与零支出原则.md` に基づき：本ラウンドの ECR / ECS / ALB / RDS 等は短時間・小スペック・終了後 cleanup；Paid plan へはアップグレードしない。
- 「作成」を押す直前には、コンソールで Free plan / credit を自分で再確認する。

### ステップ 1：手元の Image（完了）

**確認したファイル（設定は原則未変更）：**

- 必読：`Dockerfile`、`.dockerignore`、`requirements.txt`、`compose.yaml`、`.env`
- ざっと確認：`main.py`、`alembic.ini`、`migrations/`
- 本ステップでは精読不要：`requirements_for_python+sql.txt`、`.vene/`、`learning/`、`test/`、ノート md、業務詳細ファイル等

**要点：**

- `COPY . .`：左の `.` = ビルドコンテキスト（本機の `backend`）；右の `.` = イメージ内のカレント（`WORKDIR /app`）。
- `.dockerignore` は `.vene`、`.env`、`__pycache__` 等を除外；パスワードは compose / 環境変数で注入し、イメージに入れない。
- `compose.yaml`：`api` は `build: .`；`db` は `image: postgres:17`。

**実行したコマンド（ディレクトリ：`device-dashboard/backend`）：**

```bash
docker compose build api
docker compose up -d
docker compose ps
docker images | grep -E 'device-dashboard|postgres'
curl http://127.0.0.1:8005/devices
```

**当時の結果要約：**

- `api` / `db` とも Up；`db` は healthy。
- イメージ：`device-dashboard-api:latest`（後で ECR へ push）；`postgres:17`（手元のみ）。
- `GET /devices` が機器一覧 JSON を返す（HTTP 正常）。

### ステップ 2：ECR（完了）

**実施済み：コンソールで private repository を作成**

- サービス：Amazon ECR → Create private repository
- Repository name：`device-dashboard-api`
- Image tag mutability：**Mutable**（同一 tag を上書き可；Immutable だと同名 tag を上書き不可）
- Encryption：**AES-256**（デフォルト、追加の KMS 費用なし；本ラウンドは AWS KMS を選ばない）
- 作成後に Images ページへ：当時は **No active images**（まだ push 前で正常）
- リポジトリ URI の形：`853900965180.dkr.ecr.ap-northeast-1.amazonaws.com/device-dashboard-api`

**ターミナルでの作成は失敗（対照用に残す）：**

```bash
aws ecr create-repository --repository-name device-dashboard-api --region ap-northeast-1
```

- 失敗理由：`AccessDeniedException`；当時の identity は `CLI_user_test@393886308010`、権限なしかつアカウント不一致。
- 注意：`--repository-name` はハイフン 2 つ；パラメータをターミナルの改行で分断しない。

**現在の CLI 主体の確認：**

```bash
aws sts get-caller-identity
```

**CLI の揃えと push（完了）：**

```bash
aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 853900965180.dkr.ecr.ap-northeast-1.amazonaws.com
docker tag device-dashboard-api:latest 853900965180.dkr.ecr.ap-northeast-1.amazonaws.com/device-dashboard-api:v1
docker push 853900965180.dkr.ecr.ap-northeast-1.amazonaws.com/device-dashboard-api:v1
```

**コマンドの意味メモ：**

- `get-login-password | docker login ...`：現在の AWS CLI 主体で ECR の一時パスワードを取り、Docker をそのアカウントの東京 ECR へログインさせる。
- `docker tag ローカル名 ECR完全名:v1`：再 build せず、手元イメージに ECR 向けの名前と version tag を追加するだけ。
- `docker push ...:v1`：その tag を ECR リポジトリへアップロードする。

**push 結果要約（2026-10-27）：**

- 各レイヤーが `Pushed` と表示
- `v1: digest: sha256:e4c275f4f43dd64e87260d4b4adf445ff7d694b327494e8a2a6f5128002ba0da`
- コンソール確認：ECR → `device-dashboard-api` → Images、更新後に tag **`v1`** が見えること

---

## 七、ステップ 3 コンソール手順

**実行結果（2026-10-27）：**

- Cluster：`device-dashboard-cluster`；Service：`device-dashboard-api-svc`；Task Definition：`device-dashboard-api:2`
- ALB DNS：`device-dashboard-alb-2041883229.ap-northeast-1.elb.amazonaws.com`
- 疎通：`curl http://...elb.amazonaws.com/` → `{"message":"backend in container"}`
- 失敗したこと：CloudWatch ログ `exec /usr/local/bin/python: exec format error`（手元 ARM イメージ vs Fargate x86）。修正：`docker build --platform linux/amd64 ...` 後に再 tag/push `v1`、さらに Update service → Force new deployment。
- 疎通後の費用対応（実施済み）：Desired→0；ALB・TG を削除済み。空 Cluster / ECR `v1` / SG は残してよい。

**学習メモ — 「効くまでの流れ」全体を掴む（下位の仕組みは後回し可）：**

本ステップの疎通時、1 リクエストが実際に通る関係（まずこの経路を覚える；ENI／スケジューラ／コントロールプレーン等の細部は後で補う）：

```text
手元 curl
  → ALB（パブリック DNS:80、SG: alb-sg が 80 を許可）
  → Listener が HTTP を Target Group へ転送
  → TG がヘルスチェックで「healthy」なターゲット IP:8005 を選ぶ
  → トラフィックが ECS Task 内のコンテナ api（Fargate が動かしているプロセス）へ到達
  → コンテナ内 uvicorn が `/` に応答 → JSON が同じ経路で戻る
```

各部品の役割（本ラウンドで作った名前と対応）：

| 部品 | 本ラウンドの名前 | 流れの中での役割 |
|---|---|---|
| ECR Image + tag | `.../device-dashboard-api:v1` | コンテナ内で動くコード／依存の版 |
| Task Definition | `device-dashboard-api:2` | 説明書：イメージ、CPU/メモリ、ポート、環境変数、ログ、execution role |
| Task | Service が起動した 1 つ | 説明書の実際の実行インスタンス |
| Service | `device-dashboard-api-svc` | Desired 個数を維持；落ちたら再起動；ALB/TG にぶら下げる |
| Cluster | `device-dashboard-cluster` | Task/Service の所属「場所」（Fargate ではマシンを自分で管理しない） |
| ALB | `device-dashboard-alb` | パブリック入口、80 を受ける |
| Target Group | `device-dashboard-api-tg` | Task IP を登録し、`/` でヘルスチェック、ALB 転送に使う |
| SG | `alb-sg` / `ecs-sg` | alb は外→80 を許可；ecs は alb→8005 のみ許可 |
| CloudWatch Logs | `/ecs/device-dashboard-api` | コンテナ stdout/stderr、障害調査用（例：exec format error） |

リリース時に動かす点（本ステップで一度実施済み）：イメージ変更 → 同 tag または新 tag を push →（必要なら新 Task Definition revision）→ Service 更新 / Force new deployment → 新 Task 起動、旧 Task をドレイン。

**下位の仕組み（後回し可、本ステップで一度に掘り切らなくてよい）：** Fargate がどの下位ホストへ載せるか、ENI／パブリック IP の付け方、ALB ノードとターゲット登録プロトコル、ヘルスチェック失敗後の置換方針、イメージ取得・ログ書き込みへの IAM 資格情報の注入など——障害調査が必要になったときに公式ドキュメントを参照する。

記入日：2026-10-27。  
費用の取り決め：疎通後、**存在しているだけで課金されるものを削除**（主に ALB）；それ以外は停止／Desired=0。空 Cluster、ECR `v1` は残してよい；既存の S3 / CloudFront は触らない。  
Region は右上で先に選択：**アジアパシフィック（東京）ap-northeast-1**。  
イメージ：`853900965180.dkr.ecr.ap-northeast-1.amazonaws.com/device-dashboard-api:v1`  
コンテナポート：`8005`；ヘルスチェックパス：`/`（`/devices` は使わない）。  
`/devices` はステップ 4 で RDS 接続後；本ステップの成功基準は `curl http://ALBのDNS/` が通ること。

### A.1 ECS Cluster の作成

1. 上部検索で `ECS` → **Elastic Container Service** を開く
2. 左の **Clusters**
3. **Create cluster**
4. **Cluster name**：`device-dashboard-cluster`
5. **Infrastructure** で **AWS Fargate (serverless)** にチェック
6. 他はそのまま
7. 最下部の **Create**
8. 一覧に当該 Cluster が出るまで待つ

### A.2 Task Definition の作成

**概念メモ — Task Definition とは：**

- **Task Definition（タスク定義）**：ECS 向けの「コンテナの動かし方」の説明書（テンプレート）。いま動いているプロセスそのものではない。
- 内容：どの Image、CPU/メモリ、ポート、環境変数、ログの行き先、どの IAM Role など。
- 実際に動いている 1 つを **Task** と呼ぶ；**Service** がこの定義に従って複数 Task を維持する（落ちたら再起動など）。
- 設定変更は通常新しい **revision** になる（例：`device-dashboard-api:1` → `:2`）；リリース／ロールバックは Image tag や revision の切り替えで行うことが多い。
- 手元との比喩：クラウド上の「compose／実行仕様」に近い——動かし方を定義し、Cluster/Service がどこで・何個維持するかを担う。

**2 つの Role（作成時に混同しない）：**

| 名称 | 意味 | 本ステップ |
|---|---|---|
| **Task execution role** | ECS エージェント用：ECR からイメージ取得、CloudWatch Logs への書き込み | **選択／作成する**（よくある名前 `ecsTaskExecutionRole`） |
| **Task role** | コンテナ内アプリが他の AWS サービス（例：S3）へアクセスするとき用 | **本ステップは空のまま**；`s3:GetObject` 付きの作成ウィザードには進まない |

1. 左の **Task definitions**
2. **Create new task definition**
3. **Task definition family**：`device-dashboard-api`
4. Launch type / OS：**AWS Fargate**、OS は Linux
5. **CPU**：`0.25 vCPU`；**Memory**：`0.5 GB`
6. **Task execution role**：`ecsTaskExecutionRole` があれば選択；なければ案内に従って作成してから選択。**Task role** は本ステップ空でよい
7. **Container**：
   - Name：`api`
   - Image URI：`853900965180.dkr.ecr.ap-northeast-1.amazonaws.com/device-dashboard-api:v1`
   - Essential：Yes
   - Container port：`8005`、プロトコル TCP
8. **CloudWatch Logs** を有効化（デフォルトのロググループで可）
9. **Environment variables** を追加：

| Key | Value |
|---|---|
| POSTGRES_USER | postgres |
| POSTGRES_PASSWORD | temporary-change-in-step4 |
| POSTGRES_DB | device_dashboard |
| POSTGRES_HOST | 127.0.0.1 |
| POSTGRES_PORT | 5432 |

10. Command は **カスタムしない**、alembic も **書かない**
11. **Create** → `device-dashboard-api:1` などができることを確認
12. 作成後に環境変数が 0 なら：当該 revision で **Create new revision**（フォーム版を選び、JSON は選ばない）→ コンテナ `api` → **Environment and secrets** → **Add environment variable** で 5 つの `POSTGRES_*` を補完 → Create で `:2`。Service は環境変数付きの最新 revision を使う。

**構成メモ（本ラウンドでつまずいた点）：** Apple Silicon 手元の既定ビルドは ARM イメージ；Fargate Task が `X86_64` だと `exec format error` になる。クラウドへ上げる前に：

```bash
docker build --platform linux/amd64 -t device-dashboard-api:latest .
```

その後 tag / push；既にデプロイ済みなら Update service で **Force new deployment** にチェック。

### A.3 Target Group の作成

**概念メモ — Target type の 4 選択肢それぞれ：**

Target Group 作成時の **Target type** は「ALB がトラフィックをどの種類のターゲットへ送るか」。誤りだと後で ECS/Fargate をぶら下げられない。

| Target type | 意味 | 本ラウンド |
|---|---|---|
| **Instances** | ターゲットは **EC2 インスタンス**（インスタンス単位で TG に登録） | 使わない。本ラウンドは Fargate で、自分で管理する EC2 がない |
| **IP addresses** | ターゲットは **IP アドレス**（コンテナタスクが得たプライベート IP 等） | **これを使う**。Fargate の Task は IP 形式で TG に載る |
| **Lambda function** | ターゲットは **Lambda 関数**；ALB が直接 Lambda を呼ぶ | 使わない。本ラウンドはコンテナ API であり Lambda ではない |
| **Application Load Balancer** | ターゲットは **別の ALB**（LB がさらに下位の ALB へ） | 使わない。多段 ALB 構成向け；最小実践では不要 |

本ラウンドの結論：**IP addresses** を選ぶ。

1. `EC2` を検索 → **EC2** を開く
2. 左 **Load Balancing** → **Target Groups** → **Create target group**
3. **Target type**：**IP addresses**
4. Name：`device-dashboard-api-tg`
5. Protocol：HTTP；Port：`8005`
6. VPC：デフォルト VPC（または本ラウンドで統一して使う VPC）
7. Health check path：`/`；Success codes：`200`
8. Next → **Register targets** では IP を手動追加しない → **Create target group**

### A.4 ALB の作成

**概念メモ — 作成ページの 3 種 Load Balancer の違いと関係：**

コンソール **Create load balancer** によく出る 3 枚のカードは、いずれも「後ろのターゲットへ振り分ける」が、働くネットワーク層と典型用途が異なる：

| 種類 | 正式名称 | おおよその層 | 何を見て転送するか | 典型用途 |
|---|---|---|---|---|
| **Application Load Balancer（ALB）** | アプリケーション負荷分散 | 第 7 層（HTTP/HTTPS） | URL パス、Host ヘッダ、HTTP ヘッダなど | Web / API（本ラウンド `GET /devices`） |
| **Network Load Balancer（NLB）** | ネットワーク負荷分散 | 第 4 層（TCP/UDP/TLS） | IP + ポートなど | 超高同時接続、極低遅延、非 HTTP、他 LB／サービスの手前に置く場合 |
| **Gateway Load Balancer（GWLB）** | ゲートウェイ負荷分散 | 第 3／ゲートウェイ寄り（VPC Endpoint 等と併用） | トラフィックを「検査／セキュリティ機器」系ターゲットへ送る | ファイアウォール、侵入検知などの仮想アプライアンスを経路に挿入；通常の Web API 入口ではない |

**関係（覚え方）：**

- 3 つとも **Elastic Load Balancing（ELB）** ファミリーで、複数ターゲット・ヘルスチェック・AZ またぎが可能。
- **どれを選ぶかはトラフィック種別**：ブラウザ／HTTP API → まず **ALB**；TCP ポート転送のみ、または極限性能 → 多くは **NLB**；セキュリティ機器チェーンへ通す → **GWLB**。
- 組み合わせも可能（発展）：例として外側 NLB、内側 ALB；本ラウンドの最小実践は **ALB のみ**作成し、NLB/GWLB は作らない。

**本ラウンドの結論：** **Application Load Balancer** カードの **Create** を押す。

1. EC2 → **Load Balancers** → **Create load balancer** → **Application Load Balancer** → **Create**
2. Name：`device-dashboard-alb`
3. Scheme：Internet-facing；IP：IPv4
4. VPC は TG と同じ；少なくとも **2 つの AZ** にチェックし、それぞれ **パブリックサブネット** を選ぶ
5. Security group：新規 `device-dashboard-alb-sg`  
   - インバウンド：Type HTTP、Port `80`、Source `0.0.0.0/0`  
   - ALB ウィザードに戻って当該 SG を選択
6. Listener：HTTP `80` → Forward to `device-dashboard-api-tg`
7. **Create load balancer**、**Active** になるまで待つ
8. ALB の **DNS name** をメモ帳へコピー

### A.5 ECS 用 Security Group の作成

**概念メモ — Source に Security group（`device-dashboard-alb-sg`）を選ぶ意味：**

- Security Group のインバウンド規則は **「誰が → 自分のどのポートへアクセスしてよいか」** と読める。
- **Source** が「誰」：
  - `0.0.0.0/0`：インターネット上の任意 IP がそのポートへ届く（ECS の `8005` には面が大きすぎて不安全；本ラウンドではしない）。
  - **別の Security Group**（ここでは `device-dashboard-alb-sg`）：**その SG が付いた NIC／リソース**からのトラフィックだけ許可——実質 **ALB** からのみ。
- 経路は：  
  `Browser → ALB:80（alb-sg が全世界→80 を許可）→ ECS Task:8005（ecs-sg が alb-sg のみ許可）`
- 外からは ALB の 80 にしか届かず、**コンテナ 8005 を直接スキャンできない**；ALB が ECS へ転送する。
- 「`device-dashboard-alb-sg` を選ぶ」は IP を書くのではなく、**その SG を使っている ALB の ENI をすべて信頼する**こと；ALB の拡大縮小や IP 変更でも規則を直さなくてよい。

1. EC2 → **Security Groups** → **Create security group**
2. Name：`device-dashboard-ecs-sg`；同一 VPC
3. インバウンド：Custom TCP、Port `8005`、Source にセキュリティグループ `device-dashboard-alb-sg`（全世界へ 8005 を開けない）
4. 作成

### A.6 ECS Service の作成

1. ECS → Clusters → `device-dashboard-cluster` → **Services** → **Create**
2. Compute：Fargate
3. Task definition：`device-dashboard-api`（最新 revision）
4. Service name：`device-dashboard-api-svc`；Desired tasks：`1`
5. Networking：同一 VPC；**パブリックサブネット**；SG は `device-dashboard-ecs-sg`；**Public IP：Turned on**
6. Load balancing：既存 ALB `device-dashboard-alb`；コンテナ `api` ポート 8005；Listener HTTP 80；TG は `device-dashboard-api-tg`
7. Create → Task が **Running**；TG Targets が **healthy** になるまで待つ
8. 失敗時：ECS → 当該 Task → Logs でエラーを残す

### A.7 疎通確認

```bash
curl http://あなたのALBのDNS/
```

期待例：`{"message":"backend in container"}`

### B. 疎通後の費用対応

#### B.1 Desired を 0 にする

1. ECS → Clusters → `device-dashboard-cluster` → Services → `device-dashboard-api-svc`
2. **Update service** → Desired tasks を `0` → 保存
3. Running tasks = 0 になるまで待つ

#### B.2 ALB の削除（存在しているだけで課金）

1. EC2 → Load Balancers → `device-dashboard-alb` を選択
2. **Actions** → **Delete load balancer** → 確認
3. 一覧から消えるまで待つ

#### B.3 Target Group の削除

1. EC2 → Target Groups → `device-dashboard-api-tg` を選択
2. **Actions** → **Delete**（まだ参照されている場合は ALB 削除後に再試行）

#### B.4 残す／触らない

- 残してよい：空 Cluster、Desired=0 の Service（または Service を削除して次回再作成）、ECR `v1`、2 つの SG
- 触らない：既存の S3 / CloudFront フロントエンド
- 次回ステップ 4/5 で ALB を再作成し Service を起動する

---

## 八、ステップ 4 RDS 接続（コンソール手順）

記入日：2026-10-27。  
前提：ステップ 3 は疎通済みで ALB/TG 削除済み、Service Desired=0；Cluster / Task Definition / ECR `v1` / `device-dashboard-ecs-sg` は残っている。  
費用：RDS は **存在しているだけで課金**（コンピュート＋ストレージ）；ALB 再作成後も同様。疎通後は本節 **E** で停止・削除。  
注意：RDS 作成時に **Manage master credentials in AWS Secrets Manager** が出たら **自己管理パスワード / Self managed** を選び、Secrets Manager の追加月額を避ける。

成功基準：`curl http://ALBのDNS/devices` が機器一覧 JSON を返す（データは RDS 由来）。

### C.1 RDS 用 Security Group の作成

1. EC2 または VPC → **Security Groups** → **Create security group**
2. Name：`device-dashboard-rds-sg`；VPC：ECS と同じ（デフォルト VPC）
3. インバウンド：Type **PostgreSQL** または Custom TCP、Port **`5432`**  
   Source：セキュリティグループ **`device-dashboard-ecs-sg`**（`0.0.0.0/0` は使わない）
4. 作成

### C.2 RDS PostgreSQL の作成（短時間・最小）

1. **RDS** を検索 → **Databases** → **Create database**
2. **Standard create** を選ぶ
3. Engine：**PostgreSQL**（バージョンはデフォルトで可）
4. Templates：優先 **Free tier**（アカウントに表示があれば）；なければ Dev/Test、できるだけ最小インスタンス
5. Settings：
   - DB instance identifier：`device-dashboard-db`
   - Master username：`postgres`
   - パスワード：学習用を自分で決めて手元に控える（**git に書かない／チャットに送らない**）
   - 認証情報管理：**Self managed**（Secrets Manager は使わない）
6. Instance：できるだけ小さく（例：`db.t4g.micro` / `db.t3.micro`）
7. Storage：デフォルト 20 GiB で可；自動拡張の上限を大きくしない（Storage autoscaling をオフ、またはごく小さく）
8. Connectivity：
   - VPC：ECS と同じデフォルト VPC
   - Public access：**No**
   - VPC security group：**`device-dashboard-rds-sg`**（可能なら余分な default を外す）
9. Additional：Initial database name に **`device_dashboard`**
10. 監視／拡張などオフにできるものはデフォルトのままかオフにして費用を抑える
11. **Create database** → Status = **Available** まで待つ（多くの場合 5〜10 分）
12. インスタンスを開き **Endpoint** をコピー（形：`device-dashboard-db.xxxx.ap-northeast-1.rds.amazonaws.com`）

### C.3 Target Group + ALB の再作成

ステップ 3 の A.3 / A.4 と同じ（前回削除済みのため再作成）：

1. TG：`device-dashboard-api-tg`；Target type **IP**；Port **8005**；Health check **`/`**；IP は手動登録しない
2. ALB：`device-dashboard-alb`；Internet-facing；パブリックサブネット 2 つ；SG は既存 `device-dashboard-alb-sg`（残っていれば）または A.4 どおり再作成；Listener HTTP 80 → 当該 TG
3. 新しい ALB DNS をコピー

### C.4 新しい Task Definition revision（実 DB 接続 + 起動時マイグレーション）

1. ECS → Task definitions → `device-dashboard-api` → 最新 → **Create new revision**
2. コンテナ `api` → **Environment and secrets**、変数を次のように変更（HOST は自分の Endpoint）：

| Key | Value |
|---|---|
| POSTGRES_USER | postgres |
| POSTGRES_PASSWORD | （自分の RDS パスワード） |
| POSTGRES_DB | device_dashboard |
| POSTGRES_HOST | （RDS Endpoint。`http://` もポートも付けない） |
| POSTGRES_PORT | 5432 |

3. **Docker 設定 / Command**（本ステップでは表構造のマイグレーションが必要）：shell で先に alembic、続けて起動。例として Command に（コンソール形式によりパラメータ一覧へ分割する場合あり）：

```text
sh -c python -m alembic upgrade head && python -m uvicorn main:app --host 0.0.0.0 --port 8005
```

UI が「コマンド + 引数」分かれている場合：Command = `sh`；Arguments = `-c` と上記スクリプト全体（実際に保存できる形に合わせ、保存後に JSON の command が正しいか確認）。

4. Image は引き続き `.../device-dashboard-api:v1`（**linux/amd64** 版であること）
5. **Create** → 新 revision（例：`:3`）

### C.5 Service 更新：ALB を再接続、Desired=1

1. ECS → `device-dashboard-api-svc` → **Update service**
2. Task definition は最新 revision（RDS 環境変数付き）
3. Desired tasks：`1`
4. Networking 確認：パブリックサブネット、SG=`device-dashboard-ecs-sg`、Public IP on
5. Load balancing：再作成した ALB + TG（ステップ 3 と同じ付け方）
6. **Force new deployment** にチェック → Update
7. Task Running、TG **1 Healthy** まで待つ
8. 再び Failed なら CloudWatch `/ecs/device-dashboard-api` のログを見る

### C.6 サンプルデータの投入（seed）

一覧が要るので、一度 `seed.py` を実行する（イメージ内にファイルあり）。

1. ECS → Clusters → `device-dashboard-cluster` → **Tasks** タブ → **Run new task**（または Run task）
2. Launch type：Fargate；Task definition：Service と同じ最新 revision
3. Networking：同一 VPC、パブリックサブネット、`device-dashboard-ecs-sg`、Public IP on
4. コンテナコマンドを seed のみに上書き（例）：

```text
python seed.py
```

5. 実行し、Task が止まるまで待つ；ログに `Sample device data is ready.` が出るはず
6. （一回限りの Task なので Stopped で正常）

### C.7 疎通確認

```bash
curl http://あなたのALBのDNS/
curl http://あなたのALBのDNS/devices
```

期待：`/` に message あり；`/devices` が機器入りの JSON 配列を返す。

### E. ステップ 4 疎通後の費用対応

| 操作 | 対象 |
|---|---|
| Desired → 0 | `device-dashboard-api-svc` |
| 削除 | ALB、Target Group |
| 削除 | RDS インスタンス `device-dashboard-db`；作成時に最終スナップショット保持を聞かれたら **No**；事後 RDS → Snapshots で余分なスナップショットがないことを確認 |
| 確認 | Secrets Manager secret を誤って開いていないか → あれば削除 |
| 残してよい | Cluster、ECR `v1`、SG、Task Definition |

RDS は **Stop しただけで片付け完了としない**（停止中もストレージ等の費用が残ることがある）；本ラウンド終了時は **Delete** する。
