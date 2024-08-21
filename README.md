# mysql-playground
mysqlのレプリケーションなどの動き・手触りを学習するためのplayground

- [mysql-playground](#mysql-playground)
- [macでの環境セットアップ](#macでの環境セットアップ)
  - [mysql-client](#mysql-client)
  - [percona tool-kit](#percona-tool-kit)
- [初回起動から構成セットアップまで](#初回起動から構成セットアップまで)
  - [masterの起動](#masterの起動)
- [いろんな演習](#いろんな演習)
  - [初回起動後にいろいろ見てみる](#初回起動後にいろいろ見てみる)
    - [binlogポジション](#binlogポジション)
    - [binlogファイル](#binlogファイル)
    - [mysqlbinlogで初回セットアップの動きを見る](#mysqlbinlogで初回セットアップの動きを見る)
  - [初期状態からreplicationを張ってみる](#初期状態からreplicationを張ってみる)
    - [replユーザを作る](#replユーザを作る)
    - [replicationを張る](#replicationを張る)
  - [いろいろやりなおしたいとき](#いろいろやりなおしたいとき)
  - [意図的にレプリケーション不整合を起こしてみる](#意図的にレプリケーション不整合を起こしてみる)
    - [replicationを停止する](#replicationを停止する)
    - [master側にinsertを行う](#master側にinsertを行う)
    - [破壊的なreplication設定](#破壊的なreplication設定)
    - [master側でid=2のレコードを更新](#master側でid2のレコードを更新)
    - [replication不整合を確認](#replication不整合を確認)
    - [不整合の原因レコードを調べる](#不整合の原因レコードを調べる)

# macでの環境セットアップ

## mysql-client
- macでhomebrewでmysql clientを入れる場合（docker execでコンテナローカルで操作しても良い）
```bash
# 大量の依存パッケージ（なぜかbrew awscliなど）がinstallされて時間がかかる
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

## percona tool-kit
```bash
brew install percona-toolkit

# /opt/homebrew/bin/pt-*でいろいろinstallされる
ls -1 /opt/homebrew/bin/pt-* 
```

# 初回起動から構成セットアップまで

## masterの起動
```bash
docker network create mysql_test

# サーバ全部起動
docker compose up -d

# master, slaveでそれぞれコンテナを起動できます(するケースはないかもだけれど)
docker compose up db80-master -d
docker compose up db80-slave -d
```

# いろんな演習

## 初回起動後にいろいろ見てみる

### binlogポジション
- 同じbinlogファイル・ポジションが示される
```bash
# master側（まだreplicationを繋いでいない初期状態）
mysql -uroot -proot -h127.0.0.1 -P 3307 -e 'show master status'
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |      157 | playground   |                  |                   |
+------------------+----------+--------------+------------------+-------------------+

# slave側（まだreplicationを繋いでいない初期状態）
mysql -uroot -proot -h127.0.0.1 -P 3308 -e 'show master status'
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |      157 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```

### binlogファイル
- 現在、MySQLサーバ上に存在するbinlogファイルの一覧が表示される
```bash
# master側（まだreplicationを繋いでいない初期状態）
mysql -uroot -proot -h127.0.0.1 -P 3307 -e 'show master logs;'
+------------------+-----------+-----------+
| Log_name         | File_size | Encrypted |
+------------------+-----------+-----------+
| mysql-bin.000001 |       180 | No        |
| mysql-bin.000002 |      1217 | No        |
| mysql-bin.000003 |       157 | No        |
+------------------+-----------+-----------+

# slave側（まだreplicationを繋いでいない初期状態）
mysql -uroot -proot -h127.0.0.1 -P 3308 -e 'show master logs;'
+------------------+-----------+-----------+
| Log_name         | File_size | Encrypted |
+------------------+-----------+-----------+
| mysql-bin.000001 |       180 | No        |
| mysql-bin.000002 |   3118277 | No        |
| mysql-bin.000003 |       157 | No        |
+------------------+-----------+-----------+
※ mysql-bin.000002のbinlogのサイズが多い理由は不明（あとの手順のmysqlbinlogで実行結果が見れるが大量の `INSERT INTO mysql.time_zone_transition` が記録されていた)
```

### mysqlbinlogで初回セットアップの動きを見る
- `show master logs` で確認したbinlogファイルの中身を見ることで初期セットアップ時に実行された更新系SQLを見ることができる(seedデータの投入確認ができる)
```bash
# master側（まだreplicationを繋いでいない初期状態）
mysqlbinlog -uroot -proot -h127.0.0.1 -P 3307 --read-from-remote-server --base64-output=DECODE-ROWS --verbose mysql-bin.000001
mysqlbinlog -uroot -proot -h127.0.0.1 -P 3307 --read-from-remote-server --base64-output=DECODE-ROWS --verbose mysql-bin.000002
mysqlbinlog -uroot -proot -h127.0.0.1 -P 3307 --read-from-remote-server --base64-output=DECODE-ROWS --verbose mysql-bin.000003

# slave側（まだreplicationを繋いでいない初期状態）
mysqlbinlog -uroot -proot -h127.0.0.1 -P 3308 --read-from-remote-server --base64-output=DECODE-ROWS --verbose mysql-bin.000001
mysqlbinlog -uroot -proot -h127.0.0.1 -P 3308 --read-from-remote-server --base64-output=DECODE-ROWS --verbose mysql-bin.000002
mysqlbinlog -uroot -proot -h127.0.0.1 -P 3308 --read-from-remote-server --base64-output=DECODE-ROWS --verbose mysql-bin.000003
```

## 初期状態からreplicationを張ってみる

### replユーザを作る

```bash
# masterのshow master statusを確認
mysql -uroot -proot -h127.0.0.1 -P 3307 -e 'show master status;'
# 以下のポジションである前提で以下手順を書いています
| mysql-bin.000003 |      157 | playground   |                  |                   |

# master側だけで実行
mysql -uroot -proot -h127.0.0.1 -P 3307

CREATE USER 'repl'@'%' IDENTIFIED WITH 'mysql_native_password' BY 'repl';
GRANT REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

# ここでmasterのポジションが進む（SQLが実行されるので）が、replicationの動きを説明もあるのでこのまますすめて構いません（手順漏れではないです）
```

### replicationを張る
```bash
# slave側にログイン
mysql -uroot -proot -h127.0.0.1 -P 3308

# replicationを貼る前の自身のポジションを確認する
show master status;

# masterへのreplication接続設定
CHANGE MASTER TO
    MASTER_HOST='db80-master',
    MASTER_USER='repl',
    MASTER_PASSWORD='repl',
    MASTER_LOG_FILE='mysql-bin.000003',
    MASTER_LOG_POS=157;

# 設定確認
show slave status\G

# レプリケーション開始
start slave;

# レプリケーション動作確認
show slave status\G

# 上記のshow slave statusの結果で以下のような記述が見られれば問題ありません。問題があればLast_IO_Error:にエラーが出力されます。
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes

# replicationを張ったあとの自身のポジションを確認する
# ポジションが進んでいることが確認できます。
show master status;

# master側のポジションを確認して同じかどうか確認してみる
# 何が実行されてポジションが進んだかをmysqlbinlogで確認してみる

```

## いろいろやりなおしたいとき
- data_80_master, data_80_slaveのディレクトリの中身を吹き飛ばせば初期化できます
```bash
docker compose down

rm -rf ./data_80_master
rm -rf ./data_80_slave

# 再び初期化から始まります
docker compose up -d
```

## 意図的にレプリケーション不整合を起こしてみる
- master側にinsertされたデータが、slave側には存在しない状態を作り出します
- slave側に存在しないレコードをmaster側で更新した場合にどのようになるのかを例示します
- また、どのようなシチュエーションで不整合が起きるのかbinlog(Non-GTIDの場合)の仕組みも交えて触ってもらいます

### replicationを停止する
```bash
# slave側にログイン
mysql -uroot -proot -h127.0.0.1 -P 3308

stop slave
show slave status\G

# 以下の部分がNoになっている状態を確認します
             Slave_IO_Running: No
            Slave_SQL_Running: No

# 以下のようなmaster側のどのポジションまで読んでいたかを確認する
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 839

# 自分のポジション（停止した状態）を確認しておく
show master status\G
```

### master側にinsertを行う
```bash
# master側にログイン
mysql -uroot -proot -h127.0.0.1 -P 3307

INSERT INTO playground.users (name, age, email, created_at, updated_at) VALUES
('鈴木花子', 30, 'suzuki.hanako@example.com', '2023-02-01 12:00:00', '2023-02-01 12:00:00');

mysql> select * from playground.users;
+----+--------------+-----+---------------------------+---------------------+---------------------+
| id | name         | age | email                     | created_at          | updated_at          |
+----+--------------+-----+---------------------------+---------------------+---------------------+
|  1 | 田中太郎     |  25 | tanaka.taro@example.com   | 2023-01-01 12:00:00 | 2023-01-01 12:00:00 |
|  2 | 鈴木花子     |  30 | suzuki.hanako@example.com | 2023-02-01 12:00:00 | 2023-02-01 12:00:00 |
+----+--------------+-----+---------------------------+---------------------+---------------------+
2 rows in set (0.01 sec)

# master側のポジションはINSERTにより進んでいる(slave側のshow slave statusと比較して)
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |     1201 | playground   |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

### 破壊的なreplication設定
slave側で意図的にポジションを進めたreplication設定に変える
```bash
# slave側にログイン
mysql -uroot -proot -h127.0.0.1 -P 3308

# FILE, POSは上記で実際に確認したreplication設定に変えて実行する
CHANGE MASTER TO
    MASTER_HOST='db80-master',
    MASTER_USER='repl',
    MASTER_PASSWORD='repl',
    MASTER_LOG_FILE='mysql-bin.00000X',
    MASTER_LOG_POS=XXX;

# replicationを再開
start slave;

# Slave_XXX_RunningがYes/Yesであることを確認する
show slave status\G

# slave側にはid=2のレコードがない（降ってきていない）ことを確認
mysql> select * from playground.users;
+----+--------------+-----+-------------------------+---------------------+---------------------+
| id | name         | age | email                   | created_at          | updated_at          |
+----+--------------+-----+-------------------------+---------------------+---------------------+
|  1 | 田中太郎     |  25 | tanaka.taro@example.com | 2023-01-01 12:00:00 | 2023-01-01 12:00:00 |
+----+--------------+-----+-------------------------+---------------------+---------------------+
1 row in set (0.01 sec)
```

### master側でid=2のレコードを更新
```bash
# master側にログイン
mysql -uroot -proot -h127.0.0.1 -P 3307

# 更新
UPDATE playground.users SET age = 31 WHERE id = 2;
```

### replication不整合を確認
```bash
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: db80-master
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 1612
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 326
        Relay_Master_Log_File: mysql-bin.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: No
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 1032
                   Last_Error: Coordinator stopped because there were error(s) in the worker(s). The most recent failure being: Worker 1 failed executing transaction 'ANONYMOUS' at master log mysql-bin.000003, end_log_pos 1581. See error log and/or performance_schema.replication_applier_status_by_worker table for more details about this failure or others, if any.
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1201
              Relay_Log_Space: 947
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 1032
               Last_SQL_Error: Coordinator stopped because there were error(s) in the worker(s). The most recent failure being: Worker 1 failed executing transaction 'ANONYMOUS' at master log mysql-bin.000003, end_log_pos 1581. See error log and/or performance_schema.replication_applier_status_by_worker table for more details about this failure or others, if any.
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: f66e35b0-5fb4-11ef-b9d2-0242ac150002
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: 
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 240821 21:34:33
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
       Master_public_key_path: 
        Get_master_public_key: 0
            Network_Namespace: 
1 row in set, 1 warning (0.01 sec)
```

### 不整合の原因レコードを調べる
- master側のbinlogで該当箇所を見つける
```bash
# master側のbinlogを調べる
mysqlbinlog -uroot -proot -h127.0.0.1 -P 3307 --read-from-remote-server --base64-output=DECODE-ROWS --verbose mysql-bin.000003

# end_log_pos 1581 とエラーに出ているのでmaster側のposition: 1581の実行結果を探すと以下のように該当のupdate文のクエリーでエラーになっていることがわかります。
#240821 21:34:33 server id 1  end_log_pos 1581 CRC32 0xce7e3576 	Update_rows: table id 91 flags: STMT_END_F
### UPDATE `playground`.`users`
### WHERE
###   @1=2
###   @2='鈴木花子'
###   @3=30
###   @4='suzuki.hanako@example.com'
###   @5=1675220400
###   @6=1675220400
### SET
###   @1=2
###   @2='鈴木花子'
###   @3=31
###   @4='suzuki.hanako@example.com'
###   @5=1675220400
###   @6=1724243673
# at 1581
```

- slave側のrelaylogで該当箇所を見つける
- こちらの方がreplicationが止まっており該当データが見つけやすい(relay-logとbinlogの違いの理解)
```bash
# relaylogはremoteコマンドで確認することができないため、docker execでコンテナ内で実行
mysqlbinlog --no-defaults /var/lib/mysql/mysql-relay-bin.000002 --base64-output=DECODE-ROWS --verbose

# 以下のように表示される。
#240821 21:34:33 server id 1  end_log_pos 1581 CRC32 0xce7e3576 	Update_rows: table id 91 flags: STMT_END_F
### UPDATE `playground`.`users`
### WHERE
###   @1=2
###   @2='鈴木花子'
###   @3=30
###   @4='suzuki.hanako@example.com'
###   @5=1675220400
###   @6=1675220400
### SET
###   @1=2
###   @2='鈴木花子'
###   @3=31
###   @4='suzuki.hanako@example.com'
###   @5=1675220400
###   @6=1724243673
# at 706
#240821 21:34:33 server id 1  end_log_pos 1612 CRC32 0x87503e25 	Xid = 47
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```
