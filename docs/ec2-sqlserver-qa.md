## SQL Server on EC2における検討事項(設計要素、顧客ヒアリング、議論のポイントになりそうな箇所をメモする)
- 構築系: https://docs.aws.amazon.com/sql-server-ec2/latest/userguide/create-sql-server-on-ec2-instance.html
- べスプラ: https://docs.aws.amazon.com/ja_jp/prescriptive-guidance/latest/sql-server-ec2-best-practices/welcome.html

1. ライセンスオプション
SQLServerライセンスを持ち込みかAWSが提供しているライセンスを使うかの検討、顧客ヒアリングが必要
AWS提供の場合は、SQLServerがインストールされたAMIを使用する
以下参照
https://docs.aws.amazon.com/sql-server-ec2/latest/userguide/sql-server-on-ec2-amis.html
持ち込む場合はいろいろな方法ありそうなので、また別途方式は検討が必要(VMインポート/エクスポートやAWSアプリケーション以降サービス等)
https://docs.aws.amazon.com/sql-server-ec2/latest/userguide/sql-server-on-ec2-licensing.html

ライセンスやエディション、インスタンスタイプ検討の際のコスト比較のインプットは以下を参照
https://docs.aws.amazon.com/prescriptive-guidance/latest/optimize-costs-microsoft-workloads/sql-server-editions.html


2. 可用性
検討対象:
- SQL HA
- SQL Server FCI
- SQL Server Always On AG

✅ Amazon EC2 での SQL Server 高可用性 (SQL HA) 概要
https://docs.aws.amazon.com/sql-server-ec2/latest/userguide/sql-high-availability.html

Amazon EC2 High Availability for SQL Server（SQL HA） は、
ライセンス込みの SQL Server を EC2 インスタンスで稼働させる際に、高可用性構成を簡単に設定できる機能 です。
この機能を使うと、スタンバイ（待機）インスタンスの SQL Server ライセンス料を削減 できます。
通常は SQL Server ライセンス料が発生しますが、スタンバイ状態のインスタンスについては Windows ライセンス料だけが課金されます。

✅ 対応する高可用性構成

SQL HA では、次の 2 つの主要な SQL Server 高可用性構成 をサポートしています。

📌 Always On Availability Groups
→ プライマリとセカンダリのレプリカを識別し、セカンダリが読み取りトラフィックを処理していない場合に SQL Server ライセンス料を無効化。

📌 Always On Failover Cluster Instances
→ アクティブ／スタンバイインスタンスを識別し、スタンバイ側の SQL Server ライセンス料を無効化。

✅ 利用にあたっての前提条件

以下の条件を満たす必要があります：

✅ クラスター内には最大 2 台の EC2 インスタンス を設定可能（ノード数）
✅ アクティブなインスタンスの vCPU は、スタンバイ側以上であること
✅ SQL Server ライセンス込みの環境でのみライセンス節約を適用可能
✅ 同一リージョン内での Multi-AZ（複数 AZ）構成のみ対応（リージョンをまたいだ構成は不可）
✅ スタンバイノードがライセンス節約を受けるためには、以下の条件を満たす必要あり

外部からのトラフィックを処理していないこと

実行中のアクティブな SQL Server ワークロードがないこと

読み取り専用セカンダリではないこと（ただし master、msdb、tempdb、model は例外）

Always On Availability Groups の場合、グループ外の独立したデータベースが存在しないこと

✅ サポートされる SQL Server バージョン：
SQL Server 2017 以降（Standard / Enterprise）
Windows Server 2019 以降

✅ PowerShell のバージョンは 5.1 以上 が必要
✅ Reserved Instances（RI）は SQL HA の対象外
→ Savings Plans の利用を推奨（RI 割引と HA ライセンス節約の併用は効率が悪い）

✅ SQL HA を利用するための準備

以下のセットアップが必要です。

SQL Server ライセンス込みの HA ワークロードが EC2 上で稼働していること
→ AWS Launch Wizard の利用で簡単に構築可能。

AWS Systems Manager Agent（SSM Agent） が 各インスタンスでインストール・起動されていること
→ SSM 経由で SQL Server の状態を取得します。

インスタンスに適切な IAM ポリシーをアタッチ
→ AWSEC2SqlHaInstancePolicy など、SSM と Secrets Manager へのアクセス権を付与。

デフォルトでは NT AUTHORITY\SYSTEM が SSM で認証に使われる
→ セキュリティポリシーで制限している場合は、Secrets Manager に別のユーザー資格情報を登録し、そのユーザーを使用。

ネットワーク接続が SSM コマンドを通せること（VPC エンドポイント等）

