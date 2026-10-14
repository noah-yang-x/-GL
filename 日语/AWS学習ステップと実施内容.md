# AWS学習ステップと実施内容

**出発点：** フロントエンド開発の経験あり、AWSの経験なし。  
**第1段階の目標：** Cloud Full-stack Engineerになる。
**補足：** 本計画の大きな方向性は変わりませんが、詳細は変更する場合があります。

同じデバイス管理プロジェクトに機能を段階的に追加する。各学習ステップは、進める順序を示す**実践の大きな手順**と、概念の役割・関係・依存関係を示す**知識体系**の2つで構成する。**知識体系の目的：** 現在のステップで実際に関わる知識を明確にし、実践中に自分で調べ、理解し、チェックして記録できるようにする。サービスと技術分野ごとに整理し、公式用語または一般的な用語を使う。関連する概念は個別に列挙し、その関係を説明する。具体的な操作方法は自分で調べ、探索する。

実践の完了と知識の理解は別々にチェックする。子項目が目標を満たしたら、対応するステップを手動でチェックし、学習記録を追記する。学習のたびに、現在の機能とリクエストの経路に立ち戻る。

## 第1段階：6層の学習ルート

> - [ ] **主線：Device Listを動かす**
>     - [ ] **静的プロジェクトをAWSに公開する**
>         - **実践の大きな手順（上から順に）**
>             - [x] ローカルで動く静的なデバイス一覧を用意し、フロントエンドのビルド成果物を生成する
>             - [x] デプロイ先のリージョンを選び、ビルド成果物を非公開のS3に保存する
>             - [x] S3をオリジン(orgin)とするCloudFrontディストリビューション(distribution)を作成し、オリジンへのアクセス権限とページの入口を設定する
>             - [x] ブラウザから公開ページにアクセスし、ページとリソースのリクエストに対する応答経路を確認する
>             - [x] ページを変更して再公開し、ファイル更新とキャッシュの関係を体験する
>         - **知識体系（このステップで実際に関（かか）わる知識）**
>             - **フロントエンド · ビルドとデプロイ**
>                 - [x] 静的ビルド成果物（せいかぶつ）：HTML、JS、CSSとリソースのパス（path）
>                 - [x] SPAルーティング：ページ内の遷移（せんい）とパスへの直接アクセスの違い（プロジェクトで使用する場合）
>             - **AWS · Region**
>                 - [x] Region（リージョン）：デプロイ先の選択とリージョン単位のリソース
> 	                - [Later] リージョン単位のリソース
>             - **AWS · Amazon S3**
>                 - [x] Bucket（バケット）：ビルドファイルを保存する入れ物と所属リージョン
>                 - [x] Object / Object Key：ファイルの内容、オブジェクト名、アクセスパス
>                     - [note] オブジェクトの完全なKeyを確認でき、ファイルパスとの関係を理解している
>                     - [note] **Keyは大（おお）文字と小（こ）文字を区別する**：`app.js`と`App.js`は別のオブジェクト。
>                     - [note] **同じBucket内では、完全なKeyでオブジェクトを識別（しきべつ）する**：`assets/app.js`と`backup/app.js`は同時に存在できる。
>                 - [x] オブジェクトのアップロードと上書き（うわがき）：ローカルのビルド成果物をS3に反映する方法
>                 - [x] Content-Type：HTML、JS、CSSの（response）の種類 - - - [note] これはブラウザの話では？ なぜここにあるの？
>                     - [note] S3はファイルの内容に加えて、Content-Typeをオブジェクトのメタデータとして保存し、オブジェクトを読み取る際のHTTPレスポンスに使用する。具体的には、S3 → 対象のBucket → Objects（オブジェクト）→ `index.html`をクリック → Properties（プロパティ）→ System and user-defined（システム定義およびユーザー定義のメタデータ）まで下に移動 → `Content-Type`行のValueを確認する。これは、ブラウザがそのObjectをリクエストした際にResponse Headersで確認できるContent-Typeに当たる。一言で言えば、Content-Typeは返されたコンテンツの種類をブラウザに伝える。
>                 - [x] Block Public Access：バケットとオブジェクトへの直接の公開アクセスを防ぐ - - - [note] 詳しくは確認していないが、デフォルトで選択されており、現在のS3 Bucket内のコンテンツを非公開に保てることは理解している。つまり、AWSサービス外部からのリンクでも、権限がなければアクセスできない
>             - **AWS · IAMとS3のアクセス権限**
>                 - [x] IAM Principal：作成・アップロード・設定を実行する主体 
>                     - [note] この文の意味は、これまでAWSを操作した際に、どの主体として操作したのか、そしてなぜその操作が許可されたのか、ということ。
>                     - [note] 回答：touzui-learningというIAM userで操作した。このIAM user用のPolicyを作成する際に最大の権限を付与したため、操作が許可された。また、`aws sts get-caller-identity`コマンドで確認でき、返されたArnフィールドに情報がある：`"Arn": "arn:aws:iam::853900965180:user/touzui-learning"`。
>                     - [note] このIAMユーザーが<u>Principal</u>であり、そのユーザーに権限を付与する<u>Policy</u>は、AWSがアップロードを許可するか判断する根拠の1つ。
>                 - [x] IAM Policy：Action、Resource、Effectと操作権限 - - - - [note] 最大の権限を付与したが、この部分はまだ深く理解していない❄️❄️❄️
>                     - [ ] Policy、Statement、Action、Resource、Effectを説明できる。
>                     - [ ] ListBucket、GetObject、PutObject、DeleteObjectの違いを説明できる。
>                     - [ ] PolicyがどのUser / Roleに適用されるか確認できる。
>                     - [ ] デフォルトでは拒否され、Explicit Denyが優先されることを理解している。
>                     - [ ] 実際に使用したAdministratorAccessとIAMUserChangePasswordを使って、3つのフィールドと2つのPolicyの関係を説明できる。
>                 - [x] S3 Bucket Policy：指定したCloudFrontディストリビューションにオブジェクトの読み取りを許可する - - - - - - [note] 現在はAWSでCloudFrontを作成する際にデフォルトで直接設定でき、手動操作は不要❄️❄️❄️
>             - **AWS · Amazon CloudFront**
>                 - [x] Distribution：Webサイトの配信設定とアクセス用ドメイン名
>                 - [x] Origin：オリジンとなるS3バケット
>                 - [ ] Origin Access Control（OAC）：CloudFrontから非公開S3へのオリジンリクエストの認証
>                 - [x] Default Root Object：ルートパスへのリクエストで使用する入口ファイル - - - [note] 任意の設定だが、確認済みのトップページのファイル名を指定すると、アクセス時にそのファイル名を入力する必要がなくなり便利
>                 - [ ] Viewer Protocol Policy：ブラウザからのアクセスに対するHTTP / HTTPSのポリシー
>                 - [x] Edge Locations（エッジロケーション）：ユーザーのリクエストを受け取り、コンテンツをキャッシュする。CloudFrontが管理し、手動で作成する必要はない - - - [note] 関連する概念を理解した
>                 - [x] Cache Behavior：リクエストパスに対応するオリジンとキャッシュの設定
>                     - [note] 対応するドキュメントを作成した
>                 - [ ] Cache Policy / TTL：キャッシュキーとキャッシュの有効期間
>                 - [x] Cache Hit / Miss：キャッシュヒットと、ミス時のオリジンからの取得 - - - [note] 概念として学習した。手動でデータを更新した際にも、この問題を実際に理解できた
>                 - [x] Invalidation：キャッシュを削除し、更新後のファイルを再取得させる - - - [note] ローカルのターミナルからCloudFrontのInvalidationを実行し、AWS側でも対応する記録を確認した。
>                 - [x] ファイル名のバージョニング：新しいリソースパスで新しいファイルを公開する方法と、キャッシュ無効化との違い
>                     - [note] **ファイル名のバージョニング**：ファイルを更新しても古いキャッシュが読み込まれることを避けるため、バージョンごとに変わるバージョン番号やHashをファイル名に含め、ページ側の参照も更新する。ブラウザは新しいURLで新しいファイルを取得するため、古いキャッシュの期限切れを待つ必要がない。
>                     - [note] **現在のプロジェクトでの実装**：Vite BuildがHash付きのJS、CSSファイル名を自動生成し、`index.html`内の参照も更新する。
>                     - [note] **違い**：ファイル名のバージョニングはURLを変更して古いキャッシュを回避する。Invalidationは元のURLの古いCloudFrontキャッシュを無効化する。一方、S3 Versioningは同じObject Keyの下にファイルの複数のバージョンを保持し、古い内容を復元しやすくするが、CloudFrontキャッシュを自動更新するものではない。
>                         - [Later] S3 Versioningの実装方法と対応する実践❄️❄️❄️
>                     - [note] **公開時の組み合わせ**：JS、CSSには新しいURLを使う。ファイル名が変わらない`index.html`は、ブラウザが新しいリソースをリクエストできるよう、速やかにキャッシュを更新する必要がある。
>                 - [ ] [Later] Custom Error Responses：SPAのルートに直接アクセスした場合の処理（プロジェクトで使用する場合） - - - まだ遭遇していないため、遭遇してから学習する❄️❄️❄️
>             - **ネットワーク · DNS、HTTPS、HTTP**
>                 - [ ] DNS：CloudFrontのドメイン名解決とリクエストのルーティング - - - [note] 概念として学習し、ドキュメントに整理した
>                 - [ ] HTTPS / TLS：ブラウザからCloudFrontへの暗号化された接続
>                 - [ ] HTTP GET：入口ファイルと静的リソースへのリクエスト
>                 - [ ] HTTP Status Codes：200、403、404などの結果と原因調査
>                 - [ ] Cache-Control：レスポンスヘッダーのキャッシュ指示
>             - **ブラウザ · リソース読み込みとキャッシュ**
>                 - [ ] HTML解析とリソース読み込み：ページがJS、CSS、画像を取得する仕組み - - - AWSとは関係なく、もともと理解しているため、調べていない
>                 - [ ] ブラウザキャッシュ：CloudFrontキャッシュとの違いと、ページ更新への影響
>                 - [x] DevTools Network：URL、ステータスコード、レスポンスヘッダー、キャッシュの取得元を確認する
>         - **関係：** ブラウザ → CloudFront。キャッシュヒットなら直接返し、ミスならS3から取得して返す。
>         - **到達目標：** 公開ページにアクセスでき、更新が反映される。ファイル保存、アクセス、認可、キャッシュの関係を説明できる。
>         - **学習記録：**
>     - [ ] **既存ページをAPIに接続する**
>         - **実践の大きな手順（上から順に）**
>             - [ ] Lambdaでデバイス一覧のダミーデータを返し、最小限のバックエンド処理を作る
>             - [ ] API GatewayのルートとLambda統合を作成し、APIのリクエストとレスポンスをつなぐ
>             - [ ] 既存ページのデータ取得元をAPIに置き換え、実際に発生したクロスオリジンやレスポンスの問題に対応する
>             - [ ] 成功したリクエストと失敗したリクエストを1つずつ追跡し、対応するレスポンスとログを確認する
>         - **知識体系（対応する実践を通じて理解する）**
>             - [ ] AWS · Amazon API Gateway：HTTPメソッド / パス → API Gateway Route → Integration → Lambda
>             - [ ] AWS · AWS Lambda：Runtime / Handler、Event入力、業務処理、JSON / ステータスコードの出力
>             - [ ] AWS · Lambda Permissions / Execution Role：API GatewayがLambdaを呼び出す権限と、Lambda実行ロールの違い
>             - [ ] Web · Origin / CORS：ページのOrigin、APIアドレス、CORS、プリフライトリクエスト
>             - [ ] AWS / Web · Lambdaログ / HTTPレスポンス：ブラウザのリクエスト結果、APIレスポンス、Lambda実行ログの対応
>         - **関係：** ブラウザ → API Gateway → Lambda → API Gateway → ブラウザ。
>         - **到達目標：** APIが返すダミーデータをページに表示できる。リクエストがバックエンドに入り、応答が戻る経路と、失敗時の調査先を説明できる。
>         - **学習記録：**
>     - [ ] **APIをデータベースに接続する**
>         - **実践の大きな手順（上から順に）**
>             - [ ] デバイスの識別子と検索要件からデータ構造を決め、DynamoDBテーブルを作成する
>             - [ ] AWS SDKと必要な権限を使って、Lambdaからテーブルにアクセスする
>             - [ ] Lambdaのダミーデータをデータベースの検索結果に置き換え、既存ページにつなぐ
>             - [ ] 保存データを変更して再検索し、データベース、API、ページの対応を確認する
>         - **知識体系（対応する実践を通じて理解する）**
>             - [ ] AWS · DynamoDB Tables / Items / Attributes / Primary Key：Table → Item → Attribute。Partition Key / Sort Keyと検索要件の関係
>             - [ ] AWS · AWS SDK / Lambda Execution Role：Lambda → AWS SDK → DynamoDB API。実行ロールがアクセス可能なテーブルと操作を決める
>             - [ ] AWS · DynamoDB GetItem / Query / Scan：GetItem / Query / Scanの条件、違い、ページネーション、整合性
>             - [ ] AWS · DynamoDB Provisioned / On-demand Capacity：プロビジョンド / オンデマンド容量、読み書きの負荷、スロットリング
>             - [ ] バックエンド / Web · JSON / HTTP Response：データベースの結果 → バックエンド処理 → JSON → ページのデータ
>         - **関係：** 既存APIの経路にLambda → DynamoDB → Lambdaを追加し、リクエストの往復経路を完成させる。
>         - **到達目標：** データベースのレコードをページに表示できる。データ構造、検索方法、権限、返却結果の関係を説明できる。
>         - **学習記録：**


