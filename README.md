# DockerでPython実行環境を作ってみる

使っているパソコンを変えても、開発環境を揃えたい時はDockerを使うと便利。ということでDockerでPython環境を作って色々なところで使いまわせるようにします。

## Dockerとは

Dockerとはシステム開発や運用に最近よく使われるコンテナ技術を提供するサービスの一つです。コンテナとは、アプリケーションの実行に必要な環境をパッケージ化して、いつでもどこからでも実行するための仕組みです。自分のコンピュータの環境を汚すことなく、隔離された環境を作ってそこでプログラムを動かすことができるのでトライアンドエラーも簡単で、その作った環境はシェアすることでどこでも実行できるという点がメリットです。個人的に、M1 Macに最近したのですが、その独自な仕様により今までのIntel Macで使えていたPythonモジュールが使えなくなったりということがあったので、Dockerのコンテナ内でなら過去のプログラムも実行できることがわかり、必要に駆られてこれを使ってみることにしました。


## Step 1. Dockerのインストール

環境に合ったDockerをインストールします。ここではDocker Desktopを利用することを想定します。M1用のインストーラーもあります。インストール後はTerminalなどでdocker composeコマンドが使えるようになっていることを確認します。

<https://www.docker.com/products/docker-desktop>

インストールしたらDocker Desktopを起動してください。


## Step 2. Dockerの設定

### ファイル構成

次のようなファイル構成をまず作ってください。docker-pythonと書いてあるフォルダ名は好きな名前にして大丈夫です。Dockerfileとdocker-compose.ymlとsample.pyはテキストデータです。

```
docker-python/
  ├ Dockerfile
  ├ docker-compose.yml
  └ opt/
    └ sample.py

```

### Dockerfile

Dockerfileに次のように記述します。ここでは利用する開発環境を指定し、コンテナ作成時に先にインストールしておきたいOS用のライブラリや、今回のようにPythonを使いたい場合は使いたいPythonのモジュールなどをインストールします。

ちなみにDockerfileで指定せず、後から自分で追加でインストールすることも可能です。ただ、作られたコンテナを削除して、再度作り直す場合はDockerfileを再度使ってライブラリ等をインストールするので、コンテナを作成するたびに欲しいライブラリ等はここで指定しておくのがいいと思います。

最初の行でFROM python:3とあるのは、Dockerが公式で用意しているPythonのコンテナをベースとして読み込んでいるということになります。詳しくは[こちら](https://hub.docker.com/_/python)から。デフォルトで使われる環境はLinuxのDebianのようです。


```docker
FROM python:3
USER root

RUN apt-get update
RUN apt-get -y install locales && \
    localedef -f UTF-8 -i ja_JP ja_JP.UTF-8
ENV LANG ja_JP.UTF-8
ENV LANGUAGE ja_JP:ja
ENV LC_ALL ja_JP.UTF-8
ENV TZ JST-9
ENV TERM xterm

RUN apt-get install -y vim less
RUN pip install --upgrade pip
RUN pip install --upgrade setuptools

RUN python -m pip install jupyterlab
```

この時点で使いたいPythonのモジュールが既に決まっていれば、`RUN python -m pip install requests`のようにモジュールのインストールコマンドを追記していきます。


### docker-compose.yml

次にdocker-compose.ymlの中身を書きます。ここでは作成するコンテナの情報を書いていきます。バージョンやコンテナの名前を好きに決めます。working_dirというのはコンテナの中に入った直後の作業フォルダのことで、volumesは自分のコンピュータのどのフォルダとコンテナの中の環境のフォルダと同期するための設定です。ここでは、先に作ったoptという名前のフォルダが、コンテナ上のroot/optフォルダと同期されます。これにより、自分のコンピュータで作ったデータをコンテナ上の環境で読むことができるようになります。

```yaml
version: '3'
services:
  python3:
    restart: always
    build: .
    container_name: 'python3'
    working_dir: '/root/'
    tty: true
    volumes:
      - ./opt:/root/opt
```

### sample.py

このファイルには実行したいPythonのプログラムを記述します。
テスト用のものなので、簡単なコードを書いておきます。このPythonファイルを実行する時に、一緒の渡される引数（argument)をdegreesからradiansに変更してターミナルにプリントする簡単な内容です。

