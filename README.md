$log = "C:\ProgramData\Amazon\SSM\Logs\amazon-ssm-agent.log"

Get-Content $log -Tail 2000 |
  Select-String -Pattern "connect" |
  Select-Object -Last 10

出力に表示される接続先が vpce-xxxx...amazonaws.com であることを確認する


https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/ec2launch-v2.html
# 最近一天（24小时）针对某实例的会话
aws ssm describe-sessions \
  --state History \
  --filters "key=Target,value=i-0123456789abcdef0" \
            "key=InvokedAfter,value=$(date -u -d '1 day ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --query "Sessions[?DocumentName=='AWS-StartPortForwardingSession'].{SessionId:SessionId,Owner:Owner,Start:StartDate,End:EndDate,Status:Status}"


1. なぜ最後に再起動を入れても UserData 全体が正常に実行されない場合があるのか（根本原因）

EC2Launch v2 の初期化ステージ順序（YAML v1.1 新版）
	1.	Boot – システム起動初期の処理（例: システムディスクの拡張）
	2.	Network – ネットワーク設定、通信可能化
	3.	PreReady – Windows 設定の適用（ライセンス認証、DNS サフィックス、ローカル管理者設定など）
	4.	UserData（YAML v1.1 新版 / XML） – カスタムスクリプト（PowerShell/XML）の実行
	5.	PostReady – 追加の初期化サービス起動（例: SSM Agent、その他のバックグラウンドタスク）

新しい Windows AMI はデフォルトで EC2Launch v2 + YAML v1.1 を使用しており、UserData は PostReady の前に実行されます。つまり、スクリプト終了後にも EC2 初期化エージェントは PostReady ステージの処理を控えています。

⸻

なぜ最後の行で再起動しても問題になるのか
	1.	PowerShell の逐次実行 = システムタスクの完全完了ではない
	•	スクリプトは順番に実行されますが、例えばドメイン参加、サービス起動、レジストリ書き込みなどはコード行が終わっても、バックグラウンドで書き込みやサービス登録、ポリシー同期が続いています。
	2.	即時再起動はシステムレベルの強制終了を引き起こす
	•	Restart-Computer や shutdown /r /t 0 を実行すると、Windows はすぐにシャットダウンイベントをブロードキャストし、サービス停止 → 全プロセス終了（スクリプトの PowerShell や親プロセスである EC2Launch v2 も含む）を行います。
	•	このため、まだ完了していないバックグラウンド処理が中断されるだけでなく、PostReady ステージの処理も実行されなくなります。例えば：
	•	SSM Agent の初期化と登録
	•	その他のシステムや AWS サービスの起動設定
	•	ログ書き込みや状態レポート

⸻

根本原因まとめ
	•	スクリプトの実行順序は PowerShell 内での順序に過ぎず、システムレベルのタスクやバックグラウンド処理がすべて完了している保証はありません。
	•	即時再起動コマンドは EC2Launch v2 をも終了させ、残りの初期化ステージ（PostReady）が実行されません。
	•	そのため、UserData の最後に再起動を書いても、前に起動した処理や後続の初期化タスクが両方とも失敗する可能性があります。


非同期で安全に再起動する考え方
	•	ねらい：UserData の PowerShell を早く終了させ、EC2Launch v2 が収尾（PostReady 等）を続けられる状態にしてから、OS 側で後から再起動させる。
	•	方法：
shutdown /r /t <秒> … OS に「○秒後に再起動してね」と予約だけ渡して、即終了

例（PowerShell／UserData の最後に置く）

# 5分後に再起動を予約（即終了）
shutdown /r /t 300 /c "Reboot after userdata finished"


⸻

なぜ Sleep はダメで、/t が有効なのか（直感的な説明）
	•	Sleep の場合
	•	PowerShell（UserData）は Start-Sleep 300 などでずっと止まっている
	•	**EC2Launch v2 は「まだスクリプト実行中だから待つね（＝ブロック）」**という状態
	•	Sleep が終わって Restart-Computer を実行した瞬間、OS は即座にシャットダウンシーケンスに入り、PowerShell と EC2Launch v2 をまとめて終了
	•	結果：「長時間待たせたあげく、やっと収尾に入ろうとした EC2Launch が Restart で殺される」 → 初期化の残処理やバックグラウンドの書き込み等が未完になりやすい
	•	shutdown /r /t N の場合
	•	ボールを先に投げるイメージ：OS に「N 秒後に再起動して」と依頼 → コマンドは即終了
	•	PowerShell はすぐ終わる → **EC2Launch v2 は「もう終わった？OK、収尾を続けるね」**と進められる
	•	再起動のカウントダウンは OS が裏で単独管理。EC2Launch v2 の収尾や、前段でトリガーしたバックグラウンド処理が猶予内に完了しやすい
	•	時間が来たら OS が再起動するが、その頃には初期化の収尾が終わっており、中断リスクが低い