> - [ ] **主線の拡張：Device CRUD**
>     - [ ] **デバイス機能を1つずつ拡張する**
>         - **実践の大きな手順（上から順に）**
>             - [ ] デバイス詳細をつなぐ：ページ → GET /devices/{id} → データベース読み取り → ページ
>             - [ ] デバイス作成をつなぐ：フォーム → POST /devices → 検証と書き込み → 一覧
>             - [ ] デバイス変更をつなぐ：編集 → PUT /devices/{id} → 更新 → ページ
>             - [ ] デバイス削除をつなぐ：操作 → DELETE /devices/{id} → 削除 → 一覧
>             - [ ] 各機能に合わせて、不正な入力、存在しないデバイス、処理失敗へのフィードバックを追加する
>         - **知識体系（対応する実践を通じて理解する）**
>             - [ ] Web · HTTP Methods / URL / Request / Response：HTTPメソッド、リソースパス、パスパラメータ、リクエストボディ、レスポンス
>             - [ ] バックエンド · ルーティング / 入力検証 / 業務ロジック：ルート → パラメータ → 検証 → 業務ロジック → データベース → レスポンス
>             - [ ] AWS · DynamoDB Read / Write Operations / Read Consistency：DynamoDBの読み取り、書き込み、更新、削除、データ整合性
>             - [ ] Web / バックエンド · HTTP Status Codes / エラー処理：エラー原因 → ステータスコード / JSON → ページへのフィードバック。ログで処理箇所を特定する
>         - **関係：** 毎回1つの機能を拡張し、ページからデータベースを経てページに戻る経路を完成させる。
>         - **到達目標：** 5種類のデバイス操作を実行できる。対応するデータの変化と、エラーがページに戻る仕組みを説明できる。
>         - **学習記録：**