✅ まとめ（ポイント）

✅ SQL HA を使うと SQL Server の高可用性構成でライセンスコストを節約可能
✅ Always On の構成をそのまま活かして、スタンバイノードの SQL Server ライセンスを Windows ライセンスだけにできる
✅ 導入には EC2、SSM、IAM などの前提条件を満たす必要あり

https://docs.aws.amazon.com/sql-server-ec2/latest/userguide/sql-high-availability-how.html
✅ Amazon EC2 High Availability for SQL Server がどのように動作するか

Amazon EC2 High Availability for SQL Server（以降 SQL HA） は、SQL Server のインスタンスを常時監視し、その役割（アクティブ／スタンバイ）を判定して 自動的にライセンス適用を管理 する機能です。

✅ 仕組みの概要
✅ 1. 自動的な状態判定

SQL HA に登録された Windows + License Included（ライセンス込み） の SQL Server を動かしている EC2 インスタンス を常に監視します。

監視には AWS Systems Manager（SSM）経由のコマンド を使い、SQL Server の状態情報（メタデータ）を取得します。

取得した情報を元に、インスタンスが
✅ アクティブ（稼働中）
✅ スタンバイ（待機中）
のどちらの役割かを判定します。

✅ 2. ライセンス費用の最適化

スタンバイ状態のインスタンス があれば、SQL Server のライセンス費用ではなく Windows Server のみの料金 になります（SQL Server のライセンス料は免除されます）。

この判定は 手動操作不要 で行われ、SQL HA がインスタンスの役割を検出するだけで自動的にライセンス判定が切り替わります。

✅ 3. フェイルオーバー時の対応

フェイルオーバー発生時に、スタンバイ → アクティブに役割が変わった場合でも、SQL HA が継続的に監視しているため、役割変更を自動的に検出します。

その結果、課金対象の変更（Windows→SQL）も自動的に反映 されます。

✅ 4. 状態のモニタリング

現在の SQL HA の状態や、どのインスタンスがライセンス節約の対象になっているかを Amazon EC2 コンソール から確認できます。

✅ 結果として得られるメリット

✅ SQL Server を高可用性構成で運用しつつ
✅ スタンバイノードの SQL Server ライセンス費用を節約できる
✅ フェイルオーバーを含む役割変更もライセンス適用が自動的に更新される


構築手順
https://docs.aws.amazon.com/sql-server-ec2/latest/userguide/sql-high-availability-get-started.html

✅ Amazon EC2 High Availability for SQL Server を始める手順

「Amazon EC2 High Availability for SQL Server（SQL HA）」を使って、スタンバイインスタンスの SQL Server ライセンス費用を節約するための 初期設定手順 は以下の通りです。

✅ Step 1：SSM エージェントを準備する
✅ SSM Agent をインストール・起動

SQL HA は AWS Systems Manager（SSM） Agent を利用して、SQL Server の状態を監視します。

SSM エージェントが各 EC2 インスタンスでインストールされ、起動していること を確認します。

多くの Windows + SQL Server AMI では SSM エージェントが事前インストールされています。

動作確認には、Systems Manager コンソールや DescribeInstanceInformation API を使い、PingStatus が Online（稼働中） になっているかチェックします。

必要に応じて最新の SSM エージェントを手動でインストールしても構いません。

✅ Step 2：EC2 インスタンスに IAM ポリシーを付与する
✅ 必要な IAM 権限を付与

インスタンスに次の AWS マネージドポリシーをアタッチして、SQL HA が正常に動作するための権限を付与します：

✅ AWSEC2SqlHaInstancePolicy
→ SQL HA が SSM Run Command を実行し、インスタンスのスタンバイ状態を識別できるようにする権限です。

✅ AmazonSSMManagedInstanceCore
→ SSM の基本機能を利用するために必要な権限です。

カスタム IAM ロールでも可能ですが、最低限上記のポリシー相当の権限が含まれている必要があります。

✅ Step 3：（任意）SQL Server 資格情報を Secrets Manager に保存する
✅ SSM の認証ユーザー選択
認証ユーザのセットアップ手順は以下を参照
https://docs.aws.amazon.com/sql-server-ec2/latest/userguide/sql-high-availability-windows-user-setup.html

初期設定では、SSM は Windows の組み込みユーザー NT AUTHORITY\SYSTEM を利用して SQL Server に接続し、HA メタデータを取得します。

もし組織のセキュリティポリシーで NT AUTHORITY\SYSTEM の使用が制限されている場合は、AWS Secrets Manager に SQL Server の資格情報を保存して利用 することができます。

