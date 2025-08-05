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