> - [ ] **支線：ログイン、ファイル、非同期処理、イベント**
>     - [ ] **ユーザーログインと権限**
>         - **実践の大きな手順（上から順に）**
>             - [ ] 既存ページにCognitoの登録、ログイン、ログアウトを追加する
>             - [ ] 認証済みのユーザー情報をAPIリクエストで渡し、入口で検証する
>             - [ ] 管理者と一般ユーザーの操作権限を分け、許可と拒否の両方を体験する
>         - **知識体系（対応する実践を通じて理解する）**
>             - [ ] AWS · Amazon Cognito User Pools：Cognito User Pool、ユーザー、認証、ログイン状態
>             - [ ] Web · JWT / Authentication Tokens：JWT / Tokenの発行、送信、有効期限、更新
>             - [ ] AWS / バックエンド · API Gateway Authorizer / ロールによる認可：Authorizerが本人確認を行い、バックエンドがロールに応じて操作を許可する
>         - **関係：** CognitoがTokenを発行 → ページがTokenを送信 → APIで認証 → バックエンドで権限判定。
>         - **到達目標：** ユーザーの認証情報で既存APIを保護できる。認証、トークン、業務上の権限の役割を説明できる。
>         - **学習記録：**
>     - [ ] **デバイスのファイル**
>         - **実践の大きな手順（上から順に）**
>             - [ ] 既存デバイスに画像、説明書、設定ファイルなどの保存機能を追加する
>             - [ ] フロントエンドのアップロード、読み取り、デバイスレコードとの関連付けをつなぐ
>             - [ ] 一時的なアクセス権限、リンクの期限切れ、ファイル更新を体験する
>         - **知識体系（対応する実践を通じて理解する）**
>             - [ ] AWS · Amazon S3：S3にファイルを保存し、デバイスレコードにファイルの関連情報を保存する。Webサイトのビルド成果物とは用途が異なる
>             - [ ] AWS · S3 Presigned URLs：バックエンドで認可 → 署名付きURL → 操作と有効期間の制限
>             - [ ] Web / AWS · CORS / S3オブジェクトアクセス / CloudFront：フロントエンドのOrigin / CORS、非公開オブジェクトへのアクセス、CloudFrontのファイル配信とキャッシュ
>         - **関係：** デバイスレコードをS3 Objectに関連付け、認可後にページからファイルのURLでアップロードまたは読み取りを行う。
>         - **到達目標：** デバイスにファイルを関連付けてアクセスできる。ファイル保存、業務上の関連付け、アクセス認可の関係を説明できる。
>         - **学習記録：**
>     - [ ] **レポートの非同期生成**
>         - **実践の大きな手順（上から順に）**
>             - [ ] デバイス作成後のレポート生成を同期リクエストから分離する
>             - [ ] SQS経由でWorker Lambdaに処理を渡し、先に応答が返り、後からレポートが完成することを確認する
>             - [ ] 失敗時の再試行、重複メッセージ、DLQへの移動と処理を体験する
>         - **知識体系（対応する実践を通じて理解する）**
>             - [ ] バックエンド · 同期 / 非同期処理：同期レスポンスとバックグラウンドタスクの時間的な関係
>             - [ ] AWS · Amazon SQS：プロデューサー → SQS → コンシューマー。メッセージの送信、受信、削除
>             - [ ] AWS · Lambda SQS Event Source Mapping：キューとLambdaの処理トリガー、バッチ処理
>             - [ ] AWS / バックエンド · SQS Visibility Timeout / DLQ / 冪等性：Visibility Timeout → Retry → DLQ。重複配信と冪等性
>         - **関係：** デバイス作成リクエストを先に完了し、キューに入ったメッセージをWorkerが独立して処理してレポートを生成する。
>         - **到達目標：** ユーザーがレポート生成を待たずに済む。タスクの処理、再試行、追跡方法を説明できる。
>         - **学習記録：**
>     - [ ] **デバイスのイベント**
>         - **実践の大きな手順（上から順に）**
>             - [ ] デバイス削除後に業務イベントを発行する
>             - [ ] EventBridgeでイベントを照合し、監査または通知処理を起動する
>             - [ ] 配信結果と失敗時の処理を確認する
>         - **知識体系（対応する実践を通じて理解する）**
>             - [ ] AWS · EventBridge Events / Event Buses：イベントの発生元、種類、データ、Event Bus
>             - [ ] AWS · EventBridge Rules / Event Patterns / Targets：Rule / Event Pattern → Target
>             - [ ] AWS / バックエンド · EventBridgeの配信 / 再試行 / イベント処理：リクエスト処理と後続イベント処理の分離、配信失敗と再試行
>         - **関係：** デバイス削除 → 業務イベント → ルールの照合 → 後続処理。
>         - **到達目標：** 削除イベントから対応する処理を起動できる。イベントルーティングとタスクキューの用途の違いを説明できる。
>         - **学習記録：**