この場合、Secrets Manager に保存した資格情報を使うことで、SSM が SQL Server にアクセスしてメタデータを取得します。

✅ Step 4：SQL HA ライセンス節約機能を有効化する
✅ SQL HA スタンバイ検出をオンにする

Windows + SQL Server ライセンス込みインスタンスで SQL HA のスタンバイ検出を有効化 すると、スタンバイ状態であれば SQL Server ライセンス費用が適用されなくなります。

✅ 有効化の方法：
🔹 （1）EC2 コンソールで設定する場合

EC2 コンソール を開きます。

左メニューで インスタンス を選択。

高可用性構成のインスタンスを選択 → アクション > インスタンス設定 > Modify SQL High Availability settings を選択。

表示される前提条件画面で構成を確認します：

SSM Agent の状態：Online（稼働中） であること

推奨 IAM ポリシー：正しくアタッチされていること

次へ進み、SQL High Availability license savings を「Enable（有効）」にチェック。

（オプション）Secrets Manager に保存した SQL Server 資格情報を指定する場合、それも選択します。

最後に 変更を適用 して設定を完了します。

🔹 （2）AWS CLI で設定する場合

以下のコマンドで SQL HA のスタンバイ検出を有効化できます：

aws ec2 enable-instance-sql-ha-standby-detections \
  --instance-ids <インスタンスID> \
  --sql-server-credentials <Secrets Manager の ARN>


※ --sql-server-credentials は Secrets Manager に資格情報を保存した場合のみ指定します。

✅ まとめ

高可用性 SQL Server で SQL HA 機能を使うには、以下の手順で準備と設定を行う必要があります：

SSM Agent のインストールと起動

必要な IAM 権限の付与

（任意）SQL Server 資格情報を Secrets Manager に保存

SQL HA のスタンバイ検出を有効化

これらの設定を正しく行うことで、SQL Server のスタンバイノードにおけるライセンスコストを節約 しながら高可用性構成を運用できます。

3. デプロイ方式
手動構築かLaunch Wizardを利用する方法がある
またデプロイオプションとして、スタンドアロン(シンプルに単一インスタンス)か、FCI(Failover Cluster Instances)か、AG(Always On Availabilty Groups)かを選択可能で、中でもAGはデータベース高可用性および災害対策向けの構成のため要検討
これらのクラスタリング機能はWindows Serverの機能
デプロイ方式としては、Launch Wizardが推奨されており、べスプラに沿った構成を簡単に作成可能となっているので要検討

参考↓
✅ Amazon EC2 で SQL Server をデプロイする（概要）

Amazon EC2 上で Windows Server と SQL Server を組み合わせて稼働させたい場合、状況に応じて 3 つの主要な構成オプション から選択できます：

✅ 主なデプロイオプション

SQL Server 単体構成（Standalone）
EC2 上に単一の SQL Server インスタンスを構築する標準的な構成です。

SQL Server Failover Cluster Instances（FCI）
Windows クラスタリングを使って、インスタンス全体の高可用性を実現する構成です。共有ストレージが必要です。

SQL Server Always On Availability Groups（AG）
データベース単位の高可用性および災害対策向け構成。複数ノードでログを同期し、読み取りセカンダリ等の活用も可能です。

✅ 構築前の考慮事項

SQL Server を EC2 に展開する前に、次の点を検討・準備する必要があります：

✅ Amazon 提供の AMI を利用する場合

AWS が提供する Windows + SQL Server AMI を使うと、SQL Server はすでにインストール済み の状態で起動します。管理者（Administrator）権限で最初にログインして設定を行います。

✅ ストレージとパフォーマンス

インスタンスストアを持つタイプを使う場合は、tempdb をインスタンスストアボリュームに置くと パフォーマンス向上／コスト最適化 に寄与することがあります。

✅ クラスタリング機能

Windows Server のフェイルオーバークラスタ機能を有効にすることで、FCI／Always On を構成できます。FCI では共有ストレージが必要で、AG では各ノードにローカルストレージを使えます。

✅ 構築方法の選択肢
✅ AWS Launch Wizard の利用（推奨）

AWS Launch Wizard を使うと、EC2 リソース、ストレージ、ネットワークなどを自動的に設計・構築できます。

単一インスタンス / FCI / AG すべてのシナリオに対応し、ガイド付きで最適構成を構築できるため、手動構築よりミスが少なくなります。

✅ 手動構築の基本ステップ（概要）

手動で構築する場合は、次のようなステップが一般的です（詳細手順は別ページで説明されています）：

EC2 インスタンスの起動

Windows Server（2016 以降推奨）と SQL Server（対応エディション）を選択します。

