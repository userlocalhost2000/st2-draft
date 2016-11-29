## ワークフローを記述する
　本章の最後にワークフロー (Workflow) の設定について解説します。  
　冒頭で、アクションはひとつの目的に特化したプログラムコードで、ワークフローはそれらを組み合わせたものだと解説しました。通常、運用を自動化しようとした場合、ある入力イベントに対してひとつのアクションの実行で済むということはありません。  
　例えば DMM.com ラボの場合 [仮想基盤に VMWare を使い](http://news.mynavi.jp/news/2016/04/12/052/)、リソース管理に [Racktables を使っています](http://tsuchinoko.dmmlabs.com/?p=886)。また、タスクやドキュメント管理には [Atlassian のソリューションを使っています](https://seleck.cc/297)。この他にも LB やネットワーク機器など、様々な機器・サービスを跨いだ運用作業を行っています。  

　ワークフローは、こうした様々なサービスに対するアクションを組み合わせて実行させることで、一連の運用作業を自動化させることができます。更に StackStorm では入力やアクションの結果に応じて処理の流れを制御させることで、より柔軟なワークフローを定義することができます。  

### ワークフローの記述
　StackStorm はワークフローの記述に２つの方法を提供しています。ひとつは、シンプルなワークフローを簡単に記述できる StackStorm のワークフローサービス [ActionChain](https://docs.stackstorm.com/actionchain.html) を利用する方法。もうひとつは、より複雑で柔軟なワークフローを記述できる [OpenStack](http://docs.openstack.org/) のワークフローサービス [Mistral](http://docs.openstack.org/developer/mistral/overview.html) を利用する方法です。  

　ここでは前者の ActionChain を利用します。なお、作成するワークフローのソースコードは以下のリポジトリで公開しています。  
* [https://github.com/userlocalhost2000/st2-workflow-example](https://github.com/userlocalhost2000/st2-workflow-example)  
　  
　例として、以下に示すような VM を作成してからプロビジョニングを行い、サービスインするまでのワークフローを記述します。  

![ワークフロー (setup-vm) の例](image3.png)
 
　ActionChain は、具体的な処理のフローを定義する ActionChain 定義ファイルと、ActionChain に渡すパラメータなどを記述するメタデータファイルの二つから構成されます。  
　以下は、上記のワークフローを実現する 定義ファイルとメタデータファイルになります。  

```yaml
### ワークフローのメタデータファイル
---
name: "setup_vm"
description: "An example to automate VM setup operation"
runner_type: "action-chain"
enabled: true

entry_point: "workflows/setup_vm.yaml"

parameters:
    name:
        type: "string"
        required: true
    ipaddr:
        type: "string"
        default: ""
    domain:
        type: "string"
        default: "dmm.local"
    vcpu:
        type: "integer"
        default: 1
    ram:
        type: "integer"
        default: 1 # [GB]
    storage:
        type: "integer"
        default: 100 # [GB]
```

```yaml
### ワークフローの定義ファイル
---
default: create-vm
chain:
    - name: create-vm
      ref: core.local
      parameters:
          cmd: "echo '[create-vm] {{name}}'"
      on-success: provision-vm

    - name: provision-vm
      ref: core.local
      parameters:
          cmd: "echo '[provision-vm] {{action_context}}'"
      on-success: register-dns
      on-failure: unregister-dns

    - name: register-dns
      ref: core.local
      parameters:
          cmd: "echo '[register-dns] {{name}}'"
      on-success: register-lb
      on-failure: unregister-dns

    - name: register-lb
      ref: core.local
      parameters:
          cmd: "echo '[register-lb] {{name}}'"
      on-failure: unregister-dns

    - name: unregister-dns
      ref: core.local
      parameters:
          cmd: "echo '[unregister-dns] {{name}}'"
      on-success: clear-vm

    - name: clear-vm
      ref: core.local
      parameters:
          cmd: "echo '[clear-vm] {{name}}'"
```

　上記のように、それぞれ YAML 形式でワークフローを記述します。メタデータファイルでは、ワークフロー定義ファイルのパス `entry_point` と、ワークフローに渡すパラメータ `parameters` を規定しています。  
　またワークフロー定義では、図に示した流れでアクションを実行する設定を記述しています。各アクションの `on-success` パラメータは、実行したアクションが正常終了した場合に実行する次のアクションを指し、`on-failure` パラメータは逆に異常終了した場合に実行する次のアクションを指します。正常終了 / 異常終了の判定は、StackStorm のアクションの実行環境 ([ActionRunner](https://docs.stackstorm.com/actions.html#action-runner)) 毎に異なりますが、ここで指定しているアクション `core.local` の ActionRunner `local-shell-cmd` の場合はコマンドの終了ステータスコードで行います。`0` が返ってきた場合には正常終了、それ以外は異常終了と判定します。  
　なお、ここでは各アクションの実行コマンドはそれぞれ、何も実行しないで正常終了するダミーコマンドを用います。  

### 動かしてみる

　それでは、上記のワークフローを実際に動かしてみます。作成したワークフロー定義ファイル、及びメタデータファイルを StackStorm に登録し、ワークフロー中で実行するダミーのコマンドをシステムにデプロイします。面倒なので、これらの処理を記述した Makefile を実行します。  

```
$ git clone https://github.com/userlocalhost2000/st2-workflow-example.git
$ cd st2-workflow-example
$ make
```

　それではワークフロー 'default.setup-vm' を実行します。適当なトリガと紐付けたルールを作成し、イベントを発生させるでもいいですが、面倒なので以下のように `st2 run` コマンドでアクション (ワークフロー) を単体実行させます。  

```
$ st2 run default.setup-vm name=hoge
```

　`name=hoge` はメタデータで記述した、ワークフローに渡すパラメータになります。これらの値はワークフロー内部のアクションで参照されます。以下は上記コマンドの実行結果になります。  

```
(実行結果)
```

　全てのコマンドが正常終了したので、図の左側 4 つのアクション (`create-vm`, `provision-vm`, `register-dns`, `register-lb`) だけが実行されました。  

　ここでアクション `register-lb` で実行するコマンド `register-lb.sh` が異常終了するよう、以下の修正を行います。  

```diff
(修正結果)
```

　そして、先程と同様にワークフローを実行してみてください。  

```
(実行結果)
```

　結果、図の右側のフォールバックアクション (`unregister-dns`, `clear-vm`) が実行されることが確認できました。  