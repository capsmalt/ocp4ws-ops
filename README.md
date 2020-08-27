# ハンズオン概要
本ハンズオンは，OpenShift4のOps編です。  

以下を学びます。
- OpenShift4クラスターへのログインと動作確認
- コンテナイメージのビルドとデプロイ
- Jenkinsベースのビルドパイプラインの使用
- Operatorの導入
- Operatorとアプリケーションの連携
- Custom Resourceの設定
- Operatorに対するCLI操作

### Lab1: [コンテナイメージのビルド&デプロイ](Lab1)
### Lab2: [Jenkinsベースのビルドパイプラインの使用](Lab2)
### Lab3: [アプリケーション と Operator 連携](Lab3)
### (Option) Lab4: [Operator導入 と CLI活用](Lab4)

# ハンズオン環境
本ハンズオンは，Kubernetesクラスター(OpenShift4)の動作環境としてAWSを使用します。今回は構築済です。  

OpenShift4クラスターに対するCLI操作をを行う際は，クライアントPCから，踏み台サーバー(Bastion Server)にSSH接続し，**ocコマンド** を使って制御します。  
`クライントPC <--SSH--> 踏み台サーバー <--oc--> OpenShift4クラスター`

![](images/handson-env.png)

**本日は，ひとり1クラスター（6台構成+踏台）の環境を使用します。**

GUI操作は，クライアントPCのブラウザ(**Chrome/Firefox推奨**)を使用します。  

# 前提
- SSH用ツール (EC2への接続する際に使用します)
- ブラウザ (Google Chrome or Firefox)
- ブラウザでの接続テスト
  - 下記リンクにブラウザからアクセスし，メモ帳アプリに正常アクセスできるか確認してください。
  - http://bit.ly/test0828
  - **接続できない環境の場合はハンズオンを実施できない場合があります (サポートが必要な場合は担当者まで連絡ください)**

# 注意事項
OpenShift4クラスター接続情報など当日の連絡事項 (Etherpad) ==> https://tinyurl.com/ops-etherpad
