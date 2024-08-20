# mysql-playground
mysqlのレプリケーションなどの動き・手触りを学習するためのplayground

## masterの起動
```bash
docker compose up db80-master -d
```

## mysql-client
- macでhomebrewでmysql clientを入れる場合（docker execでコンテナローカルで操作しても良い）
```bash
brew install mysql-client

# 標準ではパスを通していないので、以下のようなメッセージを参考にrcファイルにパスを追加すること
==> mysql-client
mysql-client is keg-only, which means it was not symlinked into /opt/homebrew,
because it conflicts with mysql (which contains client libraries).

If you need to have mysql-client first in your PATH, run:
  echo 'export PATH="/opt/homebrew/opt/mysql-client/bin:$PATH"' >> ~/.zshrc

For compilers to find mysql-client you may need to set:
  export LDFLAGS="-L/opt/homebrew/opt/mysql-client/lib"
  export CPPFLAGS="-I/opt/homebrew/opt/mysql-client/include"
```

## replユーザを作る
CREATE USER 'repl'@'%' IDENTIFIED BY 'repl';
GRANT REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

## replicationを張る
CHANGE MASTER TO
    MASTER_HOST='master_host_ip',
    MASTER_USER='replication_user',
    MASTER_PASSWORD='your_password',
    MASTER_LOG_FILE='mysql-bin.000001',  # SHOW MASTER STATUSで得たログファイル
    MASTER_LOG_POS=12345;  # SHOW MASTER STATUSで得たポジション