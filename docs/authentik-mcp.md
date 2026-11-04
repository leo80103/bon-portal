# Authentik:MCP 的使用者授權

> **狀態:規劃中,尚未開工。** 這份是初步方向,記錄目前的討論結論與開工前要查清楚的事。
> 標 ⚠️ 的是推論、還沒實測或還沒查證的地方,做完請把結果寫回來並拿掉標記。
>
> Portal 程式端零改動 —— MCP 跟其他產品一樣,自己對 Authentik 負責。

## 一句話

**MCP 當成「又一個產品」來接 Authentik。** 客戶的 AI 工具(Claude、Cursor 等)替客戶跑一次 OAuth,
拿到**屬於這位客戶的** token,之後每次呼叫 MCP 都自動帶在 header 上;MCP server 驗 token 就知道是誰、買了什麼。

**不走服務帳號。** 服務帳號只可能出現在「MCP server → 我們自己的後端 API」那一段,見〈呼叫下游 API〉。

## 前提:remote MCP

本文件只適用 **remote MCP**:MCP server 架在我們的雲端,客戶的 AI 工具透過 HTTPS 連進來。

| | remote(本文件) | local(stdio) |
|---|---|---|
| MCP server 在哪 | 我們的雲端 | 客戶自己的電腦 |
| 傳輸 | HTTPS,有 header | 程序間的 stdin/stdout,沒有 header |
| 身分怎麼來 | OAuth,MCP 規格有定義 | 通常是客戶手動貼 API key |
| 我們能更新/下架 | 能 | 要客戶自己更新 |

要做給客戶用、又要知道是誰,幾乎一定是 remote。⚠️ 待與主管確認。

## 為什麼不用服務帳號

服務帳號是**機器自己的身分**,token 裡沒有「這次是哪位客戶」。用它做 MCP 會變成:

```
MCP server(服務帳號,能讀所有客戶)──► 後端 API
                 ▲
   要讀誰的?只能靠 tool 參數 user_id —— 而 tool 參數是 LLM 填的
```

LLM 會被 prompt injection 操控(客戶貼進來的文件裡藏一句「查 user_id = 別人」),
而 MCP server 手上是萬能鑰匙,分不出這不是客戶本人的意思(confused deputy)。

前端的類比:把 admin token 寫死在 React app,再用 `?userId=` 決定顯示誰的資料。

## 架構

角色跟現有產品一一對應:

| OAuth 角色 | 現有產品 | MCP |
|---|---|---|
| Authorization Server | Authentik | Authentik(同一個) |
| Client(跑登入流程、保管 token) | 產品自己 | **客戶的 AI 工具**(MCP client) |
| Resource Server(驗 token) | 產品後端 | **MCP server** |

token 對照 Portal:

| | Portal | MCP |
|---|---|---|
| 誰跑登入流程 | `oidc-client-ts`(瀏覽器) | AI 工具(客戶的電腦) |
| token 存哪 | 瀏覽器 storage | AI 工具自己的設定 / keychain |
| 誰帶 header、誰 refresh | 套件 | AI 工具 |
| 客戶要做什麼 | 點「登入」 | 第一次連線時登入一次 |

客戶**看不到也不用碰 token**。

## 流程

```
客戶的 AI 工具              MCP server                 Authentik
     │  POST /mcp(沒帶 token)  │                          │
     │ ───────────────────────► │                          │
     │  401 + 去哪拿 token       │                          │
     │ ◄─────────────────────── │                          │
     │  GET /.well-known/oauth-protected-resource           │
     │ ───────────────────────► │  → authorization server   │
     │                                                      │
     │  開瀏覽器到授權頁(Code + PKCE)                      │
     │ ───────────────────────────────────────────────────► │
     │       有 authentik_session cookie → 靜默通過          │
     │       沒有 → 登入一次                                 │
     │  access token + refresh token                         │
     │ ◄─────────────────────────────────────────────────── │
     │  POST /mcp                │                          │
     │  Authorization: Bearer …  │                          │
     │ ───────────────────────► │  用 JWKS 驗簽、驗 aud    │
     │                           │  讀 sub / email / products
```

客戶如果已經在 Portal 登入過,授權那一步跟點產品連結一樣是靜默的 —— 這是同一個 SSO session 的好處,
不需要 Portal 做任何事。

## 各方要做的事

