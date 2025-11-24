# EC2 Remote Controller 環境構築手順まとめ

## 1. はじめに

本ドキュメントは、AWS 上で **2台の EC2 インスタンスを用いたリモート制御環境** を構築した際の手順と学びを整理したものです。  
Python / Flask アプリケーションから、別インスタンスの起動・停止を制御できるようにすることを目標としています。

## 2. 構成の概要

- AWS EC2 インスタンスを 2 台使用
  - **コントローラーサーバー** … Web アプリケーションを配置し、操作画面を提供
  - **ターゲットサーバー** … コントローラーから起動・停止を制御される側

- 目標  
  コントローラーサーバーから、ターゲットサーバーの EC2 インスタンスを起動・停止できるようにする。

## 3. 用語整理
- **EC2**: AWS が提供する仮想サーバー
- **SQLAlchemy**: Python ORM (データベースを扱うライブラリ)
- **Alembic**: SQLAlchemy 用の DB マイグレーションツール
- **Flask**: Python 製 Web フレームワーク  
セットアップ時点では知らなかった用語も多かったため、必要に応じて公式ドキュメントや技術記事を参照しながら理解を深めていきました。

## 4. 環境準備
- ローカル PC から EC2 へ SSH 接続できるよう、秘密鍵を設定
- `~/.ssh/config` を用意し、`ssh controller` / `ssh target` のように短いコマンドで接続可能な状態に調整
- Python アプリケーションに必要なライブラリをインストール  
  （Flask, Flask-Babel, Paste, boto3, SQLAlchemy, mysql-connector-python, PyJWT など）


## 5. コントローラーサーバーのセットアップ
### 5.1.各種インストール
アプリケーションの動作に必要な Python と開発ツールを導入します。  
EC2 の初期状態では Python 3.7 が入っていましたが、今回は Python 3.8 を利用する想定としました。

### Git をインストール
```
sudo yum install git 
```

### Python3.8 を有効化
```
sudo amazon-linux-extras install python3.8
```

### Python 開発ヘッダやビルドに必要なツール
```
 sudo yum install python38-devel
 sudo yum install gcc
 sudo yum install gcc-c++
```
### 5.2. MySQL、データベース関連
次にアプリケーションが利用する MySQL をセットアップしました。
リポジトリの追加 → サーバーインストール → 初期パスワード確認 → DB 作成 → 認証方式設定、の流れで実施しました。

### MySQL クライアント/サーバー関連をインストール
```
 sudo yum install mysql
 sudo yum localinstall https://dev.mysql.com/get/mysql80-community-release-el7-1.noarch.rpm
 sudo yum install mysql-community-server
```

### サーバー起動
```
sudo service mysqld restart
```

### 初期パスワード確認
```
sudo cat /var/log/mysqld.log
```
### root ログイン
```
mysql -u root -p
```
### root パスワード変更（任意の強固なパスワードに変更）
```
ALTER USER 'root'@'localhost'
  IDENTIFIED WITH mysql_native_password BY '********';
```
### データベース作成
```
CREATE DATABASE ec2_controller CHARACTER SET utf8mb4;
```

### /etc/my.cnf にて、必要に応じて認証方式を設定します。
```
sudo vi /etc/my.cnf  
# 例：
# default-authentication-plugin=mysql_native_password
```
### 5.3. アプリケーションコードの取得
アプリケーション一式はGitHubより取得。
```
 git clone <リポジトリURL>  
 cd ec2-remote-controller  
```
主な利用ファイルは以下です。  
・requirements.txt  
 → アプリケーションが必要とする Python ライブラリ一覧を定義しており、環境構築時にインストールに使用。

・app.ini.example  
→ DB 接続情報や AWS 設定のサンプル（app.ini の元になる）

・app.py  
→ アプリケーションの起動スクリプト。

### install requirements  
requirements(requirements.txtの内容)をインストール。

### run 認証トークン生成用スクリプト
管理画面ログイン用のパスワードハッシュを生成する Python スクリプトを実行し、
ベーシック認証に使用するハッシュ値を作成。

### 5.4. 設定ファイル(app.ini)の作成・修正

### create app.ini file  
"app.ini.default"をコピーし、"app.ini"ファイルを作成。  
以下の項目を環境に合わせて修正。  

