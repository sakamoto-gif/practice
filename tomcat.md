# 最終チーム演習APサーバー構築手順
## JDKインストール
```
sudo dnf install -y java-17-amazon-corretto
```
### 確認
```
java --version
```
## Tomcat 9.0.12 インストール
### tomcat専用ユーザーを作成する。
```
セキュリティの観点から、rootユーザーでtomcatを起動するのは危ないため
sudo useradd -r -s /sbin/nologin tomcat
```

### アカウントの確認
```
id tomcat
実行結果 tomcatグループのtomcatユーザーがちゃんとあるよって言ってる
uid=992(tomcat) gid=992(tomcat) groups=992(tomcat)
```

### ホームディレクトリ移動
```
cd
```
### 公式からアーカイブをダウンロード
```
wget https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.12/bin/apache-tomcat-9.0.12.tar.gz
```
### ダウンロードしたパッケージを展開し、一覧で表示する
```
sudo tar -zxvf apache-tomcat-9.0.12.tar.gz 
```
### ダウンロードしたapache-tomcat-9.0.12ファイルを、/usr/localディレクトリに移動する
```
sudo mv apache-tomcat-9.0.12 /usr/local/
```
### シンボリックリンクを作成する（ショートカットみたいなもの）
```
sudo ln -s /usr/local/apache-tomcat-9.0.12/ /usr/local/tomcat
```
### 先ほど作成したtomcatユーザーに、/apache-tomcat-9.0.12、/tomcatの権限を付与
```
sudo chown -R tomcat:tomcat /usr/local/apache-tomcat-9.0.12
sudo chown -R tomcat:tomcat /usr/local/tomcat
```
### 権限が変更されたか確認
```
cd /usr/local/
ls -l
実行結果（下記のような表示があれば成功）
drwxr-xr-x. 9 tomcat tomcat 16384 Mar  5 04:30 apache-tomcat-9.0.12
lrwxrwxrwx. 1 tomcat tomcat    32 Mar  5 01:10 tomcat -> /usr/local/apache-tomcat-9.0.12/
```

### ユニットファイルをtomcat.serviceと言う名前で、"/etc/systemd/system/"に作成し、エディタ起動
```
sudo vi /etc/systemd/system/tomcat.service

以下をエディタに記述

[Unit]
Description=Apache Tomcat 9.0.12
ConditionPathExists=/usr/local/tomcat

[Service]
User=tomcat
Group=tomcat
Type=oneshot
RemainAfterExit=yes

ExecStart=/usr/local/tomcat/bin/startup.sh
ExecStop=/usr/local/tomcat/bin/shutdown.sh
ExecReStart=/usr/local/tomcat/bin/shutdown.sh;/usr/local/tomcat/bin/startup.sh
Restart=no

Environment="DASH_DATABASE_URL=jdbc:postgresql://172.31.22.198:5432/dash_replace"
Environment="DASH_DATABASE_USER=postgres"
Environment="DASH_DATABASE_PASS=postgres"

[Install]
WantedBy=multi-user.target
```
### ユニットファイルの読み込み（systemdに変更を反映させる）
```
daemon=デーモンとは、常駐プログラムのこと。
sudo systemctl daemon-reload
```
```
ユニットファイルの有効化（自動起動設定を有効にする）
sudo systemctl enable tomcat.service
```

### エディタで .bash_profile ファイル開く
```
sudo vi ~/.bash_profile
以下を記述

# .bash_profile

# Get the aliases and functions
if [ -f ~/.bashrc ]; then
        . ~/.bashrc
fi

# User specific environment and startup programs

PATH=$PATH:$HOME/.local/bin:$HOME/bin

export PATH
```
### bash_profileシェルを実行する
```
source ~/.bash_profile
```


### setenvを作成する
```
vi /usr/local/src/tomcat/bin/setenv.sh
---ここから--------------------------------------
#!/bin/sh
export CATALINA_HOME=/usr/local/tomcat
export JAVA_HOME=/usr/lib/jvm/java-17-amazon-corretto.x86_64
export JAVA_OPTS="-Xms128m -Xmx512m"
---ここまで--------------------------------------
```

### server.xml
```
自動デプロイを有効にする　↓演習ではfalseだった
vi /usr/local/src/tomcat/conf/server.xml
unpackWARsとautoDeployはtrueのままでOK
```

### tomcatの自動起動、有効化
```
sudo systemctl enable tomcat.service
```
### tomcatを起動
```
sudo systemctl start tomcat.service
```
### tomcatの状態確認
```
sudo systemctl status tomcat.service
実行結果
 tomcat.service - Tomcat Web Server
  ● tomcat.service - Apache Tomcat 9.0.12
     Loaded: loaded (/etc/systemd/system/tomcat.service; enabled; preset: disabled)
     Active: active (exited) since Fri 2026-03-06 00:02:30 UTC; 7min ago
```
### デプロイ
```
一度踏み台サーバーのホームディレクトリにに.warファイルを転送
その後scpコマンドでwarファイルを転送
scp -i ~/.ssh/entrycl_202601.pem ~/ROOT.war ec2-user@移行先のIP:/home/ec2-user/
```
```
移行先のサーバーで以下コマンドを実施
mv /home/ec2-user/ROOT.war /usr/local/tomcat/
```
