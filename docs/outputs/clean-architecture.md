# クリーンアーキテクチャ（Clean Architecture）

## 目次

1. [クリーンアーキテクチャとは何か](#1-クリーンアーキテクチャとは何か)
2. [主要構成要素](#2-主要構成要素)
3. [他のアーキテクチャとの比較](#3-他のアーキテクチャとの比較)
4. [FastAPIでの実装例](#4-fastapiでの実装例)
5. [メリット・デメリット](#5-メリットデメリット)
6. [Anti-patterns（アンチパターン）](#6-anti-patternsアンチパターン)
7. [バージョン情報](#7-バージョン情報)
8. [参考情報](#参考情報)

---

## 1. クリーンアーキテクチャとは何か

### 概念・定義

クリーンアーキテクチャ（Clean Architecture）は、Robert C. Martin（Uncle Bob）が 2012 年に提唱したソフトウェア設計パターンである。
ヘキサゴナルアーキテクチャ（2005年）やオニオンアーキテクチャ（2008年）など、先行する複数のアーキテクチャパターンを統合・整理した上で、同心円状の4層構造として定式化したものである。

> "The overriding rule that makes this architecture work is The Dependency Rule. This rule says that source code dependencies can only point inwards. Nothing in an inner circle can know anything at all about something in an outer circle."
>
> — Robert C. Martin, [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

クリーンアーキテクチャが達成しようとする特性は以下の通りである。

- **Framework Independent（フレームワーク独立性）**: アーキテクチャはフレームワークの存在に依存しない。フレームワークはツールとして扱われる。
- **Testable（テスト可能性）**: UI・データベース・Web サーバーなしにビジネスルールをテストできる。
- **UI Independent（UI 独立性）**: UI はビジネスルールを変更せずに変更できる。
- **Database Independent（データベース独立性）**: Oracle や SQL Server を MongoDB・BigTable・CouchDB などに差し替えられる。ビジネスルールはデータベースに縛られない。
- **External Agency Independent（外部機関独立性）**: ビジネスルールは外部インターフェースについて何も知らない。

### 誕生した背景・解決しようとした問題

従来の Layered Architecture（レイヤードアーキテクチャ）は以下の問題を抱えていた。

- **フレームワークへの密結合**: Spring や Django といったフレームワークのライフサイクルにビジネスロジックが縛られ、移行コストが高い。
- **テストの困難性**: データベースや UI に依存するため、ビジネスロジックの Unit Test が難しい。
- **外部変化への脆弱性**: データベースエンジンや外部 API の変更が全層に波及する。

クリーンアーキテクチャは「ビジネスルールを中心に据え、UI・データベース・フレームワークを詳細（Detail）として扱い、外側に配置する」という構造によってこれらの問題を解決する。

### SOLID 原則との関係

クリーンアーキテクチャは SOLID 原則、特に依存性逆転の原則（Dependency Inversion Principle, DIP）を根幹としている。

> "The Dependency Inversion Principle tells us that the most flexible systems are those in which source code dependencies refer only to abstractions, not to concretions."
>
> — Robert C. Martin

Use Cases 層が Repository の具体的な実装（SQLAlchemy など）に直接依存するのではなく、抽象（インターフェース）に依存することで、具体的な実装を自由に差し替えられる構造を実現している。

---

## 2. 主要構成要素

```mermaid
graph TB
    subgraph FD["④ Frameworks & Drivers（最外層）"]
        DB["Database\n(PostgreSQL/SQLAlchemy)"]
        WF["Web Framework\n(FastAPI)"]
        EXT["External APIs"]
    end

    subgraph IA["③ Interface Adapters"]
        CTRL["Controllers\n(FastAPI Router)"]
        GW["Gateways / Repositories\n(SQLAlchemy実装)"]
        PRES["Presenters\n(レスポンス変換)"]
    end

    subgraph UC["② Use Cases"]
        UCLS["Use Case Classes\n(アプリケーションビジネスルール)"]
        REPO_IF["Repository Interfaces\n(依存性逆転用の抽象)"]
    end

    subgraph ENT["① Entities（最内層）"]
        DOM["Domain Entities\n(エンタープライズビジネスルール)"]
    end

    WF -->|依存| CTRL
    DB -->|依存| GW
    CTRL -->|依存| UCLS
    GW -.->|実装| REPO_IF
    UCLS -->|依存| DOM
    UCLS -->|使用| REPO_IF
```

### Entities層（エンティティ層）

- **責務**: エンタープライズ全体のビジネスルールをカプセル化する。最も変化が少ない最重要のビジネスルールを含む。
- **含まれるもの**: Entity（エンティティ）、Value Object（値オブジェクト）、Domain Event
- **依存関係ルール**: 外部フレームワーク・ライブラリへの依存を持たない。Pure Python で記述される。
- **特徴**: 複数のアプリケーション間で再利用可能。外部変化（UI 変更・データベース変更）の影響を受けない。

> "Entities encapsulate Enterprise wide business rules. An entity can be an object with methods, or it can be a set of data structures and functions. It doesn't matter so long as the entities could be used by many different applications in the enterprise."
>
> — Robert C. Martin, [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

### Use Cases層（ユースケース層）

- **責務**: アプリケーション固有のビジネスルールを実装する。Entity に対する操作を調整し、ユースケースの目標を達成する。
- **含まれるもの**: Use Case クラス、Application Service、Repository インターフェース（DIP のための抽象定義）
- **依存関係ルール**: Entities 層のみに依存。データベースや UI については何も知らない。

> "The software in this layer contains application specific business rules. It encapsulates and implements all of the use cases of the system."
>
> — Robert C. Martin, [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

**重要**: Use Cases 層は Repository の具体的実装を知らない。代わりに Repository インターフェース（抽象）をこの層で定義し、具体的な実装は外側の Interface Adapters 層が行う。これが Dependency Inversion Principle の適用箇所である。

### Interface Adapters層（インターフェースアダプター層）

- **責務**: Use Cases / Entities にとって便利な形式と、外部システム（DB・UI）にとって便利な形式の間でデータを変換する。
- **含まれるもの**: MVC の Controllers・Presenters・Gateways、Repository の具体的実装（SQLAlchemy など）
- **依存関係ルール**: Use Cases / Entities に依存する。

> "This layer is the set of adapters that convert data from the format most convenient for the use cases and entities, to the format most convenient for some external agency such as the Database or the Web."
>
> — Robert C. Martin, [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

### Frameworks & Drivers層（フレームワーク・ドライバー層）

- **責務**: データベース・Web フレームワーク・UI などの詳細を管理する最外層。ほとんどコードを書かず、内側の層と通信するための「glue code（のりしろコード）」のみ記述する。
- **含まれるもの**: FastAPI アプリケーション設定、SQLAlchemy エンジン設定、外部ライブラリの初期化、DI 設定
- **依存関係ルール**: 全層の中で最も外側に位置し、Interface Adapters 層に依存する。

### Dependency Rule（依存方向のルール）

クリーンアーキテクチャの最重要原則は **The Dependency Rule** である。

- Frameworks & Drivers → Interface Adapters → Use Cases → Entities の方向にのみ依存できる。
- 内側の円は外側の円について何も知ってはいけない。
- 外側の円に宣言された名前（関数・クラス・変数）を内側の円のソースコードで参照してはいけない。

---

## 3. 他のアーキテクチャとの比較

### Layered Architecture（レイヤードアーキテクチャ）との違い

参考: [Layered vs Clean vs Onion vs Hexagonal Architecture — A Practical Guide - Medium](https://medium.com/@rup.singh88/stop-confusing-clean-onion-hexagonal-architecture-heres-when-to-use-each-692079e56267)

| 観点 | Layered Architecture | Clean Architecture |
|------|---------------------|---------------------|
| 依存の方向 | 上位層から下位層へ一方向 | 全て Entities（最内円）に向かって内向き |
| DB への依存 | Service 層が直接 Repository を参照 | Use Cases は Repository インターフェースのみ参照 |
| テスト容易性 | 下位層への依存でテストが困難 | Repository インターフェースを Mock に差し替え容易 |
| フレームワーク依存 | ビジネスロジックがフレームワークと密結合しやすい | フレームワークは最外層（詳細）として扱われる |
| 適した場面 | CRUD 中心のシンプルなアプリ | 複雑なドメインロジックを持つ大規模アプリ |

### Hexagonal Architecture（ヘキサゴナルアーキテクチャ）との違い

参考: [Clean Architecture vs. Onion Architecture vs. Hexagonal Architecture - CCD Akademie](https://ccd-akademie.de/en/clean-architecture-vs-onion-architecture-vs-hexagonal-architecture/)

| 観点 | Hexagonal Architecture | Clean Architecture |
|------|----------------------|---------------------|
| 提唱者・年 | Alistair Cockburn（2005年） | Robert C. Martin（2012年） |
| 中心概念 | Port と Adapter | 4層の同心円（Entities / Use Cases / Interface Adapters / F&D） |
| 層の構造 | Core（Domain + Application）と Adapters の2分類 | 明示的な4層の同心円構造 |
| 内部層の詳細度 | Port / Adapter の観点から定義 | Entities / Use Cases を明確に分離して定義 |
| 依存方向 | Core への内向き依存 | 依存性逆転の原則（DIP）で内向き |
| 命名体系 | Port / Adapter / Driving / Driven | Entities / Use Cases / Interface Adapters / Frameworks & Drivers |

両者の根底にある原則（ビジネスロジックを外部から独立させる・依存は内向き）は共通しており、実装上の違いは主に「命名体系」と「内部層の詳細度」にある。

### Onion Architecture（オニオンアーキテクチャ）との違い

参考: [Onion vs Clean vs Hexagonal Architecture - Medium](https://medium.com/@edamtoft/onion-vs-clean-vs-hexagonal-architecture-9ad94a27da91)

| 観点 | Onion Architecture | Clean Architecture |
|------|---------------------|---------------------|
| 提唱者・年 | Jeffrey Palermo（2008年） | Robert C. Martin（2012年） |
| 中心概念 | Domain Model を中心とした同心円 | Entities を中心とした明示的な4層の同心円 |
| Repository の位置 | Domain Model 層（最内層）内にインターフェースを配置 | Use Cases 層内にインターフェースを配置 |
| Entities との違い | "Domain Model"（エンタープライズルールとアプリルールが混在） | Entities（エンタープライズ）と Use Cases（アプリ）を明示的に分離 |
| 発展経緯 | Hexagonal Architecture を拡張した構造 | Hexagonal + Onion を整理・統合した構造 |

3つのアーキテクチャは根底の原則（DIP に基づく内向き依存）を共有しており、本質的には同じ思想の異なる表現である。

> "All of them are based on DIP (Dependency Inversion Principle)"
>
> — [Clean Architecture vs. Onion Architecture vs. Hexagonal Architecture - CCD Akademie](https://ccd-akademie.de/en/clean-architecture-vs-onion-architecture-vs-hexagonal-architecture/)

---

## 4. FastAPIでの実装例

### ディレクトリ構成例

```
my_app/
├── entities/                              # ① Entities層
│   └── user.py                            # Entity（エンタープライズビジネスルール）
│
├── use_cases/                             # ② Use Cases層
│   ├── interfaces/
│   │   └── user_repository.py             # Repository インターフェース（DIP用）
│   └── user_use_cases.py                  # Use Case（アプリケーションビジネスルール）
│
├── interface_adapters/                    # ③ Interface Adapters層
│   ├── controllers/
│   │   └── user_router.py                 # FastAPI Router（Controller）
│   └── gateways/
│       ├── orm_models.py                  # SQLAlchemy ORM Model
│       └── sqlalchemy_user_repository.py  # Repository実装（Gateway）
│
├── frameworks_drivers/                    # ④ Frameworks & Drivers層
│   ├── database.py                        # DB接続設定
│   └── dependencies.py                    # DI（依存性注入）設定
│
└── main.py                                # エントリーポイント
```

参考: [How To Implement Clean Architecture in FastAPI - Medium](https://medium.com/@bhagyasithumini/how-to-implement-clean-architecture-in-fastapi-a-step-by-step-guide-8b73a75c650b)

### Entities（`entities/user.py`）

```python
from dataclasses import dataclass, field
from uuid import UUID, uuid4


@dataclass
class User:
    name: str
    email: str
    id: UUID = field(default_factory=uuid4)

    def __post_init__(self) -> None:
        if not self.name:
            raise ValueError("ユーザー名は空にできません")
        if "@" not in self.email:
            raise ValueError("不正なメールアドレスです")

    def update_email(self, new_email: str) -> None:
        if "@" not in new_email:
            raise ValueError("不正なメールアドレスです")
        self.email = new_email
```

### Repository Interface（`use_cases/interfaces/user_repository.py`）

Use Cases 層が外部永続化層に対して期待する契約を定義する。依存性逆転の原則により、このインターフェースは Use Cases 層（内側）に配置される点が重要である。

```python
from abc import ABC, abstractmethod
from uuid import UUID

from entities.user import User


class UserRepositoryInterface(ABC):
    """Use Cases層が外部永続化に期待する契約。依存性逆転の原則を適用した抽象定義。"""

    @abstractmethod
    async def find_by_id(self, user_id: UUID) -> User | None:
        ...

    @abstractmethod
    async def find_by_email(self, email: str) -> User | None:
        ...

    @abstractmethod
    async def save(self, user: User) -> User:
        ...

    @abstractmethod
    async def delete(self, user_id: UUID) -> None:
        ...
```

### Use Case（`use_cases/user_use_cases.py`）

```python
from uuid import UUID

from entities.user import User
from use_cases.interfaces.user_repository import UserRepositoryInterface


class CreateUserUseCase:
    """ユーザー作成ユースケース。Entities と Repository Interface のみに依存する。"""

    def __init__(self, user_repository: UserRepositoryInterface) -> None:
        self._user_repository = user_repository

    async def execute(self, name: str, email: str) -> User:
        existing = await self._user_repository.find_by_email(email)
        if existing is not None:
            raise ValueError(f"メールアドレス {email} は既に使用されています")

        user = User(name=name, email=email)
        return await self._user_repository.save(user)


class GetUserUseCase:
    """ユーザー取得ユースケース。"""

    def __init__(self, user_repository: UserRepositoryInterface) -> None:
        self._user_repository = user_repository

    async def execute(self, user_id: UUID) -> User:
        user = await self._user_repository.find_by_id(user_id)
        if user is None:
            raise ValueError(f"ユーザー {user_id} が見つかりません")
        return user
```

### ORM Model（`interface_adapters/gateways/orm_models.py`）

```python
from uuid import UUID

from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class UserORM(Base):
    """SQLAlchemy ORM Model。Interface Adapters層に配置し、Entities層の User と分離する。"""
    __tablename__ = "users"

    id: Mapped[UUID] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(255), unique=True)
```

### FastAPI Router（Controller）（`interface_adapters/controllers/user_router.py`）

```python
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException, status
from pydantic import BaseModel, EmailStr

from frameworks_drivers.dependencies import get_create_user_use_case, get_get_user_use_case
from use_cases.user_use_cases import CreateUserUseCase, GetUserUseCase

router = APIRouter(prefix="/users", tags=["users"])


class CreateUserRequest(BaseModel):
    name: str
    email: EmailStr


class UserResponse(BaseModel):
    id: UUID
    name: str
    email: str

    model_config = {"from_attributes": True}


@router.post("/", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(
    request: CreateUserRequest,
    use_case: CreateUserUseCase = Depends(get_create_user_use_case),
) -> UserResponse:
    try:
        user = await use_case.execute(name=request.name, email=request.email)
        return UserResponse(id=user.id, name=user.name, email=user.email)
    except ValueError as e:
        raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail=str(e))


@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: UUID,
    use_case: GetUserUseCase = Depends(get_get_user_use_case),
) -> UserResponse:
    try:
        user = await use_case.execute(user_id=user_id)
        return UserResponse(id=user.id, name=user.name, email=user.email)
    except ValueError as e:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail=str(e))
```

### SQLAlchemy Repository（Gateway）（`interface_adapters/gateways/sqlalchemy_user_repository.py`）

```python
from uuid import UUID

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from entities.user import User
from interface_adapters.gateways.orm_models import UserORM
from use_cases.interfaces.user_repository import UserRepositoryInterface


class SQLAlchemyUserRepository(UserRepositoryInterface):
    """UserRepositoryInterface の SQLAlchemy 実装。Interface Adapters層に配置する。"""

    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def find_by_id(self, user_id: UUID) -> User | None:
        result = await self._session.execute(
            select(UserORM).where(UserORM.id == user_id)
        )
        orm_user = result.scalar_one_or_none()
        return self._to_entity(orm_user) if orm_user else None

    async def find_by_email(self, email: str) -> User | None:
        result = await self._session.execute(
            select(UserORM).where(UserORM.email == email)
        )
        orm_user = result.scalar_one_or_none()
        return self._to_entity(orm_user) if orm_user else None

    async def save(self, user: User) -> User:
        orm_user = UserORM(id=user.id, name=user.name, email=user.email)
        self._session.add(orm_user)
        await self._session.commit()
        await self._session.refresh(orm_user)
        return self._to_entity(orm_user)

    async def delete(self, user_id: UUID) -> None:
        result = await self._session.execute(
            select(UserORM).where(UserORM.id == user_id)
        )
        orm_user = result.scalar_one_or_none()
        if orm_user:
            await self._session.delete(orm_user)
            await self._session.commit()

    @staticmethod
    def _to_entity(orm_user: UserORM) -> User:
        return User(id=orm_user.id, name=orm_user.name, email=orm_user.email)
```

### DB接続設定（`frameworks_drivers/database.py`）

```python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

DATABASE_URL = "postgresql+asyncpg://user:password@localhost/dbname"

engine = create_async_engine(DATABASE_URL, echo=False)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)
```

### 依存性注入（`frameworks_drivers/dependencies.py`）

```python
from collections.abc import AsyncIterator

from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

from frameworks_drivers.database import AsyncSessionLocal
from interface_adapters.gateways.sqlalchemy_user_repository import SQLAlchemyUserRepository
from use_cases.user_use_cases import CreateUserUseCase, GetUserUseCase


async def get_db_session() -> AsyncIterator[AsyncSession]:
    async with AsyncSessionLocal() as session:
        yield session


def get_user_repository(
    session: AsyncSession = Depends(get_db_session),
) -> SQLAlchemyUserRepository:
    return SQLAlchemyUserRepository(session)


def get_create_user_use_case(
    repository: SQLAlchemyUserRepository = Depends(get_user_repository),
) -> CreateUserUseCase:
    return CreateUserUseCase(user_repository=repository)


def get_get_user_use_case(
    repository: SQLAlchemyUserRepository = Depends(get_user_repository),
) -> GetUserUseCase:
    return GetUserUseCase(user_repository=repository)
```

### テスト時の In-Memory Repository への差し替え例

依存性逆転により、テスト時は SQLAlchemy を使わず In-Memory 実装に差し替えられる。

```python
# tests/gateways/in_memory_user_repository.py
from uuid import UUID

from entities.user import User
from use_cases.interfaces.user_repository import UserRepositoryInterface


class InMemoryUserRepository(UserRepositoryInterface):
    """テスト用 In-Memory Repository。DB なしでテスト可能にする。"""

    def __init__(self) -> None:
        self._store: dict[UUID, User] = {}

    async def find_by_id(self, user_id: UUID) -> User | None:
        return self._store.get(user_id)

    async def find_by_email(self, email: str) -> User | None:
        return next((u for u in self._store.values() if u.email == email), None)

    async def save(self, user: User) -> User:
        self._store[user.id] = user
        return user

    async def delete(self, user_id: UUID) -> None:
        self._store.pop(user_id, None)


# tests/test_user_use_cases.py
import pytest
from use_cases.user_use_cases import CreateUserUseCase
from tests.gateways.in_memory_user_repository import InMemoryUserRepository


@pytest.mark.asyncio
async def test_create_user_success() -> None:
    repository = InMemoryUserRepository()
    use_case = CreateUserUseCase(user_repository=repository)

    user = await use_case.execute(name="テストユーザー", email="test@example.com")

    assert user.name == "テストユーザー"
    assert user.email == "test@example.com"
    assert user.id is not None


@pytest.mark.asyncio
async def test_create_user_duplicate_email() -> None:
    repository = InMemoryUserRepository()
    use_case = CreateUserUseCase(user_repository=repository)

    await use_case.execute(name="ユーザー1", email="dup@example.com")

    with pytest.raises(ValueError, match="既に使用されています"):
        await use_case.execute(name="ユーザー2", email="dup@example.com")
```

---

## 5. メリット・デメリット

### メリット

#### テスタビリティの向上

Dependency Rule により、Use Cases 層は Repository の具体的実装を知らない。
テスト時は In-Memory Repository に差し替えることで、データベースや外部 API なしにビジネスロジックの Unit Test が実行できる。

参考: [Clean Architecture Essentials: Transforming Python Development - Deep Engineering](https://deepengineering.substack.com/p/clean-architecture-essentials-transforming)

- Use Case の Unit Test が DB 接続なしで実行可能。
- 実行速度が速い（ネットワーク・I/O 不要）。
- フレームワーク非依存で CI/CD パイプラインでの実行が容易。

#### フレームワーク・技術スタックからの独立性

Frameworks & Drivers 層のみを差し替えることで、FastAPI から他のフレームワークへの移行や、PostgreSQL から MongoDB への変更が、ビジネスロジック（Entities / Use Cases）に影響を与えずに実現できる。

- データベースエンジン変更: `SQLAlchemyUserRepository` の差し替えのみ。
- Web フレームワーク変更: `interface_adapters/controllers/` の差し替えのみ。
- ビジネスロジック（Use Cases / Entities）への影響ゼロ。

#### ビジネスルールの可視性

同心円状の構造により、「何がビジネスルールで、何が技術的詳細か」が明確に区分される。
Entities 層と Use Cases 層を読むだけでシステムの本質的なビジネス要件が把握できる。

#### 長期的な保守性

依存方向が明確であるため、変更の影響範囲が予測しやすく、リグレッションが発生しにくい。
外部技術の変化（ORM・Web フレームワークのバージョンアップ・廃止）に対して、ビジネスロジックが保護される。

### デメリット

#### 学習コスト

Entities / Use Cases / Interface Adapters / Frameworks & Drivers という4層と、Dependency Rule・DIP の概念をチーム全員が正確に理解している必要がある。

#### コード量・複雑性の増大

「ユーザー」1件に関しても、Entity・Repository インターフェース・Use Case・ORM Model・SQLAlchemy 実装・FastAPI Router・Pydantic Schema の7ファイル以上が必要になる。
フィールド追加1件でも複数ファイルへの変更が必要になる場合がある。

#### 小規模アプリケーションへの過剰設計

CRUD 中心の単純な API や MVP 開発においては、このアーキテクチャの恩恵よりも構築コストが上回る可能性がある。

参考: [Pragmatic Clean Architecture in Python - Deep Engineering](https://deepengineering.substack.com/p/pragmatic-clean-architecture-in-python)

---

## 6. Anti-patterns（アンチパターン）

参考: [5 anti-patterns in Clean Architecture - Medium](https://medium.com/@takendra.saraswat224/5-anti-patterns-or-mistakes-developers-make-when-implementing-clean-architecture-in-android-apps-b3e80ec744fb)

### 1. Entities 層へのフレームワーク依存の混入

最も多いミス。Entity や Use Case に SQLAlchemy・FastAPI などの import が入り込む（Leaky Abstractions）。

```python
# Bad: Entity が SQLAlchemy ORM を継承している（Frameworks & Drivers の詳細が Entities 層に混入）
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy import Column, String

class User(DeclarativeBase):
    __tablename__ = "users"
    name = Column(String)

# Good: フレームワーク非依存の Pure Python
@dataclass
class User:
    name: str
    email: str
```

### 2. Dependency Rule の違反

内側の層が外側の層を直接 import する。

```python
# Bad: Use Cases 層が具体的な SQLAlchemy 実装（Interface Adapters層）に直接依存する
from interface_adapters.gateways.sqlalchemy_user_repository import SQLAlchemyUserRepository

class CreateUserUseCase:
    def __init__(self) -> None:
        self._repo = SQLAlchemyUserRepository()  # 具体的実装への依存

# Good: Use Cases 層はインターフェース（抽象）のみに依存する
from use_cases.interfaces.user_repository import UserRepositoryInterface

class CreateUserUseCase:
    def __init__(self, user_repository: UserRepositoryInterface) -> None:
        self._user_repository = user_repository
```

### 3. ビジネスロジックの配置誤り

ビジネスルール（重複チェック・バリデーションなど）が Controller や Repository に直接記述される。

```python
# Bad: Controller（Interface Adapters層）にビジネスロジックがある
@router.post("/users/")
async def create_user(request: CreateUserRequest, session: AsyncSession = Depends(get_db)):
    existing = await session.execute(select(UserORM).where(UserORM.email == request.email))
    if existing.scalar_one_or_none():
        raise HTTPException(400, "重複メールアドレス")  # ビジネスルールが Controller に記述されている
    ...

# Good: ビジネスロジックは Use Cases 層に集約する
class CreateUserUseCase:
    async def execute(self, name: str, email: str) -> User:
        existing = await self._user_repository.find_by_email(email)
        if existing is not None:
            raise ValueError(f"メールアドレス {email} は既に使用されています")
        ...
```

### 4. Anemic Entity（貧血エンティティ）

Entity がデータの入れ物（DTO）にすぎず、ビジネスルールが全て Use Cases 側に集中する状態。
Entity 自身がバリデーションとビジネスルールを持つべきである。

```python
# Bad: Entity がビジネスルールを持たない（ただのデータコンテナ）
@dataclass
class User:
    name: str
    email: str

# Good: Entity 自身がビジネスルールとバリデーションを持つ
@dataclass
class User:
    name: str
    email: str

    def __post_init__(self) -> None:
        if not self.name:
            raise ValueError("ユーザー名は空にできません")
        if "@" not in self.email:
            raise ValueError("不正なメールアドレスです")
```

### 5. Use Case 間の依存チェーン

1つの Use Case が他の Use Case を呼び出す構造はアンチパターン。
必要なデータは Repository から直接取得し、共通処理は Entity のメソッドまたは Domain Service に寄せる。

### 6. Entities 層の省略

UI 層がデータ層に直結した構造（Entities 層を省略）は、スケーラビリティとテスト性を損なう。
小規模プロジェクトであっても、ビジネスルールが存在する場合は Entities 層を保持することが推奨される。

### 7. 過剰設計の罠（CRUD アプリへの適用）

YAGNI・KISS 原則に反して、単純な CRUD API にクリーンアーキテクチャを強制適用することは推奨されない。

**適用が有効な状況**:
- 複雑なドメインロジックを持ち、長期間にわたりメンテナンスが必要なシステム。
- 複数の外部システム・データストアと統合が必要なアプリ。
- チームが大きく、ビジネスロジックとインフラを独立して開発・テストしたい場合。

**適用が不向きな状況**:
- シンプルな CRUD のみのマイクロサービス。
- MVP・PoC・スクリプト系アプリ。

---

## 7. バージョン情報

### FastAPI

| 項目 | バージョン |
|------|---------|
| 最新安定版 | **0.136.3**（2026年5月23日リリース） |
| 必要 Python バージョン | Python >= 3.10 |
| 推奨 Python バージョン | Python 3.12 または 3.13 |

参考: [FastAPI PyPI](https://pypi.org/project/fastapi/)

### SQLAlchemy

| 項目 | バージョン |
|------|---------|
| 最新安定版 | **2.0.50**（2026年5月24日リリース） |
| 必要 Python バージョン | Python >= 3.7（2.x 系） |
| 非同期サポート | asyncio 対応（`create_async_engine` / `AsyncSession`） |

参考: [SQLAlchemy PyPI](https://pypi.org/project/SQLAlchemy/)

### Python

| 項目 | バージョン |
|------|---------|
| 最新安定版 | **3.14.5**（2026年5月10日リリース） |
| 推奨（本番運用） | Python 3.12 〜 3.13（実績豊富な安定版）または 3.14（最新。2026年10月以降 security-fixes フェーズへ移行予定） |
| FastAPI 最低要件 | Python 3.10 以上 |

参考: [Python バージョン一覧 - devguide.python.org](https://devguide.python.org/versions/)

---

## 参考情報

- [The Clean Architecture - Robert C. Martin 公式ブログ](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Clean Architecture (software) - Wikipedia](https://en.wikipedia.org/wiki/Clean_architecture)
- [Clean Architecture vs. Onion Architecture vs. Hexagonal Architecture - CCD Akademie](https://ccd-akademie.de/en/clean-architecture-vs-onion-architecture-vs-hexagonal-architecture/)
- [Onion vs Clean vs Hexagonal Architecture - Medium](https://medium.com/@edamtoft/onion-vs-clean-vs-hexagonal-architecture-9ad94a27da91)
- [Layered vs Clean vs Onion vs Hexagonal Architecture — A Practical Guide - Medium](https://medium.com/@rup.singh88/stop-confusing-clean-onion-hexagonal-architecture-heres-when-to-use-each-692079e56267)
- [Clean Architecture vs Hexagonal vs Onion — Which One Should You Use? - Medium](https://medium.com/the-architecture-mindset/clean-architecture-vs-hexagonal-vs-onion-which-one-should-you-use-4f0f9b5c0d4a)
- [How To Implement Clean Architecture in FastAPI - Medium](https://medium.com/@bhagyasithumini/how-to-implement-clean-architecture-in-fastapi-a-step-by-step-guide-8b73a75c650b)
- [Building Scalable Python Services: FastAPI with Clean Architecture - Medium](https://medium.com/@ovindupathirana554/building-scalable-python-services-fastapi-with-clean-architecture-a31a0db6c923)
- [Building a Production-Ready FastAPI Boilerplate with Clean Architecture - DEV Community](https://dev.to/alwil17/building-a-production-ready-fastapi-boilerplate-with-clean-architecture-5757)
- [Clean Architecture Essentials: Transforming Python Development - Deep Engineering](https://deepengineering.substack.com/p/clean-architecture-essentials-transforming)
- [Pragmatic Clean Architecture in Python - Deep Engineering](https://deepengineering.substack.com/p/pragmatic-clean-architecture-in-python)
- [5 anti-patterns in Clean Architecture - Medium](https://medium.com/@takendra.saraswat224/5-anti-patterns-or-mistakes-developers-make-when-implementing-clean-architecture-in-android-apps-b3e80ec744fb)
- [GitHub: fastapi-clean-example (ivan-borovets)](https://github.com/ivan-borovets/fastapi-clean-example)
- [GitHub: fastapi-clean-example (0xTheProDev)](https://github.com/0xTheProDev/fastapi-clean-example)
- [FastAPI PyPI](https://pypi.org/project/fastapi/)
- [SQLAlchemy PyPI](https://pypi.org/project/SQLAlchemy/)
- [Python バージョン一覧 - devguide.python.org](https://devguide.python.org/versions/)

最終更新日: 2026/05/31