Invoke-WebRequest -Uri https://www.update.microsoft.com -UseBasicParsing


reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v UseWUServer
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate /v WUServer

reg add HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v UseWUServer /t REG_DWORD /d 0 /f
net stop wuauserv & net start wuauserv

皆さん、こんにちは。画面は見えていますでしょうか？

それでは、インフラチームのスプリントレビューの発表を始めます。

初めてお会いする方もいらっしゃるかと思いますので、あらためて自己紹介させていただきます。
インフラチームBN25新人の趙飛と申します。よろしくお願いします。

今回も以前と同じく、まずは今スプリントのスプリントゴールの達成状況を共有し、そのあと今スプリントの成果物の内容に移りたいと思います。

⸻

まずはPhase1のスプリントゴールです。

1つ目は、8月頭までにADの信頼関係構築作業を完了させるとこと、こちらは完了していますので、マルを付けました。

2つ目は、FaxGW移行の実施可否が判断できる状態にするというもので、このゴールのうち、SXI社内での性能検証とスケジュールのリプランができており、こちらも完了しています。

3つ目は、必要な開発タスクの完了で、対象チケットの約6割が完了しています。

⸻

次はPhase2とPhase3のスプリントゴールです。

設計フェーズをクローズに近づけるというゴールに関しては、Sprint09でまとめた設計をUSSさんと連携済みです。

11月の提供Webサーバーに向けてSIPさん・USSさんとの連携も完了しています。

Webサーバーを稼働させるためのリソースについては、EC2、Route53、画像サーバー、ALB、WAF、SorryPageなどのリソースはデプロイまで　開発が完了しており、IT環境が整い次第、順次デプロイ予定です。

Shieldに関しては、有効化するとコストが発生するため、現在USSさんからの回答を待っています。

⸻

続いてPhase3です。

バックアップ要件と監視要件の整理については、
JobCenterの監視はSIPさんとのヒアリングが終わっており、現在整理中です。
DBのバックアップについても整理中で、今週の金曜日の定例で報告予定です。

ヒアリング内容に基づく検討スコープについては、このゴールは達成済みです。

⸻

最後に、暫定IT環境についてです。
SorryWebと外部連携用のDBサーバーについては、まだリソースは生成されていません。

⸻

スプリントゴールの達成状況は以上となります。
ここまでで何かご質問などあればお願いします。

時間の都合上、スプリントゴールの詳細な説明は割愛させていただきます。
Pull Requestの内容をご確認いただければと思います。



License周辺の整理

結論から言うと、LicenseはUserDataを用いたJobCenterの構築に影響しない。
	1.	License Managerさえインストールされていれば、コードワードにログインしなくても他の機能をインストールできる。
	2.	コードワードにログインしなくても、MG/SV機能をインストールするたびに60日間の試用期間が発生する（インストールのたびにカウント開始）。そのため、CDKで繰り返しデプロイしてUserDataのテストを行っても影響はない。

⸻

UserDataの活用について

現在の調査では、インストールからJPFファイルの設定までの流れはすべてスクリプト（UserData）で対応可能であることが確認された。検討内容は以下のリンクに整理してある。


# JobCenter 構築方針の検討（UserDataによる自動インストール、構築）


参照資料：  
- JobCenterスタンダードモード用セットアップガイド R16.2
- JobCenter移行ガイド R16.1

## 1. このメモの目的

このメモは、JobCenter サーバ（MGサーバ／APPサーバ）の移行において、インストール作業の初期構築フェーズを自動化することを目的とする。  

## 2. MG/APPサーバの構築手順

### 2.1 MGサーバの構築手順（LVVJOBMG01〜04、EVVJOBMG01）

```
① EC2新規構築（Windows Server）
② LicenseManagerのインストールとコードワード登録
③ CL/Win、MG/SVのインストール
④  .jpfファイルの編集
⑤ 構成情報のリストア
⑥ ジョブ定義のアップロード
```