ネットワーク、セキュリティグループ、IAM ロールを適切に設定します。

ストレージの構成

データファイル（MDF）、ログファイル（LDF）、バックアップファイル用ボリュームを用意します。

tempdb の配置や IOPS 設計も検討します。

Windows フェイルオーバークラスタの設定（必要な場合）

FCI や Always On AG を使う場合は、Windows Server のクラスタ機能を有効にします。

Active Directory 参加、DNS、共有ストレージなどを整備します。

SQL Server の設定

初期管理者アカウントでログインし、インスタンスの設定やセキュリティルール、バックアップ設定などを行います。

✅ 注意事項

✅ インスタンスベースで管理するため、OS・SQL Server の保守・バックアップ・パッチ適用 はすべてユーザー側で行います。
✅ RDS のようなマネージドサービスとは異なり、インフラ全体の管理責任は自分側 になります（例：セキュリティ、可用性、スケーリング）。

✅ まとめ
項目	説明
主な構成オプション	Standalone / FCI / Always On AG
推奨構築方法	AWS Launch Wizard を利用
手動構築時のポイント	ネットワーク、ストレージ、クラスタリングの事前設計
運用管理	OS・SQL の管理はユーザー側の責任

必要であれば、個別構成（Stand-alone／FCI／Always On AG）の具体的な構築手順 を

AWS Launch Wizard 編

手動構築用のステップ・スクリーンショット付き
で翻訳・解説できます。どちらが知りたいですか？（例：「Launch Wizard で構築手順を教えて」など）

https://docs.aws.amazon.com/sql-server-ec2/latest/userguide/create-sql-server-on-ec2-instance.html

以下は、AWS公式ドキュメント 「Deploy SQL Server on Amazon EC2」 の内容を わかりやすく自然な日本語で翻訳・要点整理したもの です。
（原文: Deploy SQL Server on Amazon EC2）

✅ Amazon EC2 上に SQL Server をデプロイする

Amazon EC2 上で Windows Server + Microsoft SQL Server を稼働させる方法について説明したページです。
目的や要件に応じて適切な構成を選び、セットアップ手順を進めます。

✅ デプロイメントの種類

SQL Server を EC2 上に構築するシナリオは、以下の 3 つに分類できます：

✅ SQL Server スタンドアロン構成
→ 単一インスタンスで SQL Server を稼働させる基本的な構成です。

✅ SQL Server Failover Cluster Instances (FCI)
→ Windows Server フェイルオーバークラスタリングを用いて、SQL Server インスタンス全体の高可用性を実現する構成です。共有ストレージを使用します。

✅ SQL Server Always On 可用性グループ (AG)
→ データベース単位での高可用性／災害対策向けの構成です。複数ノードでデータの同期レプリケーションを行い、読み取りセカンダリなどを活用できます。

✅ デプロイする前に検討すべきポイント

SQL Server を EC2 で起動する前に、次の点を検討・準備します。

✅ AWS 提供 AMI の利用

AWS が用意している Windows + SQL Server の AMI を使うと、SQL Server は最初からインストール済みで起動します。
一般的に ローカル管理者 (Administrator) でログインして設定を行います。

✅ ストレージ構成

インスタンスにインスタンスストア（ローカル一時ストレージ）がある場合、tempdb をインスタンスストアに配置するとパフォーマンス向上およびコスト低減に効果があります。

✅ 高可用性機能

Windows Server に標準搭載されている「Failover Clustering」機能を利用すると、

FCI（インスタンス全体の可用性）

Always On AG（データベースレベルの可用性）
といった構成が可能になります。

✅ AWS Launch Wizard の利用（推奨）

AWS は AWS Launch Wizard for SQL Server というサービスを提供しており、
構成設計、リソース選定、ネットワーク・ストレージ設定などをガイド付きで自動プロビジョニングできます。

Launch Wizard は次の構成をサポートしています：
✅ スタンドアロン
✅ SQL Server FCI
✅ SQL Server Always On AG

必要な AWS リソースを自動的にプロビジョニングするので、ベストプラクティスに沿った構成を簡単に実現できます。

✅ 部分的な手動構築（概要）

Launch Wizard を使わずに 手動で Always On AG を構築する場合の前提条件 の一例は次の通りです：

✅ 複数 AZ にまたがる 2 台の EC2 インスタンスを起動（Windows Server 2016 以降、SQL Server Enterprise 2016 以降）
✅ 各ノードに EBS ボリュームを追加してデータ／ログ／バックアップ用ストレージを構成
✅ プライベートサブネットにインスタンスを配置し、RDP で接続可能にする
✅ セキュリティグループと Windows ファイアウォールの設定（ノード間通信を許可）
✅ Active Directory に参加、WSFC 構築、ドメイン参加済みサービスアカウントで SQL Server を実行
などが必要になります。