```python
import math
import sys

def main():
  val = float(sys.argv[1])
  print(math.radians(val))

if __name__ == "__main__":
  main()
```

## Step 3. Dockerイメージの作成、コンテナのビルド、そしてコンテナの起動

ターミナル（Mac）かGit bashやPowerShell（Win）で次のようにコマンドを打ち、docker-pythonフォルダを作業フォルダにした上でDockerイメージ（仮想環境のテンプレート）の作成し、そのイメージを利用してDockerのコンテナ（テンプレートを利用して作られ実際に実行される仮想環境が入った入れもの）を起動します。Dockerfileとdocker-compose.ymlを自動的に参照しDockerのイメージが作成されます。

このコマンドによりイメージ作成→コンテナ作成→コンテナ起動となりますが現状はまだコンテナの環境はバックグラウンドで走っている状態です。

```bash
$ cd docker-python/
$ docker compose up -d --build
```

## Step 4. 作られたイメージとコンテナの確認

では実際に作られたDockerイメージとコンテナをかくにんしてみます。次のコマンドを打ち、まず現在自分の環境で利用できるイメージのリストを取得します。

```bash
$ docker image ls
```

次のようなメッセージが出ていたら成功です。ここではdocker-python_python3というのが今作ったDockerのイメージの名前となります。コンテナを作るためのテンプレートの名前、みたいに覚えるとわかりやすいかもしれません。

```bash
REPOSITORY                                 TAG       IMAGE ID       CREATED       SIZE
docker-python_python3                      latest    fdca699ff626   13 days ago   1.14GB
```

次に次のコマンドをうち、現在走っているコンテナのリストを取得します。

```bash
$ docker container ls
```

次のようなメッセージが出てくるかと思います。

```bash
CONTAINER ID   IMAGE          COMMAND     CREATED          STATUS          PORTS     NAMES
05a6dee0f4d0   fdca699ff626   "python3"   45 minutes ago   Up 45 minutes             python3
```


## Step 5. コンテナへの接続

コンテナが走っていることを確認したところで、次のコマンドでコンテナへ接続（コンテナの中の環境に入ってその環境下でコマンドを打てるように）します。python3というのはdocker-compose.ymlで指定したコンテナの名前です。

```bash
$ docker compose exec python3 bash
```

これで次からターミナルで打つコマンドはコンテナ内の環境下で実行されるようになります。


## Step 6. Python用のライブラリをインストールする


コンテナに接続できたら、まずPythonのバージョンを確認してみます。

```bash
$ python --version
```

もし使いたいPythonのモジュールがあればこの時点でインストールしておきます。コンテナ作成時にインストールされている状態にしたければ、Dockerfileにそのインストール用のコマンドを追記しておいてください。コンテナに接続してからインストールされたモジュールは、コンテナとの接続を切り、コンテナを削除した時点で消えるのでテンポラリーなものだと思ってください。

個人的におすすめな方法としては、一度テンポラリーにモジュールをインストールして使ってみて、問題なければDockerfileに追記してコンテナ作成時にモジュールが自動インストールされるようにするという流れがいいと思います。

```bash
$ python -m pip install numpy
```

## Step 7. 自分のコンピュータ上のPythonファイルを走らせてみる

これでPythonの実行環境が作れたので、実際に使ってみましょう。その方法のひとつとして、自分のコンピュータ上にPythonで記述されたテキストファイル（.pyファイル）を作って、それをコンテナ内の環境のPythonで実行しようというものです。

