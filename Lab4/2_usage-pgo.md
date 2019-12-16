
# 2. pgoの構成とPostgreSQLリソース制御  

## 2-1. 諸注意
### 2-1-1. pgoについて
pgoは，Postgres Operatorを操作・制御するためのクライアント用CLIです。pgoはOperator Podに含まれる "apiserver" と通信することで，K8s上のPostgreSQLを制御します。例えば以下のような制御が行えます。

* PostgreSQLクラスター構築/削除
* Posgresインスタンスのスケールアウト/スケールイン
* Posgresインスタンスバックアップ/リストア

### 2-1-2. 事前準備
- 踏み台サーバー(Bastion Server)へのアクセス情報
- OpenShift4クラスターへのアクセス情報
- Postgres Operator がOpenShift4上で動作していること
- Postgres Operator PodがService(type=LoadBalancer)で公開されていること

>自身でクラスターを用意してハンズオンを実施される場合は，事前に以下を準備ください。
> - OpenShift4クラスター環境
> - ocコマンドのセットアップ
> - 利用ユーザーへのcluster-adminの権限付与
> - pgoコマンドのダウンロード
>   - Postgres Operatorのバージョンと一致したpgoを取得してください。(https://github.com/CrunchyData/postgres-operator/releases)

## 2-2. pgoクライアントの構成
pgoコマンドを使って，クライアント端末(踏み台サーバー)からOperator Podに含まれるapiserverコンテナに接続するための準備を行います。

- pgo実行時に使用する情報を格納するディレクトリを作成
- pgo実行時に使用する接続先情報(apiserverのURL)を指定
- pgo実行時に使用するクレデンシャルをOperator Podのapiserverから取得
- pgo実行時に使用するユーザー情報を作成
- 各変数のexport確認

### 2-2-1. pgo実行時に使用する情報を格納するディレクトリを作成
pgo実行時に使用する情報を格納するディレクトリ(`my-pgo-client`)を作成します。

```
$ cd $PGOROOT
$ mkdir $PGOROOT/my-pgo-client
```

>Tips:  
>
>**PGOROOTが未指定の場合のみ** 以下を実行します。  
>
>```
>cat <<EOF >> $HOME/.bashrc
>export PGOROOT=$HOME/postgres-operator
>EOF
>source $HOME/.bashrc
>```

### 2-2-2. pgo実行時に使用する接続先情報(apiserverのURL)を指定
pgo実行時に使用するapiserverのURL(`PGO_APISERVER_URL`)を確認します。

```
$ oc get svc -n pgo-<User_ID>

NAME                             TYPE           CLUSTER-IP       EXTERNAL-IP                                                                    PORT(S)                                         AGE
postgres-operator                LoadBalancer   172.30.114.68    a6615bd17b98011e992ee0e4cddef59e-1242048699.ap-northeast-1.elb.amazonaws.com   8443:32455/TCP                                  130m
```

>※注意: ワークショップ参加者の方は，必ず自身に割当てられた <User_ID> を使用し，`-n pgo-<User_ID>` のように指定してください。  

上記の実行例の結果の場合，`postgres-operator の EXTERNAL-IP` 欄 を確認して，以下の `PGO_APISERVER_URL` に指定します。
その際，`https://` と `:8443` を忘れずに付与しましょう。

```
$ export PGO_APISERVER_URL=https://a6615bd17b98011e992ee0e4cddef59e-1242048699.ap-northeast-1.elb.amazonaws.com:8443
```

.bashrcに "PGO_APISERVER_URL" を追記しておきます。

```
cat <<EOF >> $HOME/.bashrc
export PGO_APISERVER_URL=https://a6615bd17b98011e992ee0e4cddef59e-1242048699.ap-northeast-1.elb.amazonaws.com:8443
EOF
source $HOME/.bashrc
```

### 2-2-3. pgo実行時に使用するクレデンシャルをOperator Podのapiserverから取得
pgo実行時に使用するクレデンシャルをOperator Podのapiserverから取得します。

まず，Operator Pod名を確認しましょう。

```
$ oc get po -n pgo-<User_ID>

NAME                                 READY   STATUS    RESTARTS   AGE
postgres-operator-74c4fbf46c-r7llt   3/3     Running   0          135m
```

>※注意: ワークショップ参加者の方は，必ず自身に割当てられた <User_ID> を使用し，`-n pgo-<User_ID>` のように指定してください。  

上記の実行例の結果の場合，`postgres-operator-74c4fbf46c-r7llt` がPod名です。 

確認したPod名をコピーして以下の形式で指定します。

```
$ oc cp pgo-<User_ID>/<Pod名>:/tmp/server.key $PGOROOT/my-pgo-client/server.key -c apiserver
$ oc cp pgo-<User_ID>/<Pod名>:/tmp/server.crt $PGOROOT/my-pgo-client/server.crt -c apiserver
```

>実行例)
>
>```
>$ oc cp pgo-user18/postgres-operator-74c4fbf46c-r7llt:/tmp/server.key $PGOROOT/my-pgo-client/server.key -c apiserver
>$ oc cp pgo-user18/postgres-operator-74c4fbf46c-r7llt:/tmp/server.crt $PGOROOT/my-pgo-client/server.crt -c apiserver
>```