> - [ ] **主線の強化：セキュリティ、ネットワーク、監視**
>     - [ ] **監視と原因調査**
>         - **実践の大きな手順（上から順に）**
>             - [ ] 実際のエラーや遅いリクエストをもとに、入口、Lambda、データベースの観測結果を関連付ける
>             - [ ] ログと保持期間を整理し、運用上の問題に合わせてメトリクスとアラームを追加する
>             - [ ] 異常の現象から具体的な箇所を特定する調査方法を作る
>         - **知識体系（対応する実践を通じて理解する）**
>             - [ ] AWS · CloudWatch Logs：CloudWatch Log Group / Stream、実行記録、保持期間
>             - [ ] AWS · CloudWatch Metrics：リクエスト数、エラー、レイテンシ、リソースの状態
>             - [ ] AWS / バックエンド · CloudWatch Alarms / リクエスト追跡：しきい値 → Alarm。リクエストの関連付け → 問題箇所の特定
>         - **関係：** 実行動作から観測データが発生 → 異常の検知 → リクエスト追跡 → 箇所の特定。
>         - **到達目標：** 実際の問題を発見して特定でき、ログ、メトリクス、アラーム、追跡がそれぞれ何を示すか説明できる。
>         - **学習記録：**
>     - [ ] **セキュリティ境界**
>         - **実践の大きな手順（上から順に）**
>             - [ ] ユーザー、開発者、サービス間の既存のアクセス関係を整理する
>             - [ ] 最小権限、機密設定、ファイルアクセスの保護を改善する
>             - [ ] 適用可能な入口で、防御ルールによる許可とブロックを体験する
>         - **知識体系（対応する実践を通じて理解する）**
>             - [ ] AWS · IAM Principals / Policies / Roles：IAM Principal / Policy / Role、リソースと操作、最小権限
>             - [ ] AWS · Secrets Manager / AWS KMS：Secrets Managerは機密設定を保存し、KMSは暗号化キーとその使用権限を管理する
>             - [ ] Web / AWS · HTTPS / S3 / CloudFront OAC / AWS WAF：HTTPS、S3 Private / OAC、WAF Web ACL / Rule
>         - **関係：** 主体の認証情報で許可される操作が決まり、通信、保存、リクエストの入口をそれぞれ保護する。
>         - **到達目標：** 誰が何にアクセスできるか、機密情報をどう保護するか、どこでリクエストを制限するか説明できる。
>         - **学習記録：**
>     - [ ] **ネットワークの関係**
>         - **実践の大きな手順（上から順に）**
>             - [ ] 現在のリソースの配置と実際のアクセス経路を先に明示する
>             - [ ] システムの必要性に合わせてVPC、サブネット、ルート、アクセスルールを計画または改善する
>             - [ ] 関連リソースを使う際に、許可、拒否、外向き通信を体験し、その理由を記録する
>         - **知識体系（対応する実践を通じて理解する）**
>             - [ ] AWS · Amazon VPC / マネージドサービス：AWSマネージドサービスと自分のVPC内のリソースの違い
>             - [ ] AWS · VPC / Subnets：VPC → Public / Private Subnet。公開アクセスの可否はルートやアドレスなどの設定で決まる
>             - [ ] AWS · Route Tables / Internet Gateway / NAT Gateway：Route Table → Internet Gateway / NAT。どの接続に必要か、不要か
>             - [ ] AWS · Security Groups / IAM Permissions：Security Groupとサービス権限は異なる層のアクセスを制御する
>         - **関係：** リソースの配置 + ルート + アドレス + アクセスルールにより、接続の可否が決まる。
>         - **到達目標：** バックエンドがデータベースとインターネットにどうアクセスするか説明でき、実際の必要性に応じてネットワーク構成要素を使える。
>         - **学習記録：**