✅ まとめ（ポイント）

✅ 利用シナリオに合わせて最適な SQL Server 構成を選べる（単体／FCI／AG）
✅ AWS Launch Wizard を活用すると、構成の設計〜デプロイまで手間が大幅に軽減される
✅ 手動構築では、Windows フェイルオーバー クラスタリングやストレージ構成などの 事前要件を満たした上で構築する必要あり

4. バックアップリカバリ
- AWS VSSソリューションによるバックアップ

AWS VSSを使用するとSQL Serverを停止することなく、EBSボリュームのバックアップを取得可能
Windows Volume Shadow Copy Service(VSS)という機能を使用
スナップショットを作成するときは、SSM Run Commandを実行

詳細↓
https://docs.aws.amazon.com/sql-server-ec2/latest/userguide/ms-ssdb-ec2-bkup-vss.html
✅ AWS VSS ソリューションによる SQL Server EC2 バックアップ

AWS VSS ソリューションを使うと、EC2 インスタンスに接続された EBS ボリュームの快適なバックアップを Microsoft SQL Server の稼働中に停止せずに 取得できます。これは Windows Volume Shadow Copy Service（VSS） を利用した方法です。

✅ VSS ベーススナップショットとは

Windows VSS（Volume Shadow Copy Service） は、アプリケーション状態を整合性のある形でスナップショットする仕組みです。

AWS VSS ソリューションでは、この VSS を使って 稼働中の SQL Server アプリケーションに影響を与えずに EBS スナップショット を作成できます。

スナップショット中に SQL Server を停止する必要はありませんし、クライアント接続を切断する必要もありません。

✅ この機能自体に追加料金はかかりません。発生するのは 作成した EBS スナップショットの料金のみ です。

✅ 動作仕組み（高レベル）

AWS Systems Manager（SSM） Run Command を使って、VSS スナップショットのプロセスを実行します。

AWS が管理する Systems Manager ドキュメント（例：AWSEC2-VssInstallAndSnapshot）を使い、必要に応じて VSS コンポーネントをインストールした後、整合性のあるスナップショットを作成します。

Windows VSS が、OS および SQL Server アプリケーションのバッファをディスクにフラッシュし、適切に一時停止した後に EBS スナップショットを開始します。

✅ スナップショットを作成するための前提条件

以下の条件を満たす必要があります：

✅ EC2 インスタンスが Systems Manager と連携できる状態（SSM Agent がインストールされているなど）であること
✅ インスタンスプロファイルに 適切な IAM 権限 が付与されていること
✅ データベースを復元可能にするために、 対象の SQL Server データベースが「完全復旧モデル（Full Recovery）」で構成されていること（スナップショットからの復元を可能にするため）

なお、推奨される Run Command ドキュメント AWSEC2-VssInstallAndSnapshot を使えば、VSS コンポーネントのインストール作業を自動的に行うことができます。これにより手動での準備作業を省略できます。

✅ スナップショットの作成方法

スナップショットを作成するには Systems Manager Run Command で以下のような操作を実行します：

✅ AWS マネジメントコンソールから SSM Run Command を起動
✅ コマンドドキュメントに AWSEC2-VssInstallAndSnapshot を選択
✅ ターゲットとなる EC2 インスタンスを指定
✅ パラメータ SaveVssMetadata=True を指定して実行
✅ EBS スナップショットが整合性ありで作成されます

✅ SaveVssMetadata=True を設定することで、VSS スナップショット実行後に復元に必要なメタデータファイルが収集できます。これがないと復元できない場合があります。

✅ スナップショットからの SQL Server 復元

作成した VSS ベースのスナップショットから SQL Server を復元するには、AWS Systems Manager の Automation Runbook を使う方法があります。

ランブック名：AWSEC2-RestoreSqlServerDatabaseWithVss

このランブックを実行すると、指定されたスナップショットセットから EBS ボリュームを作成してアタッチし、必要に応じてデータベース復元まで自動化できます。

※ 復元前の前提条件や IAM 権限設定については別ページを参照する必要があります。

✅ ポイントまとめ

✅ VSS ベースの EBS スナップショットを使うと、SQL Server を停止せずアプリケーション整合性のあるバックアップが可能です。
✅ 料金は EBS スナップショット分だけ（AWS VSS ソリューション自体の利用は追加料金なし）。
✅ スナップショットからの復元は Automation Runbook（例：AWSEC2-RestoreSqlServerDatabaseWithVss）で自動化できます。