- `mysql_user` : root
- `mysql_pw` : 任意のパスワード
- `mysql_db` : ec2_controller
- `mysql_host` : localhost
- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_KEY` : 使用する IAM ユーザーのアクセスキー
- `app_secret_key` : Flaskアプリ用のシークレットキー
- `basic_user` / `basic_password` : 管理画面のベーシック認証用ユーザー/パスワード

### 5.5. コントローラーサーバーのcrontabを編集  
コントローラー側で定期的な更新処理を走らせるため、crontab を設定します。  
```
"* * * * * cd /home/ec2-user/ec2-remote-controller; python3 update_pull.py"
```

## 6. ターゲットサーバーのセットアップ  
コントローラーサーバーと同様に、Python と必要なライブラリを導入し、アプリケーションを配置します。 

### 6.1. コントローラーサーバーと同様の手順を実行  
```
sudo amazon-linux-extras install python3.8
sudo yum install python38-devel gcc gcc-c++
```

### 6.2.アプリケーションコードの取得  
```
git clone <このリポジトリのURL>
cd ec2-remote-controller  
```

### 6.3. install requirements  
requirements(requirements.txtの内容)をインストール。

### 6.4. 設定ファイル(app.ini)の作成・修正  
ターゲットサーバーに対応した設定を行います。

### 6.5. ターゲットサーバーのcrontabを編集  
ターゲットサーバーもcrontab を設定します。  
```
"* * * * * cd /home/ec2-user/ec2-remote-controller; python3 client.py"
```

## 7. 初期化スクリプトの実行  
コントローラーサーバーで、アプリケーションの初期化処理を行います。  
`python3.8 init.py`　

これにより、アプリケーションが利用するテーブル群が作成されます。

## 8. アプリケーションの起動
 `nohup /usr/bin/python3.8 app.py &` でアプリケーション起動。  

 起動後、ブラウザから管理画面にアクセスします。  
 `https://<コントローラーサーバーのドメインまたはIP>/server/list`  

`app.ini` で設定した `basic_user` / `basic_password` でログインします。


## 9. 動作確認　　

- 管理画面のサーバー一覧にターゲットインスタンスが表示されていること
- 「起動」「停止」操作を行うと、ターゲットインスタンスのステータスが
stopping → running と変化することを確認

`start`を押下し、停止中のターゲットサーバーを起動。
![](images/server①.png)

ステータスが`stopping`から`running`になった事を確認。
![](images/server②.png)

## 10. トラブルシュートと対応
### 10.1. MySQL リポジトリの GPG 鍵エラー
**現象**  
`sudo yum install mysql-community-server` 実行時に以下のようなエラーが発生。
> Public key for mysql-community-server-8.x.x.rpm is not installed

**原因**  
MySQL の公式リポジトリに署名された GPG 公開鍵がインポートされていなかったため。

**対応**  
以下のコマンドで公開鍵をインポートして再実行し、正常にインストールを完了。  

```
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022  
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo yum install mysql-community-server -y
```

### 10.2. requirements.txt インストール時の依存関係エラー

**現象**  
pip install -r requirements.txt 実行時に複数のライブラリでビルドエラーが発生。
特に Python 3.8 非対応の旧版が指定されていたため、インストールに失敗。

**原因**  
requirements.txt に記載されている一部のライブラリ（Flask, Flask-Babel, Flask-WTF, Paste など）が Python 3.7 時代の古い指定だったため、Python 3.8 ではエラーとなったと推測。

**対応**  
主要ライブラリを最新版で個別インストールし、互換性を確認。  
その後、依存関係を確認し、アプリの起動が正常に行えることを確認した。

## 12. 感想・学び

今回の AWS / EC2 を用いた環境構築では、これまで触れる機会の少なかった
AWS・Python アプリケーション・MySQL の連携 を体系的に理解する良い経験になりました。

特に、

EC2 間の役割分担（コントローラー / ターゲット）

VPC 内でのネットワーク構成

アプリケーションと AWS SDK（boto3）の動きのつながり

こうした「サービス同士の関係性」を意識しながら構築することの重要性を強く実感しました。

また構築の途中では、MySQL のリポジトリ設定エラーや Python ライブラリの互換性問題など、環境構築ならではのトラブルにも複数直面しました。
その中で、

依存関係管理の大切さ

環境ごとの差異を踏まえて進める姿勢

エラーメッセージを起点とした検証・調査のプロセス

といった、エンジニアリングに必要な基礎的な考え方を実践的に身につけることができました。

今回の構築を通して、
クラウド（AWS）とバックエンド（Python/Flask）を横断しながら、環境を整えていく楽しさ を強く感じました。
今後もクラウド・インフラ領域の学習を継続しつつ、開発と運用の両面に関われるエンジニアを目指してスキルを伸ばしていきたいと考えています。

## 参考資料　　
- https://qiita.com/Code_Dejiro/items/c97c400b92a85dce4468  
(【AWS EC2】Amazon Linux2 にMySQLをインストールしようとしてGPGでつまづいた話)