>Tips:  
>
>以下のようなWarningが出力されますが，今回は無視します。
>
>```
>tar: Removing leading `/' from member names
>```

鍵の所在を確認します。(`server.key`,`server.crt`)

```
$ ls $PGOROOT/my-pgo-client/

server.crt  server.key
```

.bashrcに 証明書関係のファイル群 を追記しておきます。

```
cat <<EOF >> $HOME/.bashrc
export PGO_CA_CERT=$PGOROOT/my-pgo-client/server.crt
export PGO_CLIENT_CERT=$PGOROOT/my-pgo-client/server.crt
export PGO_CLIENT_KEY=$PGOROOT/my-pgo-client/server.key
EOF
source $HOME/.bashrc
```

### 2-2-4. pgo実行時に使用するユーザー情報を作成
pgo実行時に使用するユーザー情報(PGOUSER)を作成します。

```
$ echo username:password > $PGOROOT/my-pgo-client/pgouser
$ chmod 700 $PGOROOT/my-pgo-client/pgouser
$ export PGOUSER=$PGOROOT/my-pgo-client/pgouser
```

.bashrcに "PGOUSER" を追記しておきます。

```
cat <<EOF >> $HOME/.bashrc
export PGOUSER=$PGOROOT/my-pgo-client/pgouser
EOF
source $HOME/.bashrc
```

### 2-2-5. 各変数のexport確認
これまでの手順でいくつかの変数をexportしてきました。  
これらの変数はpgoコマンド実行時に使用するため，正しくexportできているか確認しましょう。

```
$ cat $HOME/.bashrc
...
export PGOROOT=/home/user18/postgres-operator
export PGO_APISERVER_URL=https://af6626834cd8011e9a4e90a9978ec148-1958014071.ap-northeast-1.elb.amazonaws.com:8443
export PGO_CA_CERT=/home/user18/postgres-operator/my-pgo-client/server.crt
export PGO_CLIENT_CERT=/home/user18/postgres-operator/my-pgo-client/server.crt
export PGO_CLIENT_KEY=/home/user18/postgres-operator/my-pgo-client/server.key
export PGOUSER=/home/user18/postgres-operator/my-pgo-client/pgouser
...
```

上記のように，6つの変数がexportされていることを確認してから次のステップに進みましょう。

## 2-3. pgoからapiserverへの接続確認
正常にpgo(Client)からapiserver(Operator Pod)に接続できるか確認します。  

pgoコマンドを実行して接続確認します。

```
pgo version