> - [ ] **アーキテクチャ拡張：2つ目のBackendを作る**
>     - [ ] **機能を維持し、バックエンド構成を置き換える**
>         - **実践の大きな手順（上から順に）**
>             - [ ] Node.jsとPostgreSQLで既存のデバイスCRUDをまずローカルで動かす
>             - [ ] バックエンドをコンテナ化し、クラウドのネットワーク、権限、データベース接続を明確にする
>             - [ ] RDS、ECS/Fargate、ALBを接続し、同じページからのリクエストを通す
>             - [ ] デプロイ、ヘルスチェック、スケーリングを確認し、元のServerless構成と比較する
>         - **知識体系（対応する実践を通じて理解する）**
>             - [ ] バックエンド / AWS · Node.js / コンテナ / Lambda：Lambda Functionと常時稼働するNode.jsサービス、コンテナイメージと設定
>             - [ ] AWS · Amazon ECS / AWS Fargate：ECS Cluster → Task Definition → Task / Service。Fargateが実行リソースを提供する
>             - [ ] AWS · ALB Listeners / Target Groups / Health Checks：ALB Listener → Target Group → Health Check → バックエンドインスタンス
>             - [ ] AWS / データベース · Amazon RDS / PostgreSQL：RDS / PostgreSQL、SQLとリレーショナルモデル、接続、ストレージ、権限
>             - [ ] システム構成 · API Gateway / ALB / Lambda / Fargate / DynamoDB / RDS：入口、コンピューティング、データベース、スケーリング、費用の相互関係
>         - **関係：** ページ → ALB → ECS/Fargate上のNode.js → RDS → バックエンド → ページ。
>         - **到達目標：** 同じデバイス機能を2つ目のバックエンドで動かせる。2つの構成の選択理由を説明できる。
>         - **学習記録：**


