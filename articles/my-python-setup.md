---
title: "おすすめのPythonの環境構築まとめ"
emoji: "🐍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Python", "VSCode", "poetry"]
published: false
---

# おすすめのPythonの環境構築まとめ

個人的に普段はTypeScriptやKotlinを書く機会が多いのですが、バッチ処理などではPythonを採用することが多いです。
久しぶりにPythonを書くと環境構築はどうするのが良いんだっけと、いつも思ってしまうのでここにまとめます。

OSはmacOS、エディタについてはVSCodeを使用することとします。

## 使用するツールなど

- Pythonバージョン切り替え
  - pyenv

https://github.com/pyenv/pyenv

- パッケージ管理ツール
  - poetry

Pythonではpipやpipenvなど様々なパッケージ管理ツールがありましたが、現在はpoetry以外を使う選択肢は基本的にないと思います。
pipやpipenvとの比較はここでは省略します。

https://python-poetry.org/

- Linter
  - flake8

Linterはflake8かpylintが有名ですが、最近はflake8が多く使われているようなのでこちらを採用します。

https://github.com/PyCQA/flake8

- Formatter
  - black

PEP8に準拠しているフォーマッタ。自由に設定できる項目が少ないのが特徴です。
設定できる項目が少ないことは、裏を返せば細かい設定が不要ということでもあります。
今のところフォーマットに特に不満はないので、こちらを採用します。

https://black.readthedocs.io/

## 導入手順

### pyenv のインストール

私はpyenvの他にもrbenvなどを使う機会も多いので、anyenvでpyenvを導入します。
anyenvは\*\*envをまとめて管理できるツールです。
導入方法は下記リポジトリのREADMEの通りなので省略します。

https://github.com/anyenv/anyenv

### poetry のインストール

homebrewからpoetryをインストールします。

```bash
brew install poetry
```

### プロジェクトの作成

poetryのコマンドから新しいプロジェクトを作成します。

```bash
poetry new python-my-project
```

以下のような構成でファイルが作成されます。
READMEはrst形式で生成されるため、好みに応じてMarkdownに直します。

```
python-my-project
├── pyproject.toml
├── README.rst
├── python_my_project
│   └── __init__.py
└── tests
    ├── __init__.py
    └── test_python_my_project.py
```

### Python バージョンの指定

pyenvを使用してPythonのバージョンを指定します。今回は3.9.7を使用します。
プロジェクトのルートディレクトリに移動し、pyenvで使用するバージョンを指定します。

```bash
cd python-my-project
pyenv local 3.9.7
```

このプロジェクトではpyenvでインストールした3.9.7が使用されるようになりました。

また、 `pyproject.toml` には `poetry new` した際のPythonバージョンが指定されています。
使用しているPythonのバージョンと異なるとエラーが発生するため、必要に合わせてバージョンを書き換えてください。

```toml:pyproject.toml
[tool.poetry.dependencies]
python = "^3.9.7"
```

### 仮想環境の作成

`poetry install` を実行すれば仮想環境が作成されます。
しかし、このままだと以下のようなディレクトリに作成されます。

```
/Users/yuki-furukawa/Library/Caches/pypoetry/virtualenvs/python-my-project-64lWWP9p-py3.9
```

仮想環境はルートディレクトリ以下に作成されるようにすると、VSCodeから仮想環境を利用でき便利なので設定を変更します。

グローバルに設定したいときはこのコマンドで設定します。

```bash
poetry config virtualenvs.in-project true
```

プロジェクト単位で設定したい場合は `--local` を付けます。

```bash
poetry config --local virtualenvs.in-project true
```

今回はプロジェクト単位で設定することとします。
プロジェクト単位で設定した場合は、以下のような設定ファイルが作成されます。

```toml:poetry.toml
[virtualenvs]
in-project = true
```

この状態で `poetry install` をするとルートディレクトリ以下に `.venv` というディレクトリで仮想環境が作成されます。

### flake8 と black のインストール

poetryを使ってflake8とblackをインストールします。

```bash
poetry add -D flake8 black
```

dev-dependenciesとして追加するため、 `-D` オプションを使います。

### VSCode の設定

予めPythonの拡張機能をインストールしておきます。
https://marketplace.visualstudio.com/items?itemName=ms-python.python

#### 仮想環境の設定

ステータスバーを確認すると、poetryから作成した仮想環境が自動的に読み込まれています。
![](https://storage.googleapis.com/zenn-user-upload/b177cba2a10b97030bcc7e21.png)

もし他のインタープリターが選択されている場合は、ステータスバーをクリックすると選択し直すことができます。
![](https://storage.googleapis.com/zenn-user-upload/fdf43184edc9b07c5f203e18.png)

VSCode上で仮想環境を正しく認識できていると、ターミナルを開くと認識している仮想環境を自動的に読み込んでくれます。
この状態でPythonコマンドを使用すれば、仮想環境で実行することができます。
![](https://storage.googleapis.com/zenn-user-upload/b7292e67f224c0b8e88269d3.png)

#### flake8の設定

`Command + Shift + P` でコマンドパレットを開き、 `Python: Select Linter` を選択し、 `flake8` を選択します。
![](https://storage.googleapis.com/zenn-user-upload/8c102a80b27626836731d172.png)

#### blackの設定

設定から `python.formatting.provider` の値をblackに変更します。
![](https://storage.googleapis.com/zenn-user-upload/be8fae19be666b821419f6e8.png)

コマンドパレットから手動でフォーマットも出来ますが、 `editor.formatOnSave` を `true` に変更し、保存時に自動でフォーマットするのもおすすめです。

## 実際に使ってみる

### flake8とblackの動作確認

このようなファイルを作成し、保存してみます。

```python:main.py
import sys

hoge = [
    'first','second'
,'third'  ]

print(hoge)
```

![](https://storage.googleapis.com/zenn-user-upload/9db6d1c38781c51f78c772e4.png)

保存時に自動的にblackでフォーマットし、flake8によるチェックも行われるようになりました。

### Pythonの実行

右上の実行ボタンを押すだけです。
![](https://storage.googleapis.com/zenn-user-upload/8d7199604501a667586e807f.png)

ターミナルから実行する場合は仮想環境が読み込まれているため、以下のコマンドで実行可能です。

```bash
$ python python_my_project/main.py
# ['first', 'second', 'third']
```

仮想環境が読み込まれていない場合は `poetry run` 経由で実行します。

```bash
$ poetry run python python_my_project/main.py
# ['first', 'second', 'third']
```

### おわりに

毎回環境構築で時間が取られるのがもったいないと感じ、自分が見返すようにまとめました。
他に何かおすすめの設定などがあれば教えてください。
