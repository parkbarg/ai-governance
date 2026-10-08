# OIDC認証とCookieによるセッション管理

## 1. Cookieはどのような場合に保持されるのか

Cookieには「セッションCookie」と「永続Cookie」の2種類がある。

### Cookieの種類

| 項目 | セッションCookie | 永続Cookie |
|---|---|---|
| `Max-Age` / `Expires` | 指定しない | 指定する |
| ページの再読み込み | 保持 | 保持 |
| 別タブを開く | 保持 | 保持 |
| タブを閉じる | 通常保持 | 保持 |
| ブラウザを閉じる | 原則削除 | 保持 |
| ブラウザを再起動する | 復元される場合がある | 保持 |
| 有効期限切れ | ブラウザのセッション終了時 | 指定した期限 |
| 手動でCookieを削除 | 削除 | 削除 |

※ シークレットモードでは、通常、そのプライベートセッションが終了するとCookieも削除される。

※ ブラウザの設定やセッション復元機能によって動作が異なる場合がある。

### 永続Cookieの例

```http
Set-Cookie: session_id=abc123; Max-Age=604800; HttpOnly; Secure; SameSite=Lax; Path=/
```

`Max-Age=604800`は7日間を意味する。

この場合、ブラウザを閉じたりPCを再起動したりしても、Cookieが削除されていなければ7日間保持される。

### Cookieが残っていてもログインできない場合

Cookieの有効期限とサーバー側のセッション有効期限は別である。

例えば、以下の設定を考える。

- Cookieの有効期限：7日間
- サーバー側のセッション有効期限：8時間

この場合、8時間経過後もブラウザにCookieが残っている可能性はあるが、FastAPI側でセッションが無効になるためログインできない。

**ログイン状態の維持には、Cookieが存在することとサーバー側のセッションが有効であることの両方が必要である。**

---

## 2. セッション管理のセキュリティ

### Cookieのセキュリティ属性

| 属性 | 役割 |
|---|---|
| `HttpOnly` | JavaScriptからCookieを読み取れなくする |
| `Secure` | HTTPS通信でのみCookieを送信する |
| `SameSite=Strict` | クロスサイトリクエストでCookieを送信しない |
| `SameSite=Lax` | 一部のクロスサイトリクエストでCookie送信を制限する |
| `SameSite=None` | クロスサイトでも送信可能（Secure必須） |
| `Max-Age` | Cookieの有効期間を指定する |

一般的なWebアプリでは、以下の設定を基本として検討する。

```python
response.set_cookie(
    key="session_id",
    value=session_id,
    max_age=604800,
    httponly=True,
    secure=True,
    samesite="lax",
    path="/",
)
```

### 主なセキュリティリスクと対策

| リスク | 対策 |
|---|---|
| XSSによるセッションIDの窃取 | HttpOnly |
| 通信経路でのCookie漏洩 | Secure + HTTPS |
| CSRF攻撃 | SameSite + CSRFトークンやOrigin検証 |
| セッションIDの推測 | 暗号学的に安全な乱数生成 |
| セッションの長期悪用 | 絶対有効期限・アイドルタイムアウト |
| ログアウト後のセッション再利用 | サーバー側のセッション無効化 |
| セッション固定攻撃 | 認証成功時に新しいセッションIDを発行 |

※ HttpOnlyだけでXSS攻撃を防げるわけではない。

### セッションの有効期限

セッションには2種類の期限を設定できる。

**① アイドルタイムアウト**

最後のアクセスから一定時間経過すると失効する。

例：30分間アクセスしなければログアウト。

**② 絶対有効期限**

アクセスの有無に関係なく、ログインから一定時間で失効する。

例：ログインから8時間経過するとログアウト。

### 有効期限の設計目安

| アプリケーション | アイドルタイムアウト | 絶対有効期限 |
|---|---|---|
| 高機密システム | 5〜15分 | 1〜4時間 |
| 社内業務Webアプリ | 15〜30分 | 8〜12時間 |
| 一般的なWebサービス | 30分〜数時間 | 1〜7日 |
| ログイン維持重視のサービス | 用途に応じて設定 | 7〜30日 |

※ 上記は設計上の目安であり、統一された標準値ではない。実際には情報の機密性やセキュリティ要件に合わせて設定する。