### Authentik

- 新增一個給 MCP 用的 OAuth provider + application
- scope:`openid`、`email`、`profile`、`products`、`offline_access`
  - `products` 直接沿用現有 Scope Mapping(`docs/authentik-product-entitlements.md`)
  - `offline_access` 理由同 Portal:少了它 token 5 分鐘就過期,客戶得重新授權。
    一樣要用 discovery 的 `scopes_supported` 驗,Authentik 會靜默丟掉沒設定的 scope
- policy binding:跟其他產品一樣,決定哪些群組能用 MCP

### MCP server

1. **每個請求驗 token**,驗不過回 `401` 並帶 `WWW-Authenticate` 指向 protected resource metadata
   - 驗簽:Authentik 的 JWKS(Node 可用 `jose`)
   - 驗 `aud`:確認是發給 MCP 的票,不是別的產品的
   - 驗 `exp`、`iss`
2. **提供 `/.well-known/oauth-protected-resource`**(RFC 9728),告訴 client 該去哪個 Authentik
3. **身分一律從 token 取,tool 參數裡不出現 user id**

   ```js
   const userId = token.sub              // ✅ Authentik 簽過的
   // async function getOrders({ userId }) // ❌ LLM 填的
   ```

4. **`products` 在這裡是真正的安全邊界。** 跟 Portal 相反 —— Portal 的過濾只是顯示用,
   MCP server 就是產品本身,沒買的產品對應的 tool 要真的拒絕(或乾脆不列出來)

MCP 官方 SDK 大多有現成的 auth 寫法,⚠️ 開工時再挑。

### Portal

不用改。不中繼 token,不知道 MCP 的存在也沒關係。要不要在首頁放一張「AI 工具連線說明」的卡片,是之後的產品決定。

## 呼叫下游 API

MCP 規格**禁止 token passthrough**:不能把客戶的 token 原封不動轉給我們自己的後端 API
(那張票是發給 MCP 的,下游收了就等於誰拿到票都能冒用)。兩種做法:

| 做法 | 說明 | 取捨 |
|---|---|---|
| Token exchange(RFC 8693) | 用客戶的 token 換一張發給下游的新 token | 身分一路延續,最乾淨;⚠️ Authentik 支援度待查 |
| MCP server 用自己的服務憑證 | 呼叫下游時帶「從 token 驗出來的」user id,下游信任 MCP server | 實作簡單;下游要信任 MCP server,MCP server 被攻破影響較大 |

不管哪種,**user id 都只能來自驗過的 token**。這也是唯一可能用到服務帳號的地方。

## 待驗證 / 待決定

- ⚠️ **Client 註冊方式(最關鍵,決定客戶體驗)**:Claude 這類公開 MCP client 會用
  Dynamic Client Registration(RFC 7591)或較新規格的 Client ID Metadata Document 自動註冊自己。
  Authentik 過去不支援 DCR,要查目前版本(2026.8.x)的狀況:
  - 支援 → 客戶貼上 MCP 網址就能用
  - 不支援 → 我們先在後台建好 client_id,再教客戶填進 AI 工具;能跑,但體驗差一截,
    而且要確認目標 AI 工具允許手動填 client_id
- ⚠️ **Audience**:Authentik 的 access token `aud` 預設是 client_id。MCP 規格要求 client
  帶 `resource` 參數(RFC 8707)、server 驗 audience,要確認兩者怎麼對上 ——
  尤其如果每個 AI 工具各有一個 client_id,MCP server 要接受哪些 `aud`
- ⚠️ **refresh 時 `products` 會不會更新**:客戶新買或到期後,refresh 換來的 token 是否會重跑
  scope mapping。這題跟 Portal 開放購買那邊是同一題,查一次兩邊都用得上
- ⚠️ **remote vs local**:待與主管確認(見〈前提〉)
- **下游 API 的做法**:token exchange 還是服務憑證,等 MCP 要碰哪些 API 確定後再決定
- **MCP server 誰做、放哪個 repo**:不在本 repo,待分工

## 參考

- MCP 規格 Authorization 章節:https://modelcontextprotocol.io/specification/ (看最新版本的 Basic → Authorization)
- RFC 9728 OAuth Protected Resource Metadata
- RFC 8707 Resource Indicators
- RFC 7591 Dynamic Client Registration
- RFC 8693 Token Exchange