> - [ ] **専門化：エンジニアリングの実践**
>     - [ ] **インフラをコード化する**
>         - **実践の大きな手順（上から順に）**
>             - [ ] 理解済みのリソースと依存関係をAWS CDKのコードで表現する
>             - [ ] デプロイ、更新、再構築、今回の練習用リソースの削除を体験する
>         - **知識体系（対応する実践を通じて理解する）**
>             - [ ] AWS / 開発 · AWS CDK / Infrastructure as Code：IaC / CDKのリソース、依存関係、パラメータ、環境設定
>             - [ ] AWS / 開発 · デプロイ / 更新 / リソース削除：コード定義 → デプロイ / 更新 → リソースの状態。保持と削除の違い
>         - **関係：** 理解済みのリソース設定 → コードによる表現 → 再現可能な環境構築。
>         - **到達目標：** コードから環境を再構築し、リソース間の依存関係を説明できる。
>         - **学習記録：**
>     - [ ] **デリバリーの自動化**
>         - **実践の大きな手順（上から順に）**
>             - [ ] 既存のGit、テスト、ビルド、デプロイ操作を整理する
>             - [ ] CI/CDを導入し、コード変更からデリバリーまでを体験する
>         - **知識体系（対応する実践を通じて理解する）**
>             - [ ] 開発 · CI/CD：Source → Test → Build → Deploy
>             - [ ] 開発 · パイプラインのトリガー / ビルド成果物 / 実行権限：トリガー、成果物の受け渡し、環境設定、実行権限
>         - **関係：** コード変更 → 自動デリバリー → 既存システムの更新。
>         - **到達目標：** デリバリーを再現でき、失敗した箇所を特定できる。
>         - **学習記録：**
>     - [ ] **複数環境と復旧**
>         - **実践の大きな手順（上から順に）**
>             - [ ] dev、staging、productionの設定とリソースを分け、必要に応じて稼働させる
>             - [ ] 既存システムでバージョンのロールバックとデータ復旧を体験する
>             - [ ] 運用監視、アラーム、監査記録を整理する
>         - **知識体系（対応する実践を通じて理解する）**
>             - [ ] 開発 · 環境設定 / リソース分離：コード、設定、リソース分離の関係
>             - [ ] 開発 / データベース · Rollback / Backup / Restore：アプリケーションのバージョンロールバックと、データのBackup / Restoreの違い
>             - [ ] 運用 · Monitoring / Alarm / Audit：Monitoring → Alarm → 調査 / 復旧。Auditは操作の追跡を支える
>         - **関係：** 指定環境へデリバリー → 稼働確認 → 問題発見 → 復旧と追跡。
>         - **到達目標：** 環境の違いを説明し、バージョンやデータを復旧し、操作を追跡できる。
>         - **学習記録：**


