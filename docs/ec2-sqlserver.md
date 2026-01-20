## 公式ドキュメント
https://docs.aws.amazon.com/ja_jp/prescriptive-guidance/latest/sql-server-ec2-best-practices/welcome.html

## 構築参考
https://repost.aws/ja/knowledge-center/ec2-windows-launch-sql-server
https://qiita.com/ohtsuka-shota/items/ae36b326778b1f9c5252

## SQL Server構築手順

※SQL Serverが既にインストールされたAMIもあるが、ライセンス料やインスタンスタイプの指定がありコスト高のため検証では以下の手順を採用


1. スタック送信し、EC2作成

2. EC2にSSMで接続し、コマンドラインで以下のコマンドでRDP用の管理者ユーザのPWを設定
`net user Administrator Password123!`

3. EC2>接続>RDP>FleetManagerからRDP接続する際に2のPWで認証
ユーザ: Administrator
PW: 2のPW

4. 以下のURLにアクセスしてSQL Serverをインストール
https://www.microsoft.com/ja-jp/sql-server/sql-server-downloads

参考は以下の記事
https://qiita.com/ohtsuka-shota/items/ae36b326778b1f9c5252

SQL ServerのDeveloperをインストール後、SSMSをインストールしようとしたところ、NWエラーとなった
以下コマンドでデバッグするも、原因わからず、試しにSGのアウトバウンドで80ポートの許可を追加したら解消した
`Test-NetConnection -ComputerName www.microsoft.com -Port 443`
`Test-NetConnection -ComputerName aka.ms -Port 443`

NATに通信が向いているか
`Get-NetRoute -DestinationPrefix 0.0.0.0/0 | Format-Table -AutoSize`

5. 上記インストールが完了したら、Services.mscを開いて、SQL Server（MSSQLSERVER）がRunningになっていることを確認

6. 参考サイト通りにSSMSからDatabaseやTableの作成ができることを確認

## EC2上のSQL Serverへの接続
https://docs.aws.amazon.com/sql-server-ec2/latest/userguide/connect-sql-server-on-ec2-instance.html
✅ Amazon EC2 上の SQL Server へ接続する方法

AWS の Windows + SQL Server AMI から起動した EC2 インスタンス上の SQL Server に接続するには、主に SQL Server Management Studio (SSMS) を使います。EC2 側では RDP を使って管理者としてログインし、その後 SSMS で SQL Server に接続します。

✅ 1. SQL Server Management Studio (SSMS) を使って接続する
✅ 初期ログイン（管理者）

AWS 提供の Windows AMI で起動した SQL Server インスタンスでは、初期状態ではローカルの管理者アカウントのみが SQL Server に接続可能です。

手順：

RDP で EC2 に接続（Windows 管理者権限）

管理者ユーザーでログインします。

SQL Server Management Studio (SSMS) を起動

必要に応じて SSMS をインストールしてください。

SSMS の接続画面で認証を設定

Authentication：Windows Authentication

管理者ユーザーで接続します。

✅ 2. ドメインユーザーでログインできるようにする（任意）

EC2 上の SQL Server に Active Directory（AD）ユーザーでログインしたい場合は、SSMS からドメインユーザーのログインを追加します。

✅ 手順

SSMS で管理者として接続

オブジェクトエクスプローラーで Security → Logins を展開

右クリックして New Login を選択

Login name で Windows 認証を選び、
DomainName\username の形式でドメインユーザーを指定

Server roles タブで必要なサーバーロールを付与

OK をクリックしてログインを作成

一度 EC2 からログアウトし、ドメインユーザーで再ログイン

SSMS を起動して Windows Authentication で接続します。

✅ 3. SQL Server Configuration Manager を使う場合

SQL Server Configuration Manager を使った接続や設定変更については、Microsoft の公式ドキュメントを参照します（AWS ドキュメントではなく MS 純正）。

✅ 補足（接続全般の注意点）

✅ SQL Server への TCP 接続（例: 1433 ポート）での接続も可能（アプリや SSMS から）。
→ この場合は セキュリティグループのインバウンド設定でポート開放 が必要になります（例: 自分のIPからのみ 1433 へアクセス許可）。

✅ プライベートネットワークや VPN／VPC Peering 経由で接続する場合は、適切なルーティングとセキュリティグループ設定を確認します。

✅ RDP が使えない環境でも、SSM Session Manager を使して Windows に接続し、そこから SSMS を起動することも可能です（セキュリティ強化用途）。

✅ まとめ
接続方法	適用例
SSMS + RDP（Windows Authentication）	まずは管理者で接続したい
SSMS + ドメインユーザー（Windows Authentication）	AD ユーザーでのログインが必要
TCP での接続（例: 1433）	外部クライアントから SQL 接続
SSM Session Manager	RDP を開放したくない場合

## メモ

SSM経由でEC2のマネコン画面の接続から、接続するとコマンドライン画面となる
GUI操作が必要な場合は、FleetMangerでRDP接続する必要あり
その場合、Key Pair作成が必要

参考にした記事
https://techblog.techfirm.co.jp/entry/introduction-to-windows-server-on-ec2
https://techblog.techfirm.co.jp/entry/accessing-windows-server-with-ssm