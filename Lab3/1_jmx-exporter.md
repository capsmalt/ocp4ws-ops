
# 1. Prometheus JMX Exporterの展開  

## 1-1. 諸注意

### 1-1-1. JMX Exporterについて

* JMX Exporter は2通りの動作を提供する  
Java Agent (推奨): Java Agent 用 JAR ファイルからメトリクスを収集   
HTTP server: リモートの JMX ターゲットからメトリクスを取得し、HTTP サーバで公開する  
* Java Agent 用 JARファイルの配置方法は次の2通り  
Docker Build Strategy: Dockerfile にビルド済み JAR ファイルを取得し配置する  
S2I: pom.xml に Dependency をセットし、S2I 実行時に Maven でビルドする  

### 1-1-2. 事前準備
- 踏み台サーバー(Bastion Server)へのアクセス情報
- OpenShift4クラスターへのアクセス情報

>自身でハンズオンを実施される場合は，事前に以下を準備ください。
> - OpenShift4クラスター環境
> - ocコマンドのセットアップ
> - 利用ユーザーへのcluster-adminの権限付与

## 1-2. アプリケーション展開

### 1-2-1. OpenShift4へのログイン  
1. 踏み台サーバー(Bastion Server)にSSHでログインします。
    ```
    $ ssh <Bastion_User_ID>@<Bastion_Server_Hostname>
    ```

    >**※注意: ワークショップ参加者の方は，必ず自身に割当てられた <Bastion_User_ID>, <Bastion_Servier_IP>，<Password> を使用してください。**  
    >
    >
    >例) 「踏み台サーバー(Bastion Server)」のSSHログイン情報
    > - `<Bastion_User_ID>`: **lab-user**
    > - `<Bastion_Server_IP>`: **bastion.tokyo-XXXX.sandboxYYYY.opentlc.com**
    > - `<Password>`: **r3dh4t1!**
    >
    >実行例) 
    >```
    >$ ssh lab-user@bastion.tokyo-XXXX.sandboxYYYY.opentlc.com
    >lab-user@bastion.tokyo-004e.sandbox104.opentlc.com's password: r3dh4t1!(パスワードは表示されません)
    >```

2. OpenShift4クラスターにocコマンドでログインします。

    ```
    $ oc login <OpenShift_API>

    Username: "<User_ID>" を入力
    Password: "<User_PW>" を入力
    ```

    >**※注意: ワークショップ参加者の方は，必ず自身に割当てられた <OpenShift_API>，<User_ID>，<User_PW> を使用してください。**  
    >
    >
    >例) 「OpenShift_API」へのログイン情報
    > - `<OpenShift_API>`: **https://api.cluster-tokyo-XXXX.tokyo-XXXX.sandboxYYYY.opentlc.com:6443**
    > - `<User_ID>`: **kubeadmin**
    > - `<User_PW>`: **XXXXX-XXXXX-XXXXX-XXXXX**
    >
    >実行例) 
    >```
    >$ oc login https://api.cluster-tokyo-XXXX.tokyo-XXXX.sandboxYYYY.opentlc.com:6443
    >Username: kubeadmin
    >Password: XXXXX-XXXXX-XXXXX-XXXXX(パスワードは表示されません)
    >```   

3. 次に，OpenShift4クラスターを構成するノードを確認します。  
       
    ```
    $ oc get node  
    
    NAME                                              STATUS   ROLES    AGE   VERSION
    ip-10-0-128-108.ap-northeast-1.compute.internal   Ready    master   15h   v1.13.4+509f0153f
    ip-10-0-141-52.ap-northeast-1.compute.internal    Ready    worker   15h   v1.13.4+509f0153f
    ip-10-0-151-196.ap-northeast-1.compute.internal   Ready    worker   15h   v1.13.4+509f0153f
    ip-10-0-159-143.ap-northeast-1.compute.internal   Ready    master   15h   v1.13.4+509f0153f
    ip-10-0-162-88.ap-northeast-1.compute.internal    Ready    master   15h   v1.13.4+509f0153f
    ip-10-0-175-15.ap-northeast-1.compute.internal    Ready    worker   15h   v1.13.4+509f0153f
    ```
    
    >上記のように，複数台のMasterとWorkerノードで構成されており，STATUSが Readyであることを確認します。
    >なお，ハンズオン環境においては，ノード台数が異なる場合があります。

### 1-2-2. アプリケーションビルド  
1. 監視対象アプリケーション用の「jmx」という名前のプロジェクトを作ります。

    ```
    $ oc new-project jmx
    $ oc project
    Using project "jmx" on server "https://<OpenShift API>".
    ```
    >
    >
    >実行例)
    >
    >```
    >$ oc new-project jmx 
    >$ oc get project | grep jmx
    >
    >jmx               Active
    >```
    >
    >上記のように，作成したプロジェクト名が出力されることを確認します。


1. アプリケーションをリポジトリからCloneして，「jboss-eap-prometheus」イメージをビルドします。

    ```
    $ git clone https://github.com/openlab-red/jboss-eap-prometheus
    $ cd ./jboss-eap-prometheus/
    $ oc new-build .
    --> Found Docker image b72b49b (18 months old) from registry.access.redhat.com for "registry.access.redhat.com/jboss-eap-7/eap70-openshift:latest"
    …
    --> Success
    ```