## 静的公開の参考資料

[AWS公式：S3 → CloudFrontの公開手順](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/GettingStarted.SimpleDistribution.html) · [AWS公式：OACと非公開S3へのアクセス](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)

## 用語の参考資料

- 静的公開：[CloudFront入門](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/GettingStarted.SimpleDistribution.html) · [S3 Bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingBucket.html) · [エッジ配信とキャッシュ](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/HowCloudFrontWorks.html) · [OAC](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html) · [キャッシュ無効化](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Invalidation.html)

- AWS：[S3の基本概念](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html) · [CloudFront Distributions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/distribution-working-with.html) · [IAM Policies and Permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html)
- AWS：[API Gateway HTTP APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-develop.html) · [DynamoDB Core Components](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html) · [Cognito User Pools](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools.html) · [EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html)
- Web：[MDN HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Overview)

## 第2段階：IoT Cloud

> - [ ] **第2段階：IoT Cloud**
>   - [ ] **実機 / シミュレーションデバイス**
>       - [ ] デバイス識別子、送信データの形式、データの生成方法
>           - **達成条件：** 繰り返し利用できるデバイスデータの取得元がある
>   - [ ] **MQTT**
>       - [ ] 接続、Topic、パブリッシュ、サブスクライブ
>           - **達成条件：** デバイスがどのTopicで何のデータを送るか説明できる
>   - [ ] **AWS IoT Core**
>       - [ ] デバイス接続、認証とアクセス権限、メッセージ受信
>           - **達成条件：** 認可済みデバイスのデータをクラウドで受信できる
>   - [ ] **既存のCloud Backend**
>       - [ ] デバイスメッセージをデバイスレコードに関連付け、既存のデータ処理と保存につなぐ
>           - **達成条件：** デバイスデータが第1段階の既存バックエンドに入る
>   - [ ] **既存のWeb**
>       - [ ] 受信データでデバイス情報と表示を更新する
>           - **達成条件：** 元のプラットフォームで実機またはシミュレーションデバイスのデータを確認できる

## 実行方針

- 現在のステップを整えてから次へ拡張し、同じプロジェクトを継続して発展させる。
- 支出をできる限り抑える。有料サービスの使用前に費用を説明し、継続課金リソースは練習に必要な期間だけ稼働させる。
- 6か月の学習期限は設けず、実際の学習ペースで進める。

学習ルート：[[AWS学習ロードマップ]]。