pgo client version 4.0.1
pgo-apiserver version 4.0.1
```

上記のように出力されれば成功です。  

>Tips: 
>
>もしプロンプトが返ってこなかったり，Timeoutやエラーが返ってきた場合は，Operator Podにうまく接続できていません。
>以下を確認してみましょう。
>
> - Operator Podが正常に動作していること ([1-5-2](https://github.com/capsmalt/ocp4ws-infra/blob/master/Lab2/1_installtion-postgres-operator-pgo.md#1-5-2-operator-pod%E3%82%92%E7%A2%BA%E8%AA%8D))
> - Operator PodをService(type=LoadBalacer)で正常に公開できていること ([1-6)](https://github.com/capsmalt/ocp4ws-infra/blob/master/Lab2/1_installtion-postgres-operator-pgo.md#1-6-operator-pod%E3%81%AE%E5%85%AC%E9%96%8B))
> - 各変数が正しくexportされていること ([2-2-5](https://github.com/capsmalt/ocp4ws-infra/blob/master/Lab2/2_usage-pgo.md#2-2-5-%E5%90%84%E5%A4%89%E6%95%B0%E3%81%AE%E7%A2%BA%E8%AA%8D))
>

## 2-4. PostgreSQLリソース制御
pgoから様々なリソースを制御してみましょう。

### 2-4-1. PostgreSQLクラスターの作成
PostgreSQLクラスターを作成します。

```
pgo create cluster mycluster -n pgo-<User_ID>
pgo show cluster mycluster -n pgo-<User_ID>
```

Pgclusterリソースを確認します。

```
oc get Pgclusters -n pgo-<User_ID>

NAME        AGE
mycluster   17m
```

Postgres関連のPodを確認します。

```
oc get pods -n pgo-<User_ID>
    mycluster-6c5b4ddc6-qq5zg                         1/1     Running     0          5m13s
    mycluster-backrest-shared-repo-668554dc6c-mvbjg   1/1     Running     0          5m13s
    mycluster-stanza-create-bh6lf                     0/1     Completed   0          4m6s
    postgres-operator-9777dbc48-59kms                 3/3     Running     0          25m
```

Postgresの動作を確認します。

```
pgo test mycluster -n pgo-<User_ID>

    cluster : mycluster
	    psql -p 5432 -h 172.30.254.147 -U postgres postgres is Working
	    psql -p 5432 -h 172.30.254.147 -U postgres userdb is Working
	    psql -p 5432 -h 172.30.254.147 -U primaryuser postgres is Working
	    psql -p 5432 -h 172.30.254.147 -U primaryuser userdb is Working
	    psql -p 5432 -h 172.30.254.147 -U testuser postgres is Working
	    psql -p 5432 -h 172.30.254.147 -U testuser userdb is Working
```

Serviceを確認します。

```
oc get svc -n pgo-<User_ID>

NAME                             TYPE           CLUSTER-IP       EXTERNAL-IP                                                                   PORT(S)                                         AGE
mycluster                        ClusterIP      172.30.254.147   <none>                                                                        5432/TCP,9100/TCP,10000/TCP,2022/TCP,9187/TCP   7m49s
mycluster-backrest-shared-repo   ClusterIP      172.30.224.82    <none>                                                                        2022/TCP                                        7m49s
postgres-operator                LoadBalancer   172.30.77.27     a8a59cbf2b73d11e99d19066aaaf6145-115501653.ap-northeast-1.elb.amazonaws.com   8443:30861/TCP                                  26m
```

### 2-4-2. Postgresをスケーリング
PgreplicasリソースでPodを追加します。

```
pgo scale mycluster -n pgo-<User_ID>

WARNING: Are you sure? (yes/no): yes
created Pgreplica mycluster-hrbx
```

Pgreplicasリソースを確認します。

```
oc get Pgreplicas -n pgo-<User_ID>

