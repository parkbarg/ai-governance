
# ユーザー・部署管理DBと期間検索の実装

技術構成：
- Frontend：Vue.js + TypeScript
- Backend：FastAPI
- Database：PostgreSQL
- DBアクセス：SQLAlchemy

## 1. テーブル作成SQL

### 1-1. users（ユーザー）

```sql
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,
    starton DATE NOT NULL,
    endon DATE,

    CONSTRAINT uq_users_email_endon
        UNIQUE (email, endon),

    CONSTRAINT chk_users_period
        CHECK (endon IS NULL OR starton <= endon)
);

-- 在籍中のメールアドレス重複を禁止
CREATE UNIQUE INDEX uq_users_active_email
ON users (email)
WHERE endon IS NULL;
```

- `id`：主キー
- `email, endon`：複合ユニークキー
- `starton`：在籍開始日
- `endon`：在籍終了日（NULLは終了日未設定）
- 退職後、同じメールアドレスを別ユーザーが再利用可能

※ 通常のUNIQUE制約ではNULL同士は重複とみなされないため、在籍中のメールアドレスを一意にする部分ユニークインデックスを追加する。

### 1-2. department（部署）

```sql
CREATE TABLE department (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    starton DATE NOT NULL,
    endon DATE,

    CONSTRAINT chk_department_period
        CHECK (endon IS NULL OR starton <= endon)
);
```

- `id`：主キー
- `name`：ユニークキー
- `starton`：部署の有効開始日
- `endon`：部署の有効終了日（NULLは終了日未設定）

過去に存在した部署も削除せず、終了日を設定して保存する。

### 1-3. user_department（ユーザー所属履歴）

```sql
CREATE TABLE user_department (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id BIGINT NOT NULL,
    department_name VARCHAR(100) NOT NULL,
    starton DATE NOT NULL,
    endon DATE,

    CONSTRAINT fk_ud_user
        FOREIGN KEY (user_id)
        REFERENCES users(id),

    CONSTRAINT fk_ud_department
        FOREIGN KEY (department_name)
        REFERENCES department(name)
        ON UPDATE CASCADE,

    CONSTRAINT chk_ud_period
        CHECK (endon IS NULL OR starton <= endon)
);

-- JOIN用インデックス
CREATE INDEX idx_ud_user_id
ON user_department (user_id);

CREATE INDEX idx_ud_department_name
ON user_department (department_name);

-- 期間検索用インデックス
CREATE INDEX idx_ud_period
ON user_department (starton, endon);
```

- `id`：主キー
- `user_id`：users.idへの外部キー
- `department_name`：department.nameへの外部キー
- `starton`：所属開始日
- `endon`：所属終了日

---

## 2. クエリ（PostgreSQL）

### 2-1. 指定期間に所属していたユーザーの取得

検索条件：
- 指定期間と所属期間が1日でも重なれば取得
- 判定にはuser_departmentの期間のみ使用
- ユーザーの在籍期間は判定しない

```sql
SELECT
    u.id AS user_id,
    u.name,
    u.email,
    ud.department_name,
    ud.starton,
    ud.endon
FROM users AS u
JOIN user_department AS ud
    ON u.id = ud.user_id
WHERE
    ud.starton <= :end_date
    AND (
        ud.endon >= :start_date
        OR ud.endon IS NULL
    )
ORDER BY u.id, ud.starton, ud.id;
```

期間判定の意味：

```sql
ud.starton <= :end_date
```

所属開始日が検索終了日以前。

```sql
ud.endon >= :start_date
OR ud.endon IS NULL
```

所属終了日が検索開始日以降、または終了日未設定。

この2条件をANDで結合することで、期間が1日でも重なる所属履歴を取得する。

### 2-2. 指定期間に存在していた部署一覧の取得

目的：
- フロントエンドに表示する部署ボタンを取得
- 現在廃止されている部署も、指定期間に存在していれば表示
- 部署の有効期間で判断する

```sql
SELECT
    id,
    name,
    starton,
    endon
FROM department
WHERE
    starton <= :end_date
    AND (
        endon >= :start_date
        OR endon IS NULL
    )
ORDER BY name;
```

例：

| 部署 | starton | endon |
|---|---|---|
| APD | 2024-01-01 | 2025-12-31 |
| 開発部 | 2025-01-01 | NULL |
| 営業部 | 2026-01-01 | NULL |

2025年で検索：
- APD：表示
- 開発部：表示
- 営業部：非表示

2026年で検索：
- APD：非表示
- 開発部：表示
- 営業部：表示

### 2-3. 指定期間と部署名によるユーザー検索

目的：
- 部署ボタンを押した場合、その部署に所属していたユーザーを取得
- 廃止された部署でも過去の所属履歴から検索可能

```sql
SELECT
    u.id AS user_id,
    u.name,
    u.email,
    ud.department_name,
    ud.starton,
    ud.endon
FROM users AS u
JOIN user_department AS ud
    ON u.id = ud.user_id
WHERE
    ud.starton <= :end_date
    AND (
        ud.endon >= :start_date
        OR ud.endon IS NULL
    )
    AND ud.department_name = :department_name
ORDER BY u.id, ud.starton, ud.id;
```

