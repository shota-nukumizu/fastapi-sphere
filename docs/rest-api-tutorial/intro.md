# FastAPIでTodoアプリケーションのREST APIを作る

## はじめに

本チュートリアルでは、FastAPIとSQLite、SQLModelを使って簡単なTodoアプリケーションのREST APIを開発する手順を解説する。

## 技術説明

### [SQLite](https://www.sqlite.org/index.html)

SQLiteは、軽量なデータベースソフトウェアだ。最大の特徴はサーバの設定が不要である点である。大抵のプラットフォームでは、別途インストールすることなく使える。データは一つのファイルとして保存され、データのバックアップや移行が簡単だ。

### [SQLModel](https://sqlmodel.tiangolo.com/)

SQLModelはPythonのORMライブラリである。SQLModelを使うと、Pythonのクラスを使ってデータベースのテーブルを表現できる。

### [Swagger](https://swagger.io/)

Swaggerは、OpenAPI Specificationを使ってREST APIを設計するためのフレームワークの一部だ。FastAPIを使うと、Swagger UIのドキュメントとテストツールが自動で提供される。これを使うことで、エンドポイントの操作をブラウザから直接テストでき、APIの仕様も簡単に確認できる。

## 手順

### (1) 必要なライブラリのインストール

`myproject`フォルダを新規で作成する。その後、そのフォルダに移動して必要なライブラリを以下のコマンドでインストールする。

```bash
pip install fastapi[all] sqlmodel sqlite
```

### (2) DB設定

`myproject`フォルダの下に`app`フォルダを書き、`database.py`を新規で作成してデータベースを作成する。

`/myproject/app/database.py`

```py
# (1) 必要なライブラリをsqlmodelからインポートする
from sqlmodel import SQLModel, create_engine

# (2) 変数DATABASE_URLにデータベースの位置を設定する
DATABASE_URL = "sqlite:///../main.db"

# (3) create_engine関数を使ってデータベースへの接続エンジンを作成する
engine = create_engine(DATABASE_URL)
```

### (3) モデル作成

`/myproject/app/models.py`に以下のコードを書いて、Todoアイテムのモデルを作成する。

```py
# (1) 必要なライブラリをインポートする
from typing import Optional
from sqlmodel import SQLModel, Field

# (2) Todoクラスを作成する
class Todo(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    title: str
    description: str
    completed: bool
```

上述の(2)のコードに焦点を当てて解説する。

```py
class Todo(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    title: str # str: 文字列型
    description: str
    completed: bool # bool: 真偽値(true or false)
```

上述のモデルでは、ユーザはタスクのタイトル(`title`)と説明(`description`)、完了状態(`completed`)をSQLiteデータベースに保存できる。一つ一つのカラムを説明すると冗長になるので、今回の記事では以下のコードに焦点を当てて解説をする。

```py
id: Optional[int] = Field(default=None, primary_key=True)
```

上述の定義で、`id`はいくつかの特性を持っている。

* `Optional[int]`：この型ヒントは、`id`の値が`int`あるいは`None`かのいずれかを意味する。
* `Field(default=None, primary_key=True)`：`id`がプライマリキーであること、そしてデフォルト値が`None`だということを意味する。

SQLiteでデータベースを実装する場合、プライマリキーとして指定されたフィールドはそのテーブル内のそれぞれのレコードを一位に識別するためのものである。また、SQLiteではプライマリキーの整数フィールドに対してデフォルト値で`None`(あるいは`NULL`)が指定されると、SQLiteは**自動的にそのフィールドに一意の値を割り当てる**。

### (4) FastAPIアプリケーションの作成

`/myproject/app/main.py`に以下のコードを書いて、FastAPIアプリケーションを作成する。

```py
# (1) 必要なモジュール、関数やクラスをインポートする
from fastapi import FastAPI, HTTPException
from sqlmodel import Session, select
from .database import engine
from .models import Todo

# (2) FastAPIインスタンスを作成する
app = FastAPI()

# (3) app.on_event関数でデコレータし、APIが起動するときに実行される関数を定義する
@app.on_event("startup")
def on_startup():
    with Session(engine) as session:
        session.exec(select(Todo).where(Todo.id == 1))
        session.commit()

# (4)-1 エンドポイントを定義する 
@app.post("/todos/", response_model=Todo)
def create_todo(todo: Todo):
    with Session(engine) as session:
        session.add(todo)
        session.commit()
        session.refresh(todo)
        return todo

# (4)-2 エンドポイントを定義する
@app.get("/todos/", response_model=list[Todo])
def read_todos():
    with Session(engine) as session:
        return session.exec(select(Todo)).all()

# (4)-3 エンドポイントを定義する
@app.get("/todos/{todo_id}", response_model=Todo)
def read_todo(todo_id: int):
    with Session(engine) as session:
        todo = session.get(Todo, todo_id)
        if not todo:
            raise HTTPException(status_code=404, detail="Todo not found")
        return todo

# (4)-4 エンドポイントを定義する
@app.put("/todos/{todo_id}", response_model=Todo)
def update_todo(todo_id: int, todo: Todo):
    with Session(engine) as session:
        db_todo = session.get(Todo, todo_id)
        if not db_todo:
            raise HTTPException(status_code=404, detail="Todo not found")
        todo_data = todo.dict(exclude_unset=True)
        for key, value in todo_data.items():
            setattr(db_todo, key, value)
        session.add(db_todo)
        session.commit()
        session.refresh(db_todo)
        return db_todo

# (4)-5 エンドポイントを定義する
@app.delete("/todos/{todo_id}", response_model=Todo)
def delete_todo(todo_id: int):
    with Session(engine) as session:
        db_todo = session.get(Todo, todo_id)
        if not db_todo:
            raise HTTPException(status_code=404, detail="Todo not found")
        session.delete(db_todo)
        session.commit()
        return db_todo
```