NAME             AGE
mycluster-hrbx   8m45s
```

PostgresのReplica Podを確認します。

```
oc get pods -n pgo-<User_ID>
NAME                                              READY   STATUS      RESTARTS   AGE
mycluster-6c5b4ddc6-qq5zg                         1/1     Running     0          10m
mycluster-backrest-shared-repo-668554dc6c-mvbjg   1/1     Running     0          10m
mycluster-hrbx-7d9f8b569b-mvfb6                   1/1     Running     0          67s
mycluster-stanza-create-bh6lf                     0/1     Completed   0          9m46s
postgres-operator-9777dbc48-59kms                 3/3     Running     0          31m
```

### 2-4-3. その他

Postgresのバックアップを確認します。

```
pgo backup mycluster -n pgo-<User_ID>

created Pgtask backrest-backup-mycluster
```

```
oc get Pgtask -n pgo-<User_ID>

NAME                        AGE
backrest-backup-mycluster   9s
mycluster-createcluster     53m
mycluster-stanza-create     52m
```

```
pgo backup mycluster --backup-type=pgbasebackup -n pgo-<User_ID>

created backup Job for mycluster
workflow id 2176b3ad-9666-41bd-91df-081f911493f0
```

```
oc get Pgtask -n pgo-<User_ID>

NAME                        AGE
backrest-backup-mycluster   91s
mycluster-backupworkflow    5s
mycluster-createcluster     54m
mycluster-stanza-create     53m
```

```
oc get pods -n pgo-<User_ID>

NAME                                              READY   STATUS              RESTARTS   AGE
backrest-backup-mycluster-6mmcg                   0/1     Completed           0          103s
backup-mycluster-cqdj-qwzl7                       0/1     ContainerCreating   0          14s
mycluster-6c5b4ddc6-qq5zg                         1/1     Running             0          54m
mycluster-backrest-shared-repo-668554dc6c-mvbjg   1/1     Running             0          54m
mycluster-hrbx-7d9f8b569b-mvfb6                   1/1     Running             0          45m
mycluster-stanza-create-bh6lf                     0/1     Completed           0          53m
postgres-operator-9777dbc48-59kms                 3/3     Running             0          75m
```

```
oc get pods -n pgo-<User_ID>

NAME                                              READY   STATUS      RESTARTS   AGE
backrest-backup-mycluster-6mmcg                   0/1     Completed   0          2m21s
backup-mycluster-cqdj-qwzl7                       0/1     Completed   0          52s
mycluster-6c5b4ddc6-qq5zg                         1/1     Running     0          55m
mycluster-backrest-shared-repo-668554dc6c-mvbjg   1/1     Running     0          55m
mycluster-hrbx-7d9f8b569b-mvfb6                   1/1     Running     0          45m
mycluster-stanza-create-bh6lf                     0/1     Completed   0          54m
postgres-operator-9777dbc48-59kms                 3/3     Running     0          75m
```

ログを確認します。

```
pgo ls mycluster -n pgo-<User_ID> /pgdata/mycluster/pg_log

total 60K
-rw-------. 1 postgres root 53K Aug  5 05:28 postgresql-Mon.log


pgo cat mycluster -n pgo-<User_ID> /pgdata/mycluster/pg_log/postgresql-Mon.log | tail -3

2019-08-05 05:29:38 UTC [1022]: [3-1] user=postgres,db=postgres,app=psql,client=[local]LOG:  duration: 0.279 ms
2019-08-05 05:29:38 UTC [1022]: [4-1] user=postgres,db=postgres,app=psql,client=[local]LOG:  disconnection: session time: 0:00:00.002 user=postgres database=postgres host=[local]
```

PVCに対するPostgresの利用状況を確認します。

```
pgo df mycluster -n pgo-<User_ID>

POD                       STATUS    PGSIZE    CAPACITY  PCTUSED

mycluster-6c5b4ddc6-qq5zg up        30 MB     1Gi       2
```

以下の公式ドキュメントを参考にして，他にも試してみましょう。

https://access.crunchydata.com/documentation/postgres-operator/4.0.1/operatorcli/pgo-overview/


