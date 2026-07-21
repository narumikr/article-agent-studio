# レイヤードアーキテクチャ（N-Tier Architecture）

## 目次

1. [レイヤードアーキテクチャとは何か](#1-レイヤードアーキテクチャとは何か)
2. [主要構成要素](#2-主要構成要素)
3. [他のアーキテクチャとの比較](#3-他のアーキテクチャとの比較)
4. [FastAPIでの実装例](#4-fastapiでの実装例)
5. [メリット・デメリット](#5-メリットデメリット)
6. [Anti-patterns（アンチパターン）](#6-anti-patternsアンチパターン)
7. [バージョン情報](#7-バージョン情報)
8. [参考情報](#参考情報)

---

## 1. レイヤードアーキテクチャとは何か

### 概念・定義

レイヤードアーキテクチャ（Layered Architecture）は、ソフトウェアシステムをいくつかの水平な層（Layer）に分割し、各層が明確な責務を持ち、隣接する層とのみ通信するソフトウェア設計パターンである。

> "The layered architecture pattern is the most common architecture pattern, otherwise known as the n-tier architecture pattern, and is the de facto standard for most Java EE applications."
>
> — Mark Richards, [Software Architecture Patterns (O'Reilly, 2015)](https://www.oreilly.com/library/view/software-architecture-patterns/9781491971437/ch01.html)

Eric Evans は 2003 年の著書『Domain-Driven Design: Tackling Complexity in the Heart of Software』（通称「DDD本」）において、4層構成のレイヤードアーキテクチャを提唱した。

> "Partition a complex program into layers. Develop a design within each layer that is cohesive and that depends only on the layers below."
>
> — Eric Evans, [Domain-Driven Design: Tackling Complexity in the Heart of Software (2003)](https://www.informit.com/store/domain-driven-design-tackling-complexity-in-the-heart-9780321125217)

参考: [Layered architecture and Smart UI: Domain-Driven Design by Eric Evans - DEV Community](https://dev.to/ielgohary/layered-architecture-and-smart-ui-domain-driven-design-by-eric-evans-part-ii-chapter-4-1il7)

### 別名と意味

| 名称 | 説明 |
|------|------|
| **N-Tier Architecture** | 物理的なデプロイ層（Tier）に着目した呼び方。「Tier（ティア）」は物理的なサーバーやプロセスの分離を指すことが多い |
| **多層アーキテクチャ** | 日本語での一般的な呼称。N個の層に分かれた構造を表す |
| **Layered Architecture** | 論理的なコード構造の分離に着目した呼び方。「Layer（レイヤー）」は論理的なグループを指す |

**Layer と Tier の違い**

「Layer」と「Tier」はしばしば同義語として扱われるが、厳密には異なる概念である。

- **Layer（レイヤー）**: コードの論理的な分離。同一プロセス内に複数の Layer が存在し得る。
- **Tier（ティア）**: 物理的・デプロイ上の分離。異なるサーバーやプロセスで実行される。

3層の論理構造（3-Layer）を、1つの物理サーバー（1-Tier）にデプロイすることも可能である。

参考: [Multitier architecture - Wikipedia](https://en.wikipedia.org/wiki/Multitier_architecture)、[レイヤードアーキテクチャとマルチティアアーキテクチャは同じもの？ - Qiita](https://qiita.com/yonaka15/items/48eb6de2c05edad57c88)

### 誕生した背景・解決しようとした問題

**時代的な背景**

| 年代 | 状況 |
|------|------|
| 1960〜70年代 | GUI が存在せず、単一コンピュータ上の1層アプリケーションが主流 |
| 1980〜90年代 | クライアント/サーバー構成の普及により、プレゼンテーションとデータの2層分離が確立 |
| 1990年代後半〜2000年代前半 | Web ブラウザの普及に伴い、3層〜N層アーキテクチャへ進化 |
| 2003年 | Eric Evans の DDD 本により、4層構成（UI・Application・Domain・Infrastructure）が体系化 |

参考: [Layered Architecture – @hgraca](https://herbertograca.com/2017/08/03/layered-architecture/)

**解決しようとした問題**

- コードが混在し、UI・ビジネスロジック・データアクセスが密結合になっていた。
- 変更の影響範囲が不明瞭で、修正時に意図しないバグが発生しやすかった。
- テストが困難で、ビジネスロジックを単独で検証できなかった。
- 複数の開発者・チームが同一コードベースを修正する際の競合が多発していた。

---

## 2. 主要構成要素

### 構成図

```mermaid
graph TB
    subgraph Presentation["Presentation Layer（プレゼンテーション層）"]
        HTTP["HTTP Router / Controller"]
        VIEW["View / Template"]
    end

    subgraph Service["Business Logic Layer（ビジネスロジック層 / サービス層）"]
        SVC["Service / Application Service"]
        RULE["Business Rules"]
    end

    subgraph Repository["Data Access Layer（データアクセス層 / リポジトリ層）"]
        REPO["Repository / DAO"]
        ORM["ORM Model"]
    end

    subgraph Database["Database Layer（データベース層）"]
        DB[("Database\n(PostgreSQL / MySQL / etc.)")]
    end

    HTTP -->|"リクエスト"| SVC
    VIEW -->|"描画"| HTTP
    SVC -->|"データ操作"| REPO
    REPO -->|"SQL / クエリ"| DB
    DB -->|"結果"| REPO
    REPO -->|"ドメインオブジェクト"| SVC
    SVC -->|"レスポンス"| HTTP

    style Presentation fill:#4472C4,color:#fff
    style Service fill:#ED7D31,color:#fff
    style Repository fill:#70AD47,color:#fff
    style Database fill:#7030A0,color:#fff
```

### 各層の責務

#### Presentation Layer（プレゼンテーション層）

- **責務**: ユーザーまたは外部クライアントとのインターフェースを担う。HTTPリクエストの受け取りとレスポンスの返却、入力値のバリデーション（スキーマ検証）を行う。
- **含まれるもの**: FastAPI Router、Pydantic スキーマ（Request/Response DTO）、テンプレートエンジン
- **依存関係ルール**: Business Logic Layer（Service 層）のみに依存する。Repository や DB に直接アクセスしてはならない。
- **HTTP 知識**: HTTP ステータスコード、ヘッダー、クエリパラメータなどを理解する。

#### Business Logic Layer（ビジネスロジック層 / サービス層）

- **責務**: アプリケーションのビジネスルールを実装する。複数の Repository を組み合わせてユースケースを実現し、トランザクション管理を担う。
- **含まれるもの**: Service クラス、Application Service、バリデーションロジック、ビジネスルール
- **依存関係ルール**: Data Access Layer（Repository 層）のみに依存する。HTTP や フレームワーク固有の詳細（`Request` オブジェクトなど）を知ってはならない。
- **重要**: この層がアーキテクチャの中核であり、変更が最も少ない安定した層であるべきである。

#### Data Access Layer（データアクセス層 / リポジトリ層）

- **責務**: データの永続化・取得を担う。SQL クエリや ORM の操作を隠蔽し、上位層にはドメインオブジェクトとして返す。
- **含まれるもの**: Repository クラス、DAO（Data Access Object）、ORM Model（SQLAlchemy など）
- **依存関係ルール**: Database Layer（実際の DB）のみに依存する。ビジネスロジックを含んではならない。

#### Database Layer（データベース層）

- **責務**: データの永続化ストレージ。アプリケーションコードではなく、データベース製品（PostgreSQL、MySQL、SQLite など）が担う。
- **含まれるもの**: RDBMS、NoSQL、キャッシュストア（Redis など）
- **依存関係ルール**: アプリケーション層に依存しない。

### 開放層（Open Layer）と閉鎖層（Closed Layer）

レイヤードアーキテクチャには2種類のレイヤーアクセスモデルがある。

| モデル | 説明 | 特徴 |
|--------|------|------|
| **Closed Layer（閉鎖層）** | リクエストは必ず隣接する次の層を経由しなければならない | 厳密な分離・テスト容易・翻訳コードが増加 |
| **Open Layer（開放層）** | リクエストが特定の層をスキップできる | 柔軟だが、カップリングが増大しやすい |

参考: [N 層アーキテクチャ様式 - Azure Architecture Center | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/architecture/guide/architecture-styles/n-tier)

---

## 3. 他のアーキテクチャとの比較

### Hexagonal Architecture（ヘキサゴナルアーキテクチャ）との比較

参考: [Hexagonal/Clean Architecture vs Layered/N-Tier Architecture - systemsarchitect.io](https://www.systemsarchitect.io/blog/hexagonal-clean-architecture-vs-layered-n-tier-architecture-dc025)

| 観点 | Layered Architecture | Hexagonal Architecture |
|------|---------------------|----------------------|
| 提唱者・年 | Eric Evans（2003年 DDD本で体系化） | Alistair Cockburn（2005年） |
| 別名 | N-Tier Architecture | Ports and Adapters |
| 依存の方向 | 上位層から下位層への一方向 | 全て Core（ビジネスロジック）に向かって内向き |
| テスト容易性 | 下位層（DB など）への依存でテストが困難になる場合がある | Port を Mock / In-Memory 実装に差し替え可能 |
| DB 変更への対応 | Repository 層の変更が Service 層に波及する可能性がある | Driven Adapter のみ差し替えればよい |
| インターフェースの使用 | 任意（使わない実装も多い） | Secondary Port は必ずインターフェースとして定義 |
| 学習コスト | 低い（構造がシンプル） | 中程度（Port/Adapter の概念理解が必要） |
| 適した場面 | CRUD 中心のシンプルなアプリ | 複雑なドメインロジック・外部システム連携が多いアプリ |

### Clean Architecture との比較

Clean Architecture は Robert C. Martin（Uncle Bob）が 2012 年に提唱。内向き依存の原則はヘキサゴナルアーキテクチャと共通だが、同心円状の4層（Entities / Use Cases / Interface Adapters / Frameworks & Drivers）を明示的に定義している。

参考: [Layered vs Clean vs Onion vs Hexagonal Architecture — A Practical Guide - Medium](https://medium.com/@rup.singh88/stop-confusing-clean-onion-hexagonal-architecture-heres-when-to-use-each-692079e56267)

| 観点 | Layered Architecture | Clean Architecture |
|------|---------------------|-------------------|
| 提唱者・年 | Eric Evans（2003年 DDD本で体系化） | Robert C. Martin（2012年） |
| 依存の方向 | 上位層→下位層（線形） | 全て最内層（Entities）に向かって内向き |
| ビジネスロジックの位置 | Service 層（中間層） | 最内層（Entities / Use Cases）|
| DB・フレームワークの位置 | Data Access 層（下位層） | 最外層（Frameworks & Drivers） |
| テスト容易性 | 中程度 | 高い（最内層はフレームワーク非依存） |
| 複雑さ | 低い | 高い（明確な4層 + 依存性逆転の徹底） |
| 適した場面 | 中小規模・シンプルなシステム | 長期運用が前提の大規模システム |

### Onion Architecture との比較

Onion Architecture は Jeffrey Palermo が 2008 年に提唱。ドメインモデルを中心に置き、Clean Architecture に近い同心円状の構造を採用している。

参考: [Layered vs Clean vs Onion vs Hexagonal Architecture — A Practical Guide - Medium](https://medium.com/@rup.singh88/stop-confusing-clean-onion-hexagonal-architecture-heres-when-to-use-each-692079e56267)

| 観点 | Layered Architecture | Onion Architecture |
|------|---------------------|-------------------|
| 提唱者・年 | Eric Evans（2003年 DDD本で体系化） | Jeffrey Palermo（2008年） |
| 中心概念 | 層ごとの責務分離 | ドメインモデルを最内層に配置 |
| 依存の方向 | 上位層→下位層（線形） | 全て最内層（Domain Model）に向かって内向き |
| Infrastructure の位置 | 下位層（Data Access Layer） | 最外層（Infrastructure Layer） |
| DDD との親和性 | 中程度 | 高い（ドメインモデルが中核） |
| 複雑さ | 低い | 中〜高（DDD の概念理解が必要） |
| 適した場面 | CRUD 中心 | ドメイン知識が複雑なシステム（DDD 適用時） |

---

## 4. FastAPIでの実装例

### ディレクトリ構成例

```
my_app/
├── presentation/                   # Presentation Layer（プレゼンテーション層）
│   └── routers/
│       └── user_router.py          # FastAPI Router（エンドポイント定義）
│
├── service/                        # Business Logic Layer（ビジネスロジック層）
│   └── user_service.py             # UserService（ビジネスロジック）
│
├── repository/                     # Data Access Layer（データアクセス層）
│   ├── user_repository.py          # UserRepository（DB操作）
│   └── models.py                   # SQLAlchemy ORM Model
│
├── schemas/                        # Pydantic スキーマ（Request / Response DTO）
│   └── user_schema.py
│
├── database.py                     # DB接続設定
├── dependencies.py                 # 依存性注入の設定
└── main.py                         # エントリーポイント
```

### Pydantic スキーマ（`schemas/user_schema.py`）

```python
from uuid import UUID

from pydantic import BaseModel, EmailStr


class UserCreateRequest(BaseModel):
    """ユーザー作成リクエストのスキーマ"""
    name: str
    email: EmailStr


class UserResponse(BaseModel):
    """ユーザーレスポンスのスキーマ"""
    id: UUID
    name: str
    email: str

    model_config = {"from_attributes": True}
```

### SQLAlchemy ORM Model（`repository/models.py`）

```python
import uuid

from sqlalchemy import String
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class UserORM(Base):
    """SQLAlchemy ORM Model。データベーステーブルのマッピング。"""
    __tablename__ = "users"

    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), primary_key=True, default=uuid.uuid4
    )
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    email: Mapped[str] = mapped_column(String(255), nullable=False, unique=True)
```

### Repository（`repository/user_repository.py`）

```python
from uuid import UUID

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from repository.models import UserORM


class UserRepository:
    """Data Access Layer: DBアクセスを担当する。ビジネスロジックを含まない。"""

    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def find_by_id(self, user_id: UUID) -> UserORM | None:
        result = await self._session.execute(
            select(UserORM).where(UserORM.id == user_id)
        )
        return result.scalar_one_or_none()

    async def find_by_email(self, email: str) -> UserORM | None:
        result = await self._session.execute(
            select(UserORM).where(UserORM.email == email)
        )
        return result.scalar_one_or_none()

    async def create(self, name: str, email: str) -> UserORM:
        user = UserORM(name=name, email=email)
        self._session.add(user)
        await self._session.commit()
        await self._session.refresh(user)
        return user

    async def delete(self, user_id: UUID) -> None:
        user = await self.find_by_id(user_id)
        if user:
            await self._session.delete(user)
            await self._session.commit()
```

### Service（`service/user_service.py`）

```python
from uuid import UUID

from repository.models import UserORM
from repository.user_repository import UserRepository


class UserService:
    """Business Logic Layer: ビジネスルールを実装する。HTTPやDB詳細を知らない。"""

    def __init__(self, user_repository: UserRepository) -> None:
        self._user_repository = user_repository

    async def create_user(self, name: str, email: str) -> UserORM:
        existing = await self._user_repository.find_by_email(email)
        if existing is not None:
            raise ValueError(f"メールアドレス {email} は既に使用されています")
        if not name:
            raise ValueError("ユーザー名は空にできません")
        return await self._user_repository.create(name=name, email=email)

    async def get_user(self, user_id: UUID) -> UserORM:
        user = await self._user_repository.find_by_id(user_id)
        if user is None:
            raise ValueError(f"ユーザー {user_id} が見つかりません")
        return user

    async def delete_user(self, user_id: UUID) -> None:
        user = await self._user_repository.find_by_id(user_id)
        if user is None:
            raise ValueError(f"ユーザー {user_id} が見つかりません")
        await self._user_repository.delete(user_id)
```

### FastAPI Router（`presentation/routers/user_router.py`）

```python
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException, status

from dependencies import get_user_service
from schemas.user_schema import UserCreateRequest, UserResponse
from service.user_service import UserService

router = APIRouter(prefix="/users", tags=["users"])


@router.post("/", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(
    request: UserCreateRequest,
    user_service: UserService = Depends(get_user_service),
) -> UserResponse:
    try:
        user = await user_service.create_user(name=request.name, email=request.email)
        return UserResponse(id=user.id, name=user.name, email=user.email)
    except ValueError as e:
        raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail=str(e))


@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: UUID,
    user_service: UserService = Depends(get_user_service),
) -> UserResponse:
    try:
        user = await user_service.get_user(user_id=user_id)
        return UserResponse(id=user.id, name=user.name, email=user.email)
    except ValueError as e:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail=str(e))


@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(
    user_id: UUID,
    user_service: UserService = Depends(get_user_service),
) -> None:
    try:
        await user_service.delete_user(user_id=user_id)
    except ValueError as e:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail=str(e))
```

### 依存性注入（`dependencies.py`）

```python
from collections.abc import AsyncIterator

from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from repository.user_repository import UserRepository
from service.user_service import UserService

DATABASE_URL = "postgresql+asyncpg://USER:PASSWORD@localhost/DBNAME"

engine = create_async_engine(DATABASE_URL, echo=True)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)


async def get_db_session() -> AsyncIterator[AsyncSession]:  # AsyncGenerator より AsyncIterator が推奨（FastAPI Depends との型チェック互換性）
    async with AsyncSessionLocal() as session:
        yield session


def get_user_repository(
    session: AsyncSession = Depends(get_db_session),
) -> UserRepository:
    return UserRepository(session)


def get_user_service(
    user_repository: UserRepository = Depends(get_user_repository),
) -> UserService:
    return UserService(user_repository=user_repository)
```

### エントリーポイント（`main.py`）

```python
from fastapi import FastAPI

from presentation.routers.user_router import router as user_router

app = FastAPI(title="Layered Architecture Example")

app.include_router(user_router)
```

---

## 5. メリット・デメリット

### メリット

#### シンプルさと学習コストの低さ

構造が直感的でわかりやすく、ジュニア開発者でも容易に把握できる。多くのチュートリアルやフレームワークのデフォルト構成がこのパターンに沿っているため、学習コストが低い。

参考: [Layered Architecture: The Traditional N-Tier Pattern - Bitloops](https://bitloops.com/resources/software-architecture/layered-architecture)

- Web 系フレームワーク（Django、FastAPI、Spring など）のデフォルト構成と親和性が高い。
- チームの新メンバーがコードベースを理解しやすい。

#### 関心の分離（Separation of Concerns）

各層が明確な責務を持つため、変更の影響範囲が限定される。

参考: [N 層アーキテクチャ様式 - Azure Architecture Center | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/architecture/guide/architecture-styles/n-tier)

- Presentation 層は HTTP のみを担当し、ビジネスルールを知らない。
- DB を PostgreSQL から MySQL に変更する場合、Repository 層の変更のみで済む。

#### チーム分業のしやすさ

層ごとに担当チームを分けることができる。フロントエンドエンジニア（Presentation 層）、バックエンドエンジニア（Service 層）、DBA（Database 層）などが独立して作業できる。

#### 段階的な移行のしやすさ

既存のモノリシックアプリケーションからの移行が容易で、オンプレミスアプリケーションのクラウド移行にも自然に適合する。

参考: [N 層アーキテクチャ様式 - Azure Architecture Center | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/architecture/guide/architecture-styles/n-tier)

---

### デメリット

#### 密結合の発生リスク

層間のインターフェースを厳密に定義しない場合、Presentation 層が Repository 層に直接依存するなど、層を超えた密結合が発生しやすい。

参考: [Layered Architecture: The Traditional N-Tier Pattern - Bitloops](https://bitloops.com/resources/software-architecture/layered-architecture)

#### スケールの困難性

大規模システムになると、1つの機能追加に全層の変更が必要になり、複数チーム間の調整コストが急増する。

#### テスタビリティの制限

Service 層が Repository 層（具体的な DB 実装）に直接依存している場合、DB なしでのユニットテストが困難になる。インターフェースを使用しないシンプルな実装ではモックへの差し替えも手間がかかる。

参考: [Layered Architecture – @hgraca](https://herbertograca.com/2017/08/03/layered-architecture/)

#### パフォーマンスオーバーヘッド

厳密な閉鎖層モデル（Closed Layer）を採用した場合、データが不必要な層を通過することで待機時間とオーバーヘッドが発生する可能性がある。

---

## 6. Anti-patterns（アンチパターン）

### 1. Architecture Sinkhole（アーキテクチャシンクホール）

最も代表的なアンチパターン。ビジネスロジックを一切処理せずに、リクエストが全層を単純に通過する状態。

参考: [Architecture Sinkhole Anti-Pattern - Candost's Blog](https://candost.blog/notes/45a/)、[1. Layered Architecture - Software Architecture Patterns (O'Reilly)](https://www.oreilly.com/library/view/software-architecture-patterns/9781491971437/ch01.html)

```python
# Bad: Service がロジックなしに Repository を呼び出すだけ（シンクホール）
class UserService:
    async def get_user(self, user_id: UUID) -> UserORM:
        # ビジネスロジックが存在せず、Repository を呼び出すだけ
        return await self._user_repository.find_by_id(user_id)

# Good: Service が適切なビジネスルール（存在チェック、権限確認など）を実装する
class UserService:
    async def get_user(self, user_id: UUID) -> UserORM:
        user = await self._user_repository.find_by_id(user_id)
        if user is None:
            raise ValueError(f"ユーザー {user_id} が見つかりません")
        # 必要に応じてアクセス制御や追加のバリデーションを実施
        return user
```

**判断基準**: リクエストの最大 20% がシンクホール状態であれば許容範囲とされる。しかし 80% 以上がシンクホールであれば、レイヤードアーキテクチャが適切でない可能性がある（open layer の検討、またはアーキテクチャの見直しが必要）。

### 2. Layer Skipping（層スキップ）

Presentation 層が Repository 層や DB に直接アクセスするなど、中間層を飛び越えて依存する。

```python
# Bad: Router が直接 Repository に依存している（Service 層をスキップ）
@router.get("/{user_id}")
async def get_user(
    user_id: UUID,
    user_repository: UserRepository = Depends(get_user_repository),  # 層スキップ
) -> UserResponse:
    user = await user_repository.find_by_id(user_id)
    ...

# Good: Router は Service 層のみに依存する
@router.get("/{user_id}")
async def get_user(
    user_id: UUID,
    user_service: UserService = Depends(get_user_service),  # 正しい依存
) -> UserResponse:
    user = await user_service.get_user(user_id=user_id)
    ...
```

### 3. Fat Service Layer（肥大化したサービス層）

Service クラスに全てのロジックが集中し、1 つのクラスが数百〜数千行になる God Object（神クラス）状態。

```python
# Bad: 1つの Service クラスに全ての責務が集中している
class ApplicationService:
    async def create_user(self, ...): ...
    async def get_user(self, ...): ...
    async def create_order(self, ...): ...
    async def process_payment(self, ...): ...
    async def send_notification(self, ...): ...
    async def generate_report(self, ...): ...
    # ... 数百のメソッド

# Good: 責務ごとに Service クラスを分割する
class UserService:
    async def create_user(self, ...): ...
    async def get_user(self, ...): ...

class OrderService:
    async def create_order(self, ...): ...

class PaymentService:
    async def process_payment(self, ...): ...
```

### 4. Business Logic in Repository（Repository へのビジネスロジック混入）

Repository 層にビジネスロジック（重複チェック、権限確認など）が混入する。

```python
# Bad: Repository にビジネスルール（重複チェック）が含まれている
class UserRepository:
    async def create(self, name: str, email: str) -> UserORM:
        # ビジネスロジックが Repository に入り込んでいる
        existing = await self.find_by_email(email)
        if existing:
            raise ValueError("メールアドレスが重複しています")
        ...

# Good: ビジネスロジックは Service 層に配置し、Repository は純粋なデータアクセスのみ
class UserRepository:
    async def create(self, name: str, email: str) -> UserORM:
        user = UserORM(name=name, email=email)
        self._session.add(user)
        await self._session.commit()
        return user

class UserService:
    async def create_user(self, name: str, email: str) -> UserORM:
        existing = await self._user_repository.find_by_email(email)
        if existing:
            raise ValueError("メールアドレスが重複しています")
        return await self._user_repository.create(name=name, email=email)
```

### 5. HTTP 詳細の Service 層への混入

Service 層に HTTP の詳細（`Request` オブジェクト、HTTPException など）が混入する。

```python
# Bad: Service が FastAPI の HTTPException を raise している
from fastapi import HTTPException

class UserService:
    async def get_user(self, user_id: UUID) -> UserORM:
        user = await self._user_repository.find_by_id(user_id)
        if user is None:
            raise HTTPException(status_code=404, detail="Not Found")  # HTTP詳細がService層に混入
        return user

# Good: Service はドメイン例外を raise し、Router で HTTP 例外に変換する
class UserService:
    async def get_user(self, user_id: UUID) -> UserORM:
        user = await self._user_repository.find_by_id(user_id)
        if user is None:
            raise ValueError(f"ユーザー {user_id} が見つかりません")  # ドメイン例外
        return user

# Router 側で HTTP 例外に変換する
@router.get("/{user_id}")
async def get_user(user_id: UUID, user_service: UserService = Depends(get_user_service)):
    try:
        user = await user_service.get_user(user_id=user_id)
        ...
    except ValueError as e:
        raise HTTPException(status_code=404, detail=str(e))
```

### 6. Lasagna Architecture（ラザニアアーキテクチャ）

不必要に多くの層を追加し、小さな処理でも多数の層を経由する過剰設計。必要以上に層を細分化することで複雑性だけが増加する。

参考: [Layered Architecture – @hgraca](https://herbertograca.com/2017/08/03/layered-architecture/)

```python
# Bad: 単純な CRUD に不必要な層を追加している
# Presentation → Controller → Facade → Manager → Service → Helper → Repository → DAO → DB

# Good: 必要最小限の層のみを使用する
# Presentation（Router） → Service → Repository → DB
```

---

## 7. バージョン情報

### FastAPI

| 項目 | バージョン |
|------|---------|
| 最新安定版 | **0.136.3**（2026年5月23日リリース） |
| 必要 Python バージョン | Python >= 3.10 |
| 推奨 Python バージョン | Python 3.12 または 3.13 |

参考: [FastAPI PyPI](https://pypi.org/project/fastapi/)

### Python

| 項目 | バージョン |
|------|---------|
| 最新安定版 | **3.14**（2025年10月7日リリース） |
| 推奨（本番運用） | Python 3.12 〜 3.13（実績豊富な安定版）または 3.14（最新。2026年10月以降 security-fixes フェーズへ移行予定） |
| FastAPI 最低要件 | Python 3.10 以上 |

参考: [Python バージョン一覧 - devguide.python.org](https://devguide.python.org/versions/)

---

## 参考情報

- [1. Layered Architecture - Software Architecture Patterns (O'Reilly)](https://www.oreilly.com/library/view/software-architecture-patterns/9781491971437/ch01.html)
- [Layered Architecture: The Traditional N-Tier Pattern - Bitloops](https://bitloops.com/resources/software-architecture/layered-architecture)
- [Layered Architecture – @hgraca](https://herbertograca.com/2017/08/03/layered-architecture/)
- [Multitier architecture - Wikipedia](https://en.wikipedia.org/wiki/Multitier_architecture)
- [N 層アーキテクチャ様式 - Azure Architecture Center | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/architecture/guide/architecture-styles/n-tier)
- [Layered architecture and Smart UI: Domain-Driven Design by Eric Evans - DEV Community](https://dev.to/ielgohary/layered-architecture-and-smart-ui-domain-driven-design-by-eric-evans-part-ii-chapter-4-1il7)
- [Domain-Driven Design: Tackling Complexity in the Heart of Software - InformIT](https://www.informit.com/store/domain-driven-design-tackling-complexity-in-the-heart-9780321125217)
- [Layered vs Clean vs Onion vs Hexagonal Architecture — A Practical Guide - Medium](https://medium.com/@rup.singh88/stop-confusing-clean-onion-hexagonal-architecture-heres-when-to-use-each-692079e56267)
- [Hexagonal/Clean Architecture vs Layered/N-Tier Architecture - systemsarchitect.io](https://www.systemsarchitect.io/blog/hexagonal-clean-architecture-vs-layered-n-tier-architecture-dc025)
- [Architecture Sinkhole Anti-Pattern - Candost's Blog](https://candost.blog/notes/45a/)
- [Layered Architecture & Dependency Injection: A Recipe for Clean and Testable FastAPI Code - DEV Community](https://dev.to/markoulis/layered-architecture-dependency-injection-a-recipe-for-clean-and-testable-fastapi-code-3ioo)
- [The Service Layer Pattern - Marc Puig - Notes](https://mpuig.github.io/Notes/fastapi_basics/04.service_layer_pattern/)
- [GitHub - teamhide/fastapi-layered-architecture](https://github.com/teamhide/fastapi-layered-architecture)
- [FastAPI PyPI](https://pypi.org/project/fastapi/)
- [Python バージョン一覧 - devguide.python.org](https://devguide.python.org/versions/)
- [レイヤードアーキテクチャとマルチティアアーキテクチャは同じもの？ - Qiita](https://qiita.com/yonaka15/items/48eb6de2c05edad57c88)

最終更新日: 2026/05/28