まずdocker-pythonフォルダのoptフォルダにsample.pyがあることを確認した上で、Dockerのコンテナに接続されている状態で次のコマンドを打ち、現在の作業ディレクトリにあるファイルの一覧を取得します。

```bash
$ ls
```

するとsample.pyというファイルが一覧に出てくると思います。これで、自分のコンピュータ上のoptというフォルダと、コンテナ内の環境上のフォルダが同期されていることが確認できたことになります。このフォルダを介してコンテナ内の環境から自分のパソコンのファイルにアクセスができるようになるという寸法です。

ファイルの存在が確認できたところで、コンテナ内の環境にインストールされたPythonでoptフォルダの中のこのPythonファイルを走らせてみましょう。

```bash
$ python sample.py 180.0
```

結果として3.141592653589793のように円周率の値が出てきたら成功です。


## Step 8. コンテナの削除

コンテナを使い終わったらコンテナとの接続を切り、いらなくなったコンテナを削除します。まず次のコマンドでコンテナとの接続を切ります。

```bash
$ exit
```

その上で次のコマンドでDockerのコンテナを終了し、削除します。

```bash
$ docker compose down
```

次のようなメッセージが出ると思います。

```bash
[+] Running 2/2
 ⠿ Container python3              Removed                                                                           10.4s
 ⠿ Network docker-python_default  Removed    
```

この後に次のようにコマンドを打ち、コンテナのリストから作ったコンテナが消えていたら無事削除できたことになります。

```bash
$ docker container ls
```

## Step 9. 再度コンテナを起動したい場合は

コンテナはテンプレートであるイメージの中身をコピーして作られるインスタンスのようなものなので、再度コンテナを起動したい場合はStep 3でやったように再度イメージを作り直してそのイメージからコンテナを作成して起動します。あるいはすでにDockerfileとdocker-compose.ymlでイメージを作ってある場合は、--buildオプションを外して次のようなコマンドを入力してすでに作られているDockerイメージからコンテナを作成して起動することができます。

```bash
$ docker compose up -d
```


## Step 10. いらないイメージの削除

もしビルドしたイメージもいらな場合は削除します。そのために、先にイメージのリストを取得します。

```bash
$ docker image ls
```

次のようなコマンドでIDを指定して、いらないイメージを削除します。*imageid*には自分のイメージのIDに対応するものをいれてください。

```bash
$ docker image rm imageid
```


## Step 11. Jupyter Notebookでウェブブラウザ経由でPythonを使ってみる

Pythonのコードをテストしたい時、特に科学計算などにおいては時にJupyter Notebookのようなブラウザ経由でインタラクティブにPythonコードを走らせることができるツールを使いたいことがあるかもしれません。そのためのステップも載せておきます。ちなみに先に書いたDockerfileにはすでにjupyternotebookがインストールされるようなコマンドが書かれています。

先に、Dockerのイメージを削除した場合はイメージを作り直します。ただ今回はコンテナの中に入らず、自分のコンピュータのブラウザからコンテナ内の環境にアクセスすることになるので、次のようなコマンドを使ってイメージだけ作成します。その時ターミナルでは、コマンド入力先が自分のコンピュータ上の作業フォルダになっていることを確認しておいてください。

```bash
$ docker compose build
```

これで次のコマンドを入力してイメージがリストアップされていればOKです。

```bash
$ docker image ls
```

イメージがあることが確認できたら、次のコマンドでイメージから一時的にコンテナを起動し、起動直後にJupyter Notebookを利用するた目のサーバーをコンテナ内の環境下で立ち上げます。ちなみに$PWDというのはコマンドを入力しようとしているターミナルで現在の作業フォルダの場所を示しています。

```bash
$ docker run -v $PWD/opt:/root/opt -w /root/opt -it --rm -p 7777:8888 docker-python_python3 jupyter-lab --ip 0.0.0.0 --allow-root -b localhost
```

すると成功すれば次のようなメッセージが出ると思います。