上述のコードは、FastAPIを用いてRESTful APIを作成するためのプログラムである。上述の`main.py`は非常に長いので、各エンドポイント(`GET`、`POST`、`PUT`、`DELETE`)を軸に解説を進める。

### (4)で実装したエンドポイントの説明

#### `POST`メソッド

```python
@app.post("/todos/", response_model=Todo)
def create_todo(todo: Todo):
    with Session(engine) as session:
        session.add(todo)
        session.commit()
        session.refresh(todo)
        return todo
```
- `@app.post("/todos/")`: POSTリクエストを受け付けるエンドポイントを定義しています。
- `response_model=Todo`: レスポンスとして返すモデルの形式を指定しています。
- `todo: Todo`: リクエストボディから`Todo`モデルの形式でデータを受け取ります。
- `session.add(todo)`: 新しい`Todo`オブジェクトをデータベースセッションに追加します。
- `session.commit()`: データベースに変更をコミットします。
- `session.refresh(todo)`: データベースから最新の情報を取得して、`todo`オブジェクトを更新します。

#### `GET`メソッド

```python
@app.get("/todos/", response_model=list[Todo])
def read_todos():
    with Session(engine) as session:
        return session.exec(select(Todo)).all()
```
- `@app.get("/todos/")`: GETリクエストを受け付けるエンドポイントを定義しています。
- `response_model=list[Todo]`: レスポンスとして返すモデルの形式を指定しています。ここでは`Todo`のリストとしています。
- `session.exec(select(Todo)).all()`: すべての`Todo`オブジェクトをデータベースから選択して取得します。

#### `GET`メソッド(特定のTodoを取得する)

```python
@app.get("/todos/{todo_id}", response_model=Todo)
def read_todo(todo_id: int):
    with Session(engine) as session:
        todo = session.get(Todo, todo_id)
        if not todo:
            raise HTTPException(status_code=404, detail="Todo not found")
        return todo
```
- `@app.get("/todos/{todo_id}")`: 特定のIDを持つ`Todo`を取得するためのエンドポイントを定義しています。
- `todo_id: int`: URLから`todo_id`として整数を受け取ります。
- `session.get(Todo, todo_id)`: 指定されたIDを持つ`Todo`オブジェクトをデータベースから取得します。
- `if not todo`: `Todo`が見つからない場合、404エラーを返します。

#### `PUT`メソッド

```python
@app.put("/todos/{todo_id}", response_model=Todo)
def update_todo(todo_id: int, todo: Todo):
    with Session(engine) as session:
        db_todo = session.get(Todo, todo_id)
        if not db_todo:
            raise HTTPException(status_code=404, detail="Todo not found")
        todo_data = todo.dict(exclude_unset=True)
        for key, value in todo_data.items():
            setattr(db_todo, key, value)
        session.add(db_todo)
        session.commit()
        session.refresh(db_todo)
        return db_todo
```
- `@app.put("/todos/{todo_id}")`: 特定のIDを持つ`Todo`を更新するためのエンドポイントを定義しています。
- `todo: Todo`: リクエストボディから更新データを受け取ります。
- `db_todo = session.get(Todo, todo_id)`: 指定されたIDを持つ`Todo`オブジェクトをデータベースから取得します。
- `todo_data = todo.dict(exclude_unset=True)`: 未設定のフィールドを除外して、`todo`のデータを辞書として取得します。
- `for key, value in todo_data.items()`: 辞書の各キーと値をループして、`db_todo`オブジェクトの属性を更新します。

#### `DELETE`メソッド

```python
@app.delete("/todos/{todo_id}", response_model=Todo)
def delete_todo(todo_id: int):
    with Session(engine) as session:
        db_todo = session.get(Todo, todo_id)
        if not db_todo:
            raise HTTPException(status_code=404, detail="Todo not found")
        session.delete(db_todo)
        session.commit()
        return db_todo
```
- `@app.delete("/todos/{todo_id}")`: 特定のIDを持つ`Todo`を削除するためのエンドポイントを定義しています。
- `db_todo = session.get(Todo, todo_id)`: 指定されたIDを持つ`Todo`オブジェクトをデータベースから取得します。
- `session.delete(db_todo)`: `db_todo`オブジェクトをデータベースから削除します。

これらのエンドポイントは、基本的なCRUD操作を提供しており、ユーザーはAPIを通じて`Todo`モデルのデータを操作できます。

### (5) アプリケーションの実行

ルートディレクトリ、いわゆる`/myproject`フォルダに移動してアプリケーションを起動する。

```bash
uvicorn app.main:app --reload
```

`http://127.0.0.1:8000`にFastAPIアプリケーションにアクセスできる。あと、`http://127.0.0.1:8000/docs`でSwaggerを経由してAPIドキュメントも作成できる。

FastAPIには**デフォルトでSwagger UIが搭載されており、ブラウザから直接APIをテストできる**。Swagger UIを使うと、それぞれのAPIエンドポイントの詳細を確認し、リクエストを送信してレスポンスを確認できる。コマンドラインツールを使わなくても実際にテストできるのだ。詳細は[公式ドキュメント](https://fastapi.tiangolo.com/features/)を参照してほしい。

## 最終的なディレクトリ

```
/myproject
    /app
        __init__.py
        main.py
        models.py
        database.py
    main.db
```