追加条件：

```sql
AND ud.department_name = :department_name
```

例えば以下の条件を指定する。

```text
start_date      = 2025-01-01
end_date        = 2025-12-31
department_name = APD
```

結果：
2025年中に1日でもAPDに所属していたユーザーを取得する。

### 2-4. 部署名を任意条件にする方法

部署名が指定された場合だけ絞り込み、未指定の場合は全部署を対象とする。

```sql
SELECT
    u.id AS user_id,
    u.name,
    u.email,
    ud.department_name,
    ud.starton,
    ud.endon
FROM users AS u
JOIN user_department AS ud
    ON u.id = ud.user_id
WHERE
    ud.starton <= :end_date
    AND (
        ud.endon >= :start_date
        OR ud.endon IS NULL
    )
    AND (
        :department_name IS NULL
        OR ud.department_name = :department_name
    )
ORDER BY u.id, ud.starton, ud.id;
```

- `department_name = NULL`：全部署
- `department_name = 'APD'`：APDのみ

SQLAlchemy側からNULLをバインドして実行できる。

---

## 3. Vue.jsから受け取った期間をFastAPI経由で検索に反映

### 3-1. 全体の処理フロー

```text
Vue.js
  │
  ├─ 開始日・終了日を選択
  │
  ├─ GET /api/departments
  │    ?start_date=2025-01-01
  │    &end_date=2025-12-31
  │
  ▼
FastAPI
  │
  ├─ 期間の検証
  ├─ PostgreSQLで部署検索
  │
  ▼
Vue.js
  │
  ├─ 指定期間の部署ボタンを表示
  │
  ├─ APDボタンをクリック
  │
  ├─ GET /api/users
  │    ?start_date=2025-01-01
  │    &end_date=2025-12-31
  │    &department_name=APD
  │
  ▼
FastAPI
  │
  ├─ パラメータ受信・検証
  ├─ SQLAlchemyで条件をバインド
  │
  ▼
PostgreSQL
  │
  ├─ usersとuser_departmentをJOIN
  ├─ 所属期間で絞り込み
  ├─ 部署名で絞り込み
  │
  ▼
FastAPI
  │
  ├─ 検索結果をJSONに変換
  │
  ▼
Vue.js
  │
  └─ 該当ユーザーを表示
```

### 3-2. データベース接続

database.py

```python
import os
from collections.abc import Generator

from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

DATABASE_URL = os.environ["DATABASE_URL"]

engine = create_engine(
    DATABASE_URL,
    pool_pre_ping=True
)

SessionLocal = sessionmaker(bind=engine)


def get_db() -> Generator[Session, None, None]:
    with SessionLocal() as db:
        yield db
```

環境変数例：

```text
DATABASE_URL=postgresql+psycopg://postgres:password@localhost:5432/company_db
```

### 3-3. FastAPIで期間と部署名を受け取る

main.py

```python
from datetime import date
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException, Query
from fastapi.middleware.cors import CORSMiddleware
from sqlalchemy import text
from sqlalchemy.orm import Session

from .database import get_db

app = FastAPI()

# Vue.js開発サーバーからのアクセスを許可
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],
    allow_methods=["GET"],
    allow_headers=["*"],
)


def validate_period(start_date: date, end_date: date):
    if start_date > end_date:
        raise HTTPException(
            status_code=422,
            detail="開始日は終了日以前にしてください"
        )


# ① 指定期間に存在していた部署の取得
@app.get("/api/departments")
def get_departments(
    start_date: Annotated[date, Query()],
    end_date: Annotated[date, Query()],
    db: Annotated[Session, Depends(get_db)],
):
    validate_period(start_date, end_date)

    sql = text("""
        SELECT
            id,
            name,
            starton,
            endon
        FROM department
        WHERE
            starton <= :end_date
            AND (
                endon >= :start_date
                OR endon IS NULL
            )
        ORDER BY name
    """)

    result = db.execute(
        sql,
        {
            "start_date": start_date,
            "end_date": end_date
        }
    )

    return [dict(row) for row in result.mappings()]


# ② 指定期間と部署によるユーザー検索
@app.get("/api/users")
def get_users(
    start_date: Annotated[date, Query()],
    end_date: Annotated[date, Query()],
    db: Annotated[Session, Depends(get_db)],
    department_name: Annotated[
        str | None, Query(min_length=1, max_length=100)
    ] = None,
):
    validate_period(start_date, end_date)

    # SQL
    sql = text("""
        SELECT
            u.id AS user_id,
            u.name,
            u.email,
            ud.department_name,
            ud.starton,
            ud.endon
        FROM users AS u
        JOIN user_department AS ud
            ON u.id = ud.user_id
        WHERE
            ud.starton <= :end_date
            AND (
                ud.endon >= :start_date
                OR ud.endon IS NULL
            )
            AND (
                :department_name IS NULL
                OR ud.department_name = :department_name
            )
        ORDER BY u.id, ud.starton, ud.id
    """)

    # フロントエンドから受け取った値を渡す
    result = db.execute(
        sql,
        {
            "start_date": start_date,
            "end_date": end_date,
            "department_name": department_name
        }
    )

    return [dict(row) for row in result.mappings()]
```