### 2.2 APPサーバの構築手順（LVVAPP01〜08、EVVAPP01〜04）

```
① EC2新規構築（Windows Server）
② LicenseManagerのインストールとコードワード登録
③ MG/SVのインストール
④ .jpfファイルの編集
⑤ 構成情報のリストア
⑥ Back系ソースコードの配置
```

## EC2新規構築（①）略
- OS設定
- AWSCLIのインストール

## EC2 UserDataによる構築自動化（②～③）

### 概要：
- S3からインストールメディアをダウンロードする
- S3からJobCenterのMG/SV機能とCL/WIN機能の設定ファイルをダウンロードする
※MG/SV機能とCL/WIN機能の設定ファイルは一回GUIインストールすることで作成
※MG/SV機能の設定ファイルは編集可能、参考PDF「設定ファイルの変更可能なパラメータ一覧」に沿って編集
- ISOをマウントして、必要なインストーラファイルをコピーし、実行する
- 以下の機能を自動インストールする
    - LicenseManager
    - CL/Win（MGサーバの場合）
    - MG/SV


### スクリプト

```
powershell

#[Step 1] インストールメディアのダウンロード元（S3 URL）
$isoS3Url = "https://your-bucket.s3.amazonaws.com/JobCenter_R16.2.iso"

#[Step 2] 保存先パスの定義
$localDir = "C:\Windows\Temp"
$isoPath = "$localDir\JobCenter_R16.2.iso"

#[Step 3] ローカル作業用ディレクトリ作成
New-Item -ItemType Directory -Force -Path $localDir

#[Step 4] S3からISOファイルをダウンロード
Invoke-WebRequest -Uri $isoS3Url -OutFile $isoPath

#[Step 5] ISOファイルをマウント（仮想DVDとして読み込み）
$iso = Mount-DiskImage -ImagePath $isoPath -PassThru
$driveLetter = ($iso | Get-Volume).DriveLetter  # 例: Q

#[Step 6] パッケージファイルを Temp\LM にコピー
$lmDir = "C:\Windows\Temp\LM"
New-Item -ItemType Directory -Force -Path $lmDir

Copy-Item "$driveLetter`:\PACKAGE\LM\WINDOWS\x64\setup.bat" -Destination $lmDir

#[Step 7] カレントディレクトリを Temp\LM に変更
Set-Location -Path $lmDir

#[Step 8] LicenseManager をインストール（インストール先を C:\JobCenter\LM に指定）
& "$lmDir\setup.bat" "C:\JobCenter\LM"

#[Step 8] CL/Win（MGサーバの場合のみ）

# CLWIN 作業用ディレクトリ
$clDir = "C:\Windows\Temp\CLWIN"
New-Item -ItemType Directory -Force -Path $clDir

# install.bat スクリプトをコピー
Copy-Item "$driveLetter`:\PACKAGE\JB\WINDOWS\CLWIN\script\*" -Destination $clDir -Recurse

# サイレントインストール設定ファイルを S3 から取得
$confS3Url = "https://your-bucket.s3.amazonaws.com/clsetup.conf"
$confPath = "C:\Windows\Temp\clsetup.conf"
Invoke-WebRequest -Uri $confS3Url -OutFile $confPath

# カレントディレクトリを CLWIN に変更
Set-Location -Path $clDir

# install.bat を実行
& "$clDir\install.bat" "$confPath"

####[Step 9] MG/SV（Manager/Server）

# MGSV 作業用ディレクトリ
$mgsvDir = "C:\Windows\Temp\MGSV"
New-Item -ItemType Directory -Force -Path $mgsvDir

# install.bat スクリプトなどをコピー
Copy-Item "$driveLetter`:\PACKAGE\JB\WINDOWS\MGSV\x64\script\*" -Destination $mgsvDir -Recurse

# サイレントインストール設定ファイルを S3 から取得
$mgsvConfUrl = "https://your-bucket.s3.amazonaws.com/jcsetup.conf"
$mgsvConfPath = "C:\Windows\Temp\jcsetup.conf"
Invoke-WebRequest -Uri $mgsvConfUrl -OutFile $mgsvConfPath

# カレントディレクトリを MGSV に変更
Set-Location -Path $mgsvDir

# install.bat を実行（/c オプション指定）
& "$mgsvDir\install.bat" "/c" "$mgsvConfPath"

