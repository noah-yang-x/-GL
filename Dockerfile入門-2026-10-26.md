# Dockerfile 入門

`Dockerfile` は、イメージを作る手順を書いたファイルです。Docker は上から読み、1行が1つの命令です。

出来上がるのはイメージです。中には、システム、ファイル、環境変数、起動するコマンドが入っています。コンテナは、あとでこのイメージから起動します。このファイルだけでは、プログラムは動きません。

## 例

```dockerfile
ARG PYTHON_VERSION=3.12  # イメージを作るときに使う置き換え用の変数です。環境変数ではありません。${PYTHON_VERSION} は 3.12 に置き換わります

FROM python:${PYTHON_VERSION}-slim  # 用意されているイメージ python:3.12-slim から始めます。python が名前、3.12-slim がタグです

ENV PORT=8000  # イメージに書き込む環境変数です。このイメージから起動したコンテナで PORT を読めます。今のターミナルには入りません

WORKDIR /app  # イメージの中に /app を作り、これ以降の命令の現在位置をここにします

RUN useradd --create-home app  # イメージを作るときに、イメージの中で1回実行し、ユーザー app を作ります。結果はイメージに残ります
                               # RUN：イメージを作るときに、後ろのコマンドをイメージの中で実行します。結果はイメージに残ります。コンテナの起動時には、この行は動きません
                               # useradd：Linux でユーザーを作るコマンドです
                               # --create-home：useradd のオプションです。このユーザーのホームディレクトリも作ります。ふつうは /home/app です
                               # app：作るユーザーの名前です

COPY requirements.txt .  # 用意したフォルダの requirements.txt を、イメージの /app に入れます。ピリオドはイメージの中の場所です
RUN python -m pip install --no-cache-dir -r requirements.txt  # イメージを作るときに、イメージ内の Python で、一覧どおりにライブラリを入れます。ダウンロードのキャッシュはイメージに残しません
                                                              # RUN：イメージを作るときに、後ろのコマンドをイメージの中で実行します。結果はイメージに残ります。コンテナの起動時には、この行は動きません
                                                              # python：イメージの中にある Python のプログラムです
                                                              # -m pip：この Python で pip を実行します。pip は、Python のライブラリを入れる道具です
                                                              # install：pip でライブラリを入れる動作です
                                                              # --no-cache-dir：pip のオプションです。ダウンロードのキャッシュをイメージに残しません
                                                              # -r：一覧ファイルの内容どおりに入れます
                                                              # requirements.txt：ライブラリの一覧ファイルです

COPY app.py .  # 用意したフォルダの app.py を、イメージの /app に入れます
               # .：COPY の置き先です。WORKDIR で決めた /app と同じです。app.py は /app/app.py に置かれます

USER app  # イメージは、最初は管理者の root で実行します。上の useradd で、一般ユーザー app を作りました。この行で、これ以降の命令と、コンテナ起動後のプログラムを、ユーザー app で実行します

EXPOSE 8000  # イメージに、コンテナの中で待ち受けるポートが 8000 であることを記録します。このポートはコンテナの中にあります。コンテナを起動するとき、ポートの対応づけで、パソコン側のポートをコンテナの 8000 につなぎます

ENTRYPOINT ["python"]  # コンテナを起動するときに実行するプログラムです。イメージに記録しておき、今は実行しません
CMD ["app.py"]  # 前の行に渡す引数です。起動時に実際に動くのは python app.py です
```

## コメントに書いていないこと

`FROM` の前に書いた `ARG` は、`FROM` の中だけで有効です。

`3.12-slim` の `slim` は、容量の小さい版です。`FROM` が終わると、イメージの中は Python 3.12 が入った Linux です。

`RUN useradd --create-home app` には角括弧がありません。イメージの中のシェルに渡して実行します。

先に一覧を `COPY` してインストールし、そのあとで `app.py` を `COPY` します。あとから `app.py` だけを変えたときは、入っているライブラリをそのまま使えます。

`COPY . .` では、前の `.` は渡されたフォルダ全体、後ろの `.` はイメージの中の `/app` です。

`USER` は `useradd` のあとに書きます。ユーザー `app` を先に作っておくためです。

`ENTRYPOINT` と `CMD` の角括弧では、1項目が1つの単語です。コンテナの起動時に別の引数を渡すと、入れ替わるのは `CMD` です。`ENTRYPOINT` は `python` のままです。`other.py` を渡すと、実行されるのは `python other.py` です。`CMD` がいくつもあるときは、最後の1つを使います。`CMD python app.py` のように角括弧なしで書くと、先にシェルを通します。

## この例にはない命令

**`ADD`**

`COPY` と同じように、ファイルをイメージへ入れます。圧縮ファイルを展開することや、URL からファイルを取ってくることもできます。ふつうのコピーには `COPY` を使います。

**`LABEL`**

イメージに説明を書きます。プログラムの実行には使いません。例えば `LABEL org.opencontainers.image.title="demo"` です。

**`VOLUME`**

イメージの中で、残しておきたいデータを置くディレクトリを示します。例えば `VOLUME /data` です。この行は、ディレクトリを示すだけです。

**`HEALTHCHECK`**

コンテナの起動後、決めた間隔で、コンテナの中の確認コマンドを実行します。例えば 30 秒ごと、1回の待ち時間は最大 5 秒、3回続けて失敗すると異常です。

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/')"
```