---

## 3. Vue.js + FastAPI + OIDCでの実装イメージ

### 全体の処理フロー

```text
① 初回ログイン

[Vue.js]
    |
    | ログイン開始
    v
[OIDC認証サーバー（IdP）]
    |
    | ユーザー認証
    | 認可コード発行
    v
[FastAPI]
    |
    | 認可コードをトークンに交換
    | IDトークン等を検証
    | セッションIDを生成
    v
[PostgreSQL / Redis]
    |
    | セッション情報を保存
    v
[FastAPI]
    |
    | Set-Cookieヘッダーで
    | セッションIDを返す
    v
[ブラウザ]
    |
    | Cookieを保存
    v
ログイン完了


② 2回目以降のアクセス

[Vue.js / ブラウザ]
    |
    | GET /api/users/me
    | Cookie: session_id=abc123
    v
[FastAPI]
    |
    | セッションIDを取得
    | セッション情報を検索
    | 有効期限を検証
    v
[PostgreSQL / Redis]
    |
    | 有効なセッション
    v
[FastAPI]
    |
    | ユーザー情報を返す
    v
[Vue.js]
```

### ① FastAPIでセッションIDを生成する

OIDC認証の成功後、セッションIDを生成する。

```python
import secrets

session_id = secrets.token_urlsafe(32)
```

セッション情報をPostgreSQLやRedisに保存する。

```text
sessions

session_id_hash
user_id
created_at
last_accessed_at
expires_at
revoked_at
```

セッションIDそのものではなく、ハッシュ化した値を保存する構成が考えられる。

### ② FastAPIからブラウザにCookieを設定する

```python
response.set_cookie(
    key="session_id",
    value=session_id,
    max_age=60 * 60 * 24 * 7,
    httponly=True,
    secure=True,
    samesite="lax",
    path="/",
)
```

HTTPレスポンスでは、JSONボディではなく`Set-Cookie`ヘッダーに設定される。

```http
HTTP/1.1 200 OK
Content-Type: application/json
Set-Cookie: session_id=abc123; Max-Age=604800; HttpOnly; Secure; SameSite=Lax; Path=/

{"message": "ログイン成功"}
```

### ③ ブラウザからAPIを呼び出す

Vue.js側：

```javascript
const response = await fetch(
  "/api/users/me",
  {
    credentials: "include"
  }
);
```

ブラウザがCookieを自動送信する。

```http
GET /api/users/me HTTP/1.1
Host: example.com
Cookie: session_id=abc123
```

ブラウザが送信するのはCookieの名前と値であり、`HttpOnly`や`SameSite`などの属性は送信しない。

なお、別オリジンのAPIを使用する場合は、CORS設定やCookieの送信条件にも注意する。

### ④ FastAPIでセッションを検証する

以下は処理のイメージ。

```python
from fastapi import Cookie, HTTPException

@app.get("/api/users/me")
def get_me(
    session_id: str | None = Cookie(default=None)
):
    if session_id is None:
        raise HTTPException(status_code=401)

    # セッションIDをハッシュ化
    # DBからセッション情報を取得
    # 有効期限を検証
    # 無効化されていないか確認

    # 検証成功後、ユーザー情報を返す
    return {"message": "認証成功"}
```

実際には、検証処理をFastAPIのDependencyとして共通化すると、複数のAPIで再利用できる。

### ⑤ ログアウト処理

ログアウト時には次の2つを実行する。

1. PostgreSQL / Redisに保存されたセッションを無効化する。
2. ブラウザのCookieを削除する。

Cookieを削除する例：

```python
response.delete_cookie(
    key="session_id",
    path="/",
)
```

Cookie削除だけでは不十分であり、サーバー側での失効処理が必要となる。

---

## まとめ

- **Cookieの保持**：ブラウザを閉じてもログインを維持したい場合は、有効期限付きの永続Cookieを使用する。
- **セキュリティ**：HttpOnly・Secure・SameSiteを設定し、サーバー側でセッションの有効期限と失効状態を管理する。
- **実装**：OIDCで認証した後、FastAPIがセッションIDを生成し、HTTPレスポンスのSet-Cookieヘッダーでブラウザに保存させる。以降のAPIリクエストではCookieを自動送信し、FastAPIがセッションを検証する。