```bash
    To access the server, open this file in a browser:
        file:///root/.local/share/jupyter/runtime/jpserver-1-open.html
    Or copy and paste one of these URLs:
        http://46102976db71:8888/lab?token=xxxxxxxxxx
        http://127.0.0.1:8888/lab?token=xxxxxxxxxx

```

Jupyter Notebookを使うためのサーバーがコンテナ内の環境で無事立ち上がりました。ただ、メッセージにはhttp://127.0.0.1:8888 にアクセスしろと書いてありますが、これはコンテナの環境の中で使えるアドレスで、コンテナ環境の外である自分のコンピュータ環境からはこのアドレスにはアクセスできません。アクセスするためには、8888のポートの代わりにコマンドで指定した7777というポートを利用します。コマンドでやっているのはつまりコンテナの環境内で使える8888というポートを自分のコンピュータで使えるように7777というポートにマッピングしたということになります。コマンドの7777の部分は好きな数値に変えてもらって大丈夫です。

ではウェブブラウザを開き、URL欄に`http://127.0.0.1:7777`と入力してJupyter Notebookのサーバーにアクセスしてみましょう。

するとToken authentication is enabledというページが出ると思うので、そこに上のメッセージでtoken=に続くコードをコピーしてPassword or tokenの入力欄に入力してLog inボタンを押します。

うまくいけばJupyter Notebookにログインされ、無事使えるようになります。

もしブラウザ経由ではなく、`docker run`コマンドで立ち上げたコンテナに接続したい場合は、別のターミナルウィンドウで次のコマンドで接続します。この時*docker-container-id*には`docker container ls`で確認した現在起動しているコンテナのIDを入力します。

```bash
$ docker exec -it *docker-container-id* bash
```

サーバーを止めたい場合は、サーバーを起動したターミナルで Controlキー + C を入力します。すると次のようなメッセージが出ます。

```bash
Shutdown this Jupyter server (y/[n])? 
```

ターミナルに`y`と入力してEnterキーを押しましょう。サーバーが止まり、同時に一時的に作られていたコンテナも削除されます。接続解除とともにコンテナが削除されたのは、`docker run`コマンドを利用したとき`-rm`というオプションを使ったからです。このオプションがないと、接続を解除した時にコンテナが停止した状態で削除されずに残ります。この時、停止されたコンテナは`docker container ls`では表示されなくなるので、確認したい場合は`docker container ls -a`で停止したコンテナも全て表示するようにします。


##　Step 12. 停止したコンテナを削除する

停止したコンテナそのままにしているとずっと残り続けます。`docker start`コマンドで停止したコンテナを起動させ再利用することも可能ですが、削除したい場合は次のようなステップを踏みます。

まず次のようなコマンドを入力して停止しているコンテナ含めて全てのコンテナを表示します。

```bash
$ docker container ls -a
```

次のように表示されたら、その中から消したいコンテナのIDを確認します。この場合`5e00e61e8717`がそれに当たります。この値はコンテナを作り直すたびに変わります。

```bash
CONTAINER ID   IMAGE                                      COMMAND                  CREATED          STATUS                     PORTS     NAMES
5e00e61e8717   docker-python_python3                      "jupyter-lab --ip 0.…"   11 seconds ago   Exited (0) 6 seconds ago             relaxed_wiles
```

削除したいコンテナのIDが確認できたら、次のコマンドで指定のコンテナを削除します。*container-id*は削除したいコンテナのIDと入れ替えてください。

```bash
$ docker container rm container-id
```


## おわりに

今回はDockerが何かを理解するためにPythonの環境を試しに作って使ってみました。少しは可能性を感じていただけましたでしょうか？プログラムの実行環境をこのように手軽に作れるのは非常に便利だと思います。積極的に使っていきましょう。

以上です。お疲れ様でした。



参考サイト
[dockerで簡易にpython3の環境を作ってみる](https://qiita.com/reflet/items/4b3f91661a54ec70a7dc)