2. ビルドの状況をocコマンドと、OpenShift4コンソールからも確認します。

    ```
    $ oc logs -f bc/jboss-eap-prometheus
    …
    Writing manifest to image destination
    Storing signatures
    Push successful
    ※イメージがPushされると自動的にログから開放されるので待つ。
    (もし「Errorとなってしまった場合は」、[Ctl] + [C]で出て再度やり直す)

    $ oc get build
    NAME                     TYPE     FROM          STATUS     STARTED          DURATION
    jboss-eap-prometheus-1   Docker   Git@23160b8   Complete   38 minutes ago   1m28s    
    
    $ oc get imagestream
    NAME                   IMAGE REPOSITORY                                                                   TAGS     UPDATED
    eap70-openshift        image-registry.openshift-image-registry.svc:5000/jmx/eap70-openshift        latest   37 minutes ago
    jboss-eap-prometheus   image-registry.openshift-image-registry.svc:5000/jmx/jboss-eap-prometheus   latest   36 minutes ago
    ```

    OpenShift4コンソールにログインして，[Builds]>[Image Streams]から，ビルドしたイメージがImageStreamに登録されていることも確認しましょう。

    ![](images/ocp4-i-lab1-1-imagestream-jboss.png)

### 1-2-3. アプリケーションデプロイ  

1. アプリケーションの展開

    ここでは，登録した「jboss-eap-prometheus」を利用して，アプリケーションを展開します。  
    展開の際には，Java Agent用JARファイルやJMX Exporter設定ファイルのパスを環境変数(jmx-prometheus.jar=9404)で指定しておきましょう。

    ```
    $ export JBOSS_HOME=/opt/eap
    $ oc new-app -i jboss-eap-prometheus:latest \
      --name=jboss-eap-prometheus \
      -e PREPEND_JAVA_OPTS="-javaagent:${JBOSS_HOME}/prometheus/jmx-prometheus.jar=9404:${JBOSS_HOME}/prometheus/config.yaml"

    上記を `\` で改行しながら1コマンドとして実行します。
    
    --> Found image 55806df (About a minute old) in image stream "jmx/jboss-eap-prometheus" under tag "latest" for "jboss-eap-prometheus:latest"
    …
    --> Success
        Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
         'oc expose svc/jboss-eap-prometheus'
        Run 'oc status' to view your app.
    ```

2. 展開したアプリケーションの確認

    この時点で「jboss-eap-prometheus-1」がRunning状態になれば，デプロイ成功です。  
    JMX Exporter はデフォルトで9404ポートを公開します。

```
$ oc get svc/jboss-eap-prometheus
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                               AGE
jboss-eap-prometheus   ClusterIP   172.30.159.173   <none>        8080/TCP,8443/TCP,8778/TCP,9404/TCP   30s

$ oc get deploy jboss-eap-prometheus
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
jboss-eap-prometheus   1/1     1            1           47s   

$ oc get pod
NAME                                    READY   STATUS      RESTARTS   AGE
jboss-eap-prometheus-1-build            0/1     Completed   0          6m50s
jboss-eap-prometheus-7b78f9bc88-b98zx   1/1     Running     0          77s
```

「jboss-eap-prometheus-7b78f9bc88-b98zx」(7b78f9bc88-b98zxはデプロイしたときにランダムに生成される)がRunning状態になるまで待ちましょう。

### 1-2-4. アプリケーションのアノテーション設定

JMX ExporterのServiceに対して、アノテーションをつけておく。   

```
$ oc annotate svc jboss-eap-prometheus prometheus.io/scrape='true'
service/jboss-eap-prometheus annotated

$ oc annotate svc jboss-eap-prometheus prometheus.io/port='9404'
service/jboss-eap-prometheus annotated
```

### 1-2-5. アプリケーションのルータ設定 

「jboss-eap-prometheus」のアプリケーション(tcp-8080)ポートを、ルータに接続。

```
$ oc expose svc/jboss-eap-prometheus --name=tcp-8080 --port=8080
route.route.openshift.io/tcp-8080 exposed

$ oc get route tcp-8080
NAME       HOST/PORT                                PATH   SERVICES               PORT   TERMINATION   WILDCARD
tcp-8080   tcp-8080-jmx.XXX.openshiftworkshop.com          jboss-eap-prometheus   8080                 None
```

Host/Port (http://tcp-8080-jmx.XXXX.openshiftworkshop.com) をブラウザ上からアクセスすると、アプリケーションコンテンツが確認できる。   
    
![Jboss Application](images/jboss-eap-prometheus-8080.jpg "jboss-eap-prometheus-8080")

次に「jboss-eap-prometheus」のPromtheus Exporter(tcp-9404)ポートを、ルータに接続。

```
$ oc expose svc/jboss-eap-prometheus --name=tcp-9404 --port=9404
route.route.openshift.io/tcp-9404 exposed

$ oc get route tcp-9404
NAME       HOST/PORT                                PATH   SERVICES               PORT   TERMINATION   WILDCARD
tcp-9404   tcp-9404-jmx.XXX.openshiftworkshop.com          jboss-eap-prometheus   9404                 None
```

Host/Port (tcp-9404-jmx.XXXX.openshiftworkshop.com) をブラウザ上からアクセスすると、JMX Exporterから取得したPromSQLのクエリが確認できる。   
    
![Jboss Application](images/jboss-eap-prometheus-9404.jpg "jboss-eap-prometheus-9404")

これで、JMX Exporterの設定は完了。次に[Prometheus Operator](2_PrometheusOperator.md)の作業に進む   