### 3-4. FastAPIの重要な処理

#### ① クエリパラメータの取得

```python
start_date: Annotated[date, Query()]
end_date: Annotated[date, Query()]
```

FastAPIがHTTPのクエリパラメータをPythonのdate型に変換する。

例えば：

```http
GET /api/users?start_date=2025-01-01&end_date=2025-12-31
```

内部的には以下の値として扱える。

```python
start_date = date(2025, 1, 1)
end_date = date(2025, 12, 31)
```

日付形式が不正な場合はHTTP 422となる。

#### ② 部署名の取得

```python
department_name: Annotated[
    str | None, Query(min_length=1, max_length=100)
] = None
```

部署名は任意パラメータ。

- 未指定：全部署
- APD指定：APDのみ

#### ③ データベースSessionの取得

```python
db: Annotated[Session, Depends(get_db)]
```

`Depends(get_db)` によってデータベースSessionを取得する。

API処理終了後、Sessionを閉じる。

#### ④ SQLへの値のバインド

```python
result = db.execute(
    sql,
    {
        "start_date": start_date,
        "end_date": end_date,
        "department_name": department_name
    }
)
```

フロントエンドから受け取った値をSQLのバインドパラメータに渡す。

- `:start_date`
- `:end_date`
- `:department_name`

値を直接SQL文字列に連結しないため、SQLインジェクション対策にもなる。

#### ⑤ JSONへの変換

```python
return [dict(row) for row in result.mappings()]
```

検索結果を辞書のリストとして返し、FastAPIがJSONに変換する。

### 3-5. Vue.jsからAPIを呼び出す

```typescript
// 指定期間
const startDate = '2025-01-01'
const endDate = '2025-12-31'

// ① 指定期間に存在する部署を取得
const deptParams = new URLSearchParams({
    start_date: startDate,
    end_date: endDate
})

const deptResponse = await fetch(
    `http://localhost:8000/api/departments?${deptParams}`
)

if (!deptResponse.ok) {
    throw new Error(`HTTP ${deptResponse.status}`)
}

const departments = await deptResponse.json()

// ② APDボタンが押された場合
const params = new URLSearchParams({
    start_date: startDate,
    end_date: endDate,
    department_name: 'APD'
})

const response = await fetch(
    `http://localhost:8000/api/users?${params}`
)

if (!response.ok) {
    throw new Error(`HTTP ${response.status}`)
}

const users = await response.json()
```

Vueコンポーネントでは、取得した `departments` からボタンを描画し、選択された部署名を `department_name` に設定する。

期間が変更されたら、部署一覧も再取得する。

### 3-6. APIレスポンス例

GET /api/departments

```json
[
    {
        "id": 1,
        "name": "APD",
        "starton": "2024-01-01",
        "endon": "2025-12-31"
    },
    {
        "id": 2,
        "name": "開発部",
        "starton": "2025-01-01",
        "endon": null
    }
]
```

GET /api/users?start_date=2025-01-01&end_date=2025-12-31&department_name=APD

```json
[
    {
        "user_id": 1,
        "name": "田中太郎",
        "email": "tanaka@example.com",
        "department_name": "APD",
        "starton": "2024-04-01",
        "endon": "2025-06-30"
    }
]
```

### 3-7. 最終的な設計のポイント

| 処理 | 使用するテーブル | 判定条件 |
|---|---|---|
| 部署ボタン取得 | department | 部署の有効期間 |
| 所属ユーザー取得 | users + user_department | 所属期間 |
| 部署による絞り込み | user_department | department_name |
| 全部署のユーザー取得 | users + user_department | 所属期間のみ |

重要事項：

1. 部署が廃止されても、過去のdepartmentレコードを残す。
2. 過去の部署を表示する際は、現在の日付ではなく検索期間で有効性を判定する。
3. ユーザー検索はusersの在籍期間ではなく、user_departmentの所属期間を使用する。
4. 同じメールアドレスの別ユーザーを区別するため、JOINはuser_idを使用する。
5. 期間は検索条件に適用するだけで、INSERTやUPDATEは行わない。
6. 同じユーザーに複数の所属履歴がある場合は複数行返る。
7. 部署名の再利用や部署の再設置まで考慮する場合は、department_idによる管理を検討する。
8. 今回の制約だけでは、同一メールアドレスの在籍期間の重複や、ユーザー・部署・所属期間の整合性は完全には保証されない。
9. 部署ボタンの検索は部署の存在期間を判定するため、該当期間に所属ユーザーが0人の部署も表示される可能性がある。