#[Step 10] 後処理：ISOのアンマウント
Dismount-DiskImage -ImagePath $isoPath


```

---

## 構成情報ファイルのリストア手順（④～⑤）

### 概要

- S3からルールファイルと構成情報ファイルをダウンロードする
※ルールファイルには、ノード名、IPアドレス、ホスト名の変更を記述している。
※本メモ隣のルールファイル例を参照すること
- ルールファイルを用いての.jpfファイルの編集
- obCenterの停止（リストア前に必要）
- 変換済み構成情報のリストア

### スクリプト

```powershell
#[Step 1] S3からルールファイルと構成情報ファイル（.jpf）をダウンロード

$ruleS3Url = "https://your-bucket.s3.amazonaws.com/restore_rule.txt"
$jpfS3Url  = "https://your-bucket.s3.amazonaws.com/cluster_config.jpf"

$localDir = "C:\Windows\Temp\JCRestore"
New-Item -ItemType Directory -Force -Path $localDir

$ruleFilePath = "$localDir\restore_rule.txt"
$jpfFilePath  = "$localDir\cluster_config.jpf"
$convertedJpf = "$localDir\converted_cluster_config.jpf"

Invoke-WebRequest -Uri $ruleS3Url -OutFile $ruleFilePath
Invoke-WebRequest -Uri $jpfS3Url -OutFile $jpfFilePath


#[Step 2] jpf_config を使用して.jpfファイルを変換

$jcBin = "D:\JobCenter\SV\bin"
& "$jcBin\jpf_config" update -f $ruleFilePath -o $convertedJpf $jpfFilePath


#[Step 3] JobCenter を停止
Stop-Service -Name "jobcenter" -Force


#[Step 4] jc_restore conf を使って構成情報をリストア

& "$jcBin\jc_restore" conf $convertedJpf

# （必要に応じてサービス再起動）
Start-Service -Name "jobcenter"

```
---

## ジョブ定義ファイルのアップロード手順（⑥ ）

### 概要

- S3からルールファイルとジョブ定義ファイルをダウンロードする
※ルールファイルには、ノード名、IPアドレス、ホスト名の変更を記述している。
※本メモ隣のルールファイル例を参照すること
- ルールファイルを用いての.jpfファイルの編集
- JobCenterの停止（リストア前に必要）
- 変換済み構成情報のリストア

### スクリプト

```powershell
#[Step 6] ジョブ定義ファイル（JPF）のアップロード

#変換済み JPF ファイルのパス
$convertedJpf = "C:\Windows\Temp\converted_cluster_config.jpf"
$ruleFilePath = "C:\Windows\Temp\restore_rule.txt"  # 同じルールを再利用する場合

$jcBin = "D:\JobCenter\SV\bin"

#jdh_upload 実行
& "$jcBin\jdh_upload" -r $ruleFilePath
```
---
```
※コマンド全利用可能なパラメータ（移行ガイド240ページ）：
%InstallDirectory%\bin\jdh_upload [-h %hostname%[:%port%]] [-u %user%] [-
p %password%] [-c] [-r %rulefile%] [-f] [-i] [-w %second%] {%jpf_file%|-a
%jpf_dir%}

-h %hostname%[:%port%] アップロード先のマシン名[:jccombaseのTCPポート番号]
指定しないとコマンド実行マシンへアップロードする。
-u %user% 接続するログインユーザ
指定しないとコマンド実行ユーザでログインする。
-p %password% ログイン先ユーザのパスワード（平文）
指定しないとパスワードプロンプトが表示される。
-c チェックモード
チェックモードでは定義情報の依存関係確認のみを行い、サーバ上の定義情
報を更新しません。
-r %rulefile% ホスト名変換用のルールファイル名を指定する。
-f キュー情報のチェックを省略する。
-i 拡張アイコンデータも対象に含めます。
拡張カスタムジョブテンプレートをアップロードする場合は、本オプション
を設定すること。
-w %second% サーバとの通信タイムアウト時間の指定
30～86400秒が設定可能
指定しないと600秒
%jpf_file% アップロードする定義情報(JPFファイル)を指定する。本パラメータか-a
オプションのディレクトリパラメータどちらかを必ず設定すること。
-a %jpf_dir% 指定されたディレクトリ(%jpf_dir%)内のjpfファイルをすべてアップロード
する。全ユーザダウンロードで生成したディレクトリを指定してくださ
い。

```
