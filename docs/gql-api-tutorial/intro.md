# FastAPIで簡単なGraphQL APIをつくる

今回のトピックでは、主に以下の技術を用いてユーザ名とメールアドレスを表示するGraphQL APIを実装する方法を解説する。

* SQLite
* Strawberry

## Strawberryとは

Strawberryは、PythonでGraphQL APIを構築するためのライブラリだ。FastAPIと組み合わせて使用することで、非常に効率的かつ強力なAPIを開発できる。

### 特徴

* **型安全**：Pythonの型ヒントを使って、型安全なWeb APIを構築できる。
* **デコレータ**：ResolverやMutationを簡単に定義できるデコレータが提供される。
* **データクラス**：Pythonのデータクラスを使って、GraphQLスキーマを定義できる。

### 基本的な使い方

```py
import strawberry

@strawberry.type
class User:
    id: int
    name: str

@strawberry.type
class Query:
    @strawberry.field
    def user(self, info, id: int) -> User:
        return User(id=id, name="Example")

schema = strawberry.Schema(query=Query)
```

### FastAPIとの組み合わせ

FastAPIの`add_graphql_route`メソッドを使って、Strawberryで定義したスキーマをFastAPIアプリケーションに追加できる。

```py
from fastapi import FastAPI
from strawberry.fastapi import GraphQLRouter

app = FastAPI()
app.add_router("/graphql", GraphQLRouter(schema=schema))
```

詳細は[公式ドキュメント](https://strawberry.rocks/)を参照してほしい。

## 手順

### (1) セットアップ

新しくディレクトリを作成する。

```bash
mkdir fastapi-graphql
cd fastapi-graphql
```

### (2) 必要なパッケージのインストール

以下のコマンドを入力して、必要なパッケージをインストールする。

```bash
pip install fastapi[all] uvicorn sqlite strawberry-graphql
```

### (3) `main.py`ファイルの作成

```py
# (1) 必要なモジュールやクラスをインポートする
from typing import List
from fastapi import FastAPI
from strawberry.fastapi import GraphQLRouter
import strawberry
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session

# (2) データベースを設定する
DATABASE_URL = "sqlite:///./test.db"

# (3) モデルを定義する
Base = declarative_base()

# (4) Userモデルを定義する
class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, index=True)
    email = Column(String, unique=True, index=True)

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# (5) FastAPIのインスタンスを作成する
app = FastAPI()

# (6) User型を定義する
@strawberry.type
class UserType:
    id: int
    username: str
    email: str

@strawberry.type
class Query:
    @strawberry.field
    def users(self, info) -> List[UserType]:
        db: Session = info.context["db"]
        return db.query(User).all()

# (7) GraphQLルーターを設定する
router = GraphQLRouter(schema=strawberry.Schema(query=Query))
app.include_router(router, route_path="/graphql")

# (8) アプリケーションを起動する関数を書く
@app.on_event("startup")
async def startup_event():
    Base.metadata.create_all(bind=engine)

```

### (4) データベースをセットアップする

(3)で記述したコードを実行する前に、SQLiteデータベースをセットアップしなければならない。以下のコマンドを実行しよう。

```bash
sqlite3 test.db
```

そして、以下のSQLコマンドを実行して`users`テーブルを作成する。

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    username TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE
);
```

### (5) サーバの起動

最後に、以下のコマンドでFastAPIサーバを起動する。

```bash
uvicorn main:app --reload
```

ブラウザで`http://127.0.0.1:8000/graphql`二アクセスすると、GraphQL Playgroundが表示されてGraphQLクエリを実行できるようになる。

## 実際にデータを送信する手順

### (1) Mutationの追加

最初に、`main.py`にMutationを追加して新規ユーザをデータベースに追加できるようにする。

```py
# ... (他のインポート)
from sqlalchemy.exc import IntegrityError

# ... (他のコード)

# (1) Mutationの型定義を行う
@strawberry.type
class Mutation:
    @strawberry.mutation
    def create_user(self, info, username: str, email: str) -> UserType:
        db: Session = info.context["db"]
        user = User(username=username, email=email)
        db.add(user)
        try:
            db.commit()
        except IntegrityError:
            db.rollback()
            raise ValueError("User with this email already exists")
        return user

# (2) GraphQLルーターの設定を行う
router = GraphQLRouter(schema=strawberry.Schema(query=Query, mutation=Mutation))
app.include_router(router, route_path="/graphql")

# ... (他のコード)
```

上述のコードは`create_user`というMutationを追加し、これを使って新しいユーザをデータベースに追加できる。特に(1)のプログラムの部分に焦点を当てて解説する。

```py
@strawberry.type
class Mutation:
    # メソッドがGraphQLのMutationであることを示すデコレータ
    @strawberry.mutation
    # 余談：Pythonでクラスを定義するとき、必ず第一引数にselfを置く
    def create_user(self, info, username: str, email: str) -> UserType:
        db: Session = info.context["db"]
        # 新しいユーザのオブジェクトを作成する
        user = User(username=username, email=email)
        db.add(user)
        # この部分を詳細に説明する
        try:
            db.commit()
        except IntegrityError:
            db.rollback()
            raise ValueError("User with this email already exists")
        return user
```

特に、上述のコードにおける以下の部分に焦点を当てて解説する。

```py
try:
    db.commit()
except IntegrityError:
    db.rollback()
    raise ValueError("User with this email already exists")
```

上述は`try...except`構文を用いて、データベースに接続する際にエラーが発生した際の対処を書いている。より具体的に解説すると、同じメールアドレスを持つユーザが既に存在している場合(`IntegrityError`)にロールバックを行い、エラーメッセージを返す。

このコードで、GraphQLのエンドポイントを経由して新しいユーザをデータベースに追加できるのだ。

### (2) サーバの再起動

変更を適用するために、サーバを再起動する。

```bash
uvicorn main:app --reload
```

### (3) Mutationのテスト

GraphQL Playgroundにて以下のMutationを実行。新しいユーザをデータベースに追加する。

```
mutation {
  createUser(username: "john_doe", email: "john_doe@example.com") {
    id
    username
    email
  }
}
```

### (4) Queryのテスト

最後に、以下のQueryを実行して、データベースに追加されたユーザを取得する。

```
{
  users {
    id
    username
    email
  }
}
```

これでSQLiteにデータを送信し、QueryやMutationを実行できるようになる。