# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- `npm run dev` — Vite dev server,固定 port 6030(`strictPort`,被占用時直接失敗)
- `npm run build` — `tsc -b` 型別檢查後 `vite build`,產物在 `dist/`
- `npm run preview` — 預覽 build 產物
- `npm run lint` — oxlint

沒有測試框架(計畫中會用 Vitest,見 `openspec/changes/` 的規劃)。

## What this app is

提供給**客戶**的服務登入入口(launcher)—— 前端 only,身分由自架的 [Authentik](https://goauthentik.io/) 提供,這個 repo 裡沒有任何後端。

**不是**內部員工工具,也**不是**行銷網站(行銷另有站點)。

帳號有三種來源:業務簽約後在 Authentik 預先建立、客戶第一次用 Google 登入時自動建立、客戶自行用 email + 密碼註冊(點過確認信才開通帳號)。**一個 email 只對應一個帳號** —— 不管用密碼還是 Google 登入,都要進到同一個 user。規則與理由見 `docs/authentik-google-login.md`。忘記密碼已設定(`docs/authentik-password-recovery.md`,寄信見 `docs/authentik-email.md`);email 自行註冊也已設定(`docs/authentik-self-signup.md`),填 email 時先建停用、沒有密碼的申請中帳號,點了確認信才能設密碼。

不管帳號從哪裡來,**授權只來自 Authentik 的群組**,自動建立或自行註冊的帳號一開始沒有任何產品。線上購買後怎麼把群組給上去尚未定案(等後端分工),目前仍由我們在後台設定。

流程只有兩件事:

1. 客戶在 Portal 登入 → 建立起 Authentik 的 SSO session(`authentik_session` cookie)
2. 客戶點首頁的產品連結 → 該產品**自己**跑一次 OIDC,靠那個 cookie 靜默完成登入

**Portal 全程不碰產品的 token。** 這是刻意的責任邊界:Portal 不是憑證的中繼站,各產品自己對 Authentik 負責。舊版曾經把 `id_token` 放在網址 fragment 交給下游服務(`#token=`),那套已經整個移除,連帶移除了為它存在的 open-redirect 網域白名單。

### 登入

`react-oidc-context` + `oidc-client-ts` 處理 Authorization Code + PKCE,包含 `nonce` 與 refresh。設定在 `src/config/oidc.ts`:

- **`authority` 一個值就夠** —— endpoint 由 `<authority>/.well-known/openid-configuration` 取得
- **`redirect_uri` 指向首頁 `/`**,沒有獨立的 callback 路由。`oidc-client-ts` 偵測到網址帶 `code`/`state` 就自動完成交換,`onSigninCallback` 負責把它們從網址列抹掉
- **`offline_access` 不是裝飾**:少了它拿不到 refresh token,而 `react-oidc-context` 的 `isAuthenticated` 是 `user && !user.expired` —— access token 一過期(預設 5 分鐘)登出按鈕就會消失,使用者做不了 SLO。Authentik 會靜默把未設定的 scope 交集掉不報錯,所以要用 discovery 的 `scopes_supported` 驗,不能靠「本機看起來正常」

### 登出

`auth.signoutRedirect()`。它內部依序是「取 `id_token` 當 `id_token_hint` → `removeUser()` → 導向 end-session」,順序正是需要的 —— 特別是 `removeUser()` 在導向之前:登出會帶 `post_logout_redirect_uri` 回到 Portal,本機 user 還留著的話,使用者回來會看到「已登入」,但 Authentik 的 session 其實已經結束了。

`id_token_hint` 不能省 —— Authentik 只有搭配它才接受 `post_logout_redirect_uri`。而 `post_logout_redirect_uri` 與 `redirect_uri` 在 Authentik 都是**字串完全比對**,連結尾斜線都算不同。

> **歷史(留著當診斷線索)**:Authentik **2026.8.0** 有一個登出白畫面的 bug —— session 裡殘留未完成的 flow plan 時,`EndSessionView.dispatch()` 會回一個 body 全空的 HTTP 200,而且**根本沒登出**,症狀是「第一次登出正常、第二次以後白畫面」。當時的解法是先繞去 `/flows/-/cancel/` 清掉殘留再進 end-session。
>
> **在 2026.8.1 實測已不再重現,所以繞道移除了。** 如果哪天登出又出現空白的 200,先查 Authentik 版本 —— 那是同一個 bug 回來了,繞道的實作見 git 歷史(`src/utils/authentikLogout.ts`)。

### 路由

**沒有 router。** Portal 只有一個畫面,依 `useAuth()` 切換:`isLoading` → 載入中、未登入 → 登入按鈕、已登入 → 產品清單。

任何其他路徑由 nginx 的 `try_files` 回 `index.html`,再由 `src/main.tsx` 開頭兩行把網址列收回 `/`。

### 產品清單

首頁列的是**產品目錄**與**該客戶的授權**的交集。這兩份資料刻意放在不同地方:

```
對所有客戶都一樣的  →  src/products.ts(程式碼,進版控,有型別有註解)
每個客戶不一樣的    →  Authentik 的群組(目前由我們在後台設定)
```

所以「有哪些產品、叫什麼、網址是什麼」是程式碼;「這位客戶能看到哪些」是 Authentik 的事。Portal 不維護任何「誰買了什麼」的副本。

授權透過自訂的 `products` scope 回來,而且**是已經歸納到產品層級的**:Authentik 端的 Scope Mapping 把 `bonsale:admin` 這類細項權限 `split(":")[0]` 收斂成 `bonsale`。Portal 因此只做單純的交集,不需要 prefix 比對,token 也不會被幾十條細項權限撐大(撐大會連帶讓登出的 `id_token_hint` URL 過長)。設定步驟見 `docs/authentik-product-entitlements.md`。

`readEntitlements()`(`src/products.ts`)把 claim 判成三態,**三者的畫面必須不同**:

| claim | 意思 | 文案 |
|---|---|---|
| 不存在 / 不是陣列 | **我們的設定錯了** | 無法取得您的服務授權,請與我們聯繫 |
| 對不上任何產品(含空陣列) | 客戶確實沒有可用服務(例如合約到期、自行註冊但還沒開通) | 目前沒有可使用的服務 |
| 有對上 | 正常 | 產品連結 |

把前兩者混在一起,等於把自己的設定錯誤講成客戶的合約狀態。

**這個過濾不是安全邊界,也不該做成安全邊界。** 客戶改 devtools 就能讓所有連結出現 —— 不要緊,真正的關卡在各產品那邊(產品自己跑 OIDC,Authentik 依 policy binding 決定放不放行)。在 Portal 加一層驗證會讓它變回刻意不當的角色,而且擋不住任何真實攻擊:攻擊者本來就能直接開產品網址。

### 設定

**四個 `VITE_*` 變數,沒有一個是機密**(client_id 會明文出現在授權請求的網址上;其餘都是網址)。Portal 是 public client,沒有 client secret。

設定在 **build 時**就被 Vite 換成字面值寫進產物 —— 一個 image 對應一個環境,換設定要重新 build。這是刻意的:Portal 只有一個部署,而重 build 不到一秒、push 的只有變動的那層;何況在 Cloud Run 上改環境變數本來就會產生一個新 revision,也是一次部署,所以執行時注入並沒有省掉那一步。

- **本機開發** 讀 `.env`
- **build image** 由 `Dockerfile` 的 `ARG` 提供,`.env` 被 `.dockerignore` 擋在外面 —— 否則忘記改就會把 `localhost:9000` 安靜地燒進正式版的 image
- **必要參數缺漏時 build 會直接失敗**,不會產出一個「跑得起來但登入必壞」的 image

**規則:一律用具名存取 `import.meta.env.VITE_XXX`,不要用動態 key(`import.meta.env[key]`)。** 具名存取 Vite 只會替換掉那一個值;動態存取會讓它把**整個 env 物件**塞進 bundle,連你沒打算曝露的變數都會進去。所有變數都要在 `src/vite-env.d.ts` 宣告,那份宣告就是設定的單一清單。

稽核:

```bash
grep -rn "import.meta.env" src/          # 應該只有具名存取,沒有 [key]
grep -o 'VITE_[A-Z_]*' dist/assets/*.js  # build 後不該出現變數名(UI 文案除外)
```

## Conventions

- UI 文案與程式碼註解用繁體中文(zh-TW)。
- 註解寫「為什麼」,尤其是那些看起來可以簡化、但簡化了就會壞的地方(登出繞道、`offline_access`、`runtimeEnv` 的單一入口)。那些都是踩過坑才長出來的。
- 參考實作在 `~/Desktop/Program/Demo/authentik/code/authentik-react-demo-app` —— 這個 repo 以它為基底重建。動 OIDC / 登出 / runtime env 之前先去對照。
