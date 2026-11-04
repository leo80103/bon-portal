# Authentik:email 自行註冊(申請中帳號做法)

> **狀態:必做項目(1~3)已完成並上線。** 剩下的 4、5 是選用,目前不做。
> 標 ⚠️ 的是推論、還沒實測的地方,做完請把結果寫回來並拿掉標記。
>
> Portal 程式端零改動 ——「註冊」連結長在 Authentik 的登入頁上。

## 為什麼先建申請中帳號

完整的取捨見 artifact〈會員註冊流程〉:https://claude.ai/artifact/NvhuZUnLJAMPkwi2Je8Dfo

一句話:**填 email 時先建一個「停用、沒有密碼」的帳號(申請中帳號),點了確認信的人才能設密碼。**

**申請中帳號** = 停用 + 沒有密碼 + 路徑 `users/signup-pending`,三者同時成立。
它跟「被管理員停用的帳號」在 Authentik 裡都是停用狀態,**靠路徑區分** —— 下面的規則全都建立在這個區分上。
(artifact 裡為了讓非工程人員好懂,稱它為「空白卡」。)

- 理想做法是「點信之前什麼都不建」,但 Authentik 的 Email 階段寄信時需要一個已存在的
  使用者(信裡的 token 綁在使用者上)。⚠️ 尚未用實驗確認,見「待驗證」
- 官方範例是「填 email 和密碼 → 建停用帳號 → 點信開通」。密碼在確認前就寫進去,
  會有「重新申請後密碼錯誤」與「別人先用你的 email 和他的密碼佔位」的問題
- 申請中帳號沒有密碼,密碼只能由收得到信的人設定,上面兩個問題都不存在

## 現況(已完成)

**流程 `primary-signup`**(使用目的:註冊;身分認證:Require no authentication)

| 順序 | 階段 | 類型 | 重點設定 |
| --- | --- | --- | --- |
| 10 | `primary-signup-email-prompt` | Prompt | 欄位 `primary-signup-field-email`(鍵 `email`、類型電子郵件、必需);驗證政策 `primary-signup-username-from-email` |
| 20 | `primary-signup-pending-write` | User Write | 需要時建立;**建立停用狀態**;外部;路徑 `users/signup-pending`;群組空白 |
| 30 | `primary-signup-email` | Email | 成功時啟用使用者;全域 SMTP;token `minutes=30`;主旨「確認您的 email」;範本 Account Confirmation |
| 40 | `primary-signup-password-prompt` | Prompt | `default-password-change-field-password` / `-repeat`;驗證政策 `default-password-change-password-policy` |
| 50 | `primary-signup-password-write` | User Write | **不建立使用者**;建立停用狀態**關閉** |
| 60 | `primary-signup-complete-prompt` | Prompt | 只有一段「註冊完成」說明文字(`primary-signup-field-complete-message`)和繼續按鈕;驗證政策留空 |
| 70 | `primary-signup-redirect-portal` | Redirect | 模式:靜態;目標 URL:Portal 首頁 |

**為什麼最後不自動登入,而是導回 Portal 首頁**:確認信常在**新分頁或另一台裝置**上點開。
原本最後一步是 User Login,流程結束會照 `next` 走回當初的 authorize 網址,把 `?code=&state=` 帶回 Portal;
但 `oidc-client-ts` 的 state 存在 sessionStorage(綁分頁),新分頁找不到就顯示
`No matching state found in storage`。Redirect 把 `next` 截斷、改導到不帶 code/state 的首頁才是解法;
拿掉自動登入則是讓行為一致 —— 一律用剛設定的密碼登入,也不會在別人的裝置上留下登入狀態。
(Redirect Stage 是 Authentik 2024.12 之後才有。)

**政策 `primary-signup-username-from-email`**:把 email 複製成帳號名稱(帳號名稱就是 email,
點信後只設密碼),並擋下重複 email(見下面 1-a)。

**政策 `primary-signup-reuse-pending`**:綁在順序 20 的附加上,沿用既有的申請中帳號(見下面 1-b)。

**政策 `primary-signup-finalize`**:綁在順序 60 的附加上,開通後移出申請中並標上 `signup_source: email`(見下面 2)。

**登入頁**:`primary-login-identification` → 註冊流程 = `primary-signup`。

**Google Source**(見下面 3):

| 欄位 | 值 | 原本 |
| --- | --- | --- |
| 認證流程 | `primary-source-authentication`(碰到申請中帳號就開通) | `default-source-authentication` |
| 註冊流程 | `primary-source-enrollment`(第一次就用 Google 的人標上 `signup_source: google`) | `default-source-enrollment` |
| 使用者路徑 | `users` | `goauthentik.io/sources/%(slug)s` |

> 正式環境已經開放註冊,1~3 也已上線。
>
> 5(擋機器人)暫不做,接受機器人可能大量申請、用光寄信額度的風險;觀察到異常再補(見 5 的「什麼時候該做」)。

### 路徑與 `signup_source`

**路徑 = 狀態,`signup_source` = 來源**,兩件事分開記:

| 路徑 | 意思 |
| --- | --- |
| `users/signup-pending` | 申請中帳號,只會是**停用**的 |
| `users` | Portal 的正式客戶,不管從哪來 |
| `users/external` | 同事的,不歸 Portal 管 |
| `goauthentik.io/...` | Authentik 自己的(outposts、service-accounts) |

| 帳號怎麼來的 | `signup_source` | 誰寫的 |
| --- | --- | --- |
| 業務在後台建的 | 無 | — |
| email 自行註冊 | `email` | `primary-signup-finalize`(2) |
| 申請中改用 Google | `google` | `primary-source-activate-pending`(3-a) |
| 第一次就用 Google | `google` | `primary-source-enrollment-mark`(3-b) |

不用資料夾分來源:一個帳號可能先 email 註冊、之後又連結 Google,資料夾只能放一個地方,而且沒人會回頭搬。
「現在能用什麼登入」看帳號的「已連結的來源」和有沒有密碼,不看 `signup_source`。

第一次就用 Google 的帳號,Authentik 自己也會寫 `goauthentik.io/user/sources: [Google]`(值是 Source 的**名稱**)。
統計時只看 `signup_source` 就好,不用合併兩個欄位。

## 待驗證

- [x] **正常註冊跑一次**:輸入 email → 使用者清單出現停用、路徑 `users/signup-pending` 的帳號
      → 點信 → 設密碼 → 註冊完成 → 導回 Portal 首頁 → 帳號變啟用 → 用 email + 密碼登入
- [x] **政策改 `prompt_data` 有沒有效**:建申請中帳號那步沒有出現 `username` 相關錯誤,
      驗證政策裡改的 `prompt_data` 有傳到後面的階段
- [ ] **理想做法是否真的不可行**:暫時刪掉順序 20 的**附加**(不是刪階段),跑一次,
      看是否出現「No pending user」。有 → 文件說法成立;沒有 → 理想做法其實可行,重新評估
- [ ] **Google 開通後,回頭點原本那封確認信**:可能讓他設一組密碼(等同忘記密碼),也可能顯示連結失效。
      **不能多出帳號**,記下實際行為

---

# 計畫

| 項目 | 狀態 |
| --- | --- |
| 1. 重複 email | ✅ 已完成並驗證 |
| 2. 開通後移出「申請中」 | ✅ 已完成並驗證 |
| 3. Google 登入(開通申請中帳號、第一次登入的路徑與標記) | ✅ 已完成並驗證 |
| 4. 清理申請中帳號 | 選用,目前不做 |
| 5. 擋機器人 | 選用,目前不做 |

**2 和 3 是一組**:3-a 靠路徑判斷「申請中」,開通後的帳號若還留在 `users/signup-pending`,
之後被管理員停用,3-a 會把它當成申請中帳號重新打開。2 確保開通的帳號一定離開那個路徑。

## 1. 重複 email(✅ 已完成)

**目標**(對照 artifact 故事二、三):

| 註冊時輸入的 email | 處理 |
| --- | --- |
| 沒人用過 | 照常建申請中帳號 |
| 已有**啟用中**的帳號 | 擋下,提示「這個 email 已經註冊過,請直接登入,或改用 Google 登入。」 |
| 已有**申請中帳號**(停用 + 路徑 `users/signup-pending`) | **沿用**那個帳號,重寄確認信 |
| 已有**被停用**的帳號(不是申請中帳號) | 擋下,提示「這個帳號目前無法使用,請與我們聯繫。」—— 不能讓註冊重新打開被停用的帳號 |

**做法**:拆成兩條政策,因為「擋下並顯示訊息」和「沿用申請中帳號」要在不同時間點做。

**1-a 擋下**:改寫 `primary-signup-username-from-email`(Prompt 的驗證政策,送出 email 時執行,
回傳 `False` 就停在原頁並顯示訊息):

```python
from authentik.core.models import User

prompt_data = request.context.get("prompt_data", {})
email = prompt_data.get("email", "").strip()

# 帳號名稱就是 email(原本這條政策的工作)
prompt_data["username"] = email

existing = User.objects.filter(email__iexact=email).first()
if existing is None:
    return True

if existing.is_active:
    ak_message("這個 email 已經註冊過,請直接登入,或改用 Google 登入。")
    return False

# 停用中:只有申請中帳號可以繼續,被管理員停用的帳號一律擋下
if existing.path != "users/signup-pending":
    ak_message("這個帳號目前無法使用,請與我們聯繫。")
    return False

return True
```

- 用 `email__iexact`:大小寫不同視為同一個 email(Google 回來的大小寫不一定跟使用者打的一樣)
- ⚠️ 帳號名稱也是 email,而業務開的舊帳號帳號名稱不一定是 email。若有人的**帳號名稱**剛好等於
  別人的 email,User Write 會撞名報錯 —— 實測時注意

**1-b 沿用申請中帳號**:新增政策 `primary-signup-reuse-pending`,綁在流程 `primary-signup` 的
**順序 20(`primary-signup-pending-write`)那一筆附加上**(編輯附加 → 政策),
「在執行階段時進行評估」保持開啟:

```python
from authentik.core.models import User

plan = request.context.get("flow_plan")
email = plan.context.get("prompt_data", {}).get("email", "").strip()

pending = User.objects.filter(
    email__iexact=email, is_active=False, path="users/signup-pending"
).first()
if pending:
    # 把既有的申請中帳號設成「待處理的使用者」。
    # User Write 是「需要時建立」,看到已經有待處理的使用者就只更新、不新建;
    # 後面的 Email 階段也就寄給這個帳號。
    plan.context["pending_user"] = pending

return True   # 一定回 True:回 False 會讓這個階段被跳過,申請中帳號就不會建出來
```

- `request.context["flow_plan"]` 在附加上的政策可用,且修改會留到後面的階段 —— 已實測
  (同一個 email 註冊兩次沒有撞名,代表順序 20 拿到的是沿用的帳號)
- **在執行階段時進行評估**必須開啟:只在規劃流程時評估的話,那時還沒有 email,這條政策等於沒作用

**驗證(已通過)**:

| 動作 | 預期結果 |
| --- | --- |
| 輸入全新的 email | 顯示「請去收信」;`users/signup-pending` 多一個帳號 |
| 不點信,用同一個 email 再註冊一次 | 沒有錯誤,收到第二封信;`users/signup-pending` **仍然只有一個**帳號 |
| 點兩封信中的**任一封** → 設密碼 | 註冊完成,帳號變成啟用 |
| 輸入已經在用的 email | 停在原頁,顯示「這個 email 已經註冊過…」 |
| 後台建一個停用的測試帳號(路徑 `users`),用它的 email 註冊 | 停在原頁,顯示「這個帳號目前無法使用…」 |

## 2. 開通後移出「申請中」並加上永久標記(✅ 已完成)

**問題**:路徑只在順序 20 **新建**帳號時從階段設定讀進來,之後沒有任何階段會再改它
(順序 30 點信只改 `is_active`,順序 50 只寫密碼)。帳號一直留在 `users/signup-pending` 的話:

- 之後被管理員停用,看起來就跟申請中帳號一樣 → 1-a 會讓它重新註冊、3-a 會讓 Google 重新打開它
- 「申請中帳號有幾個」這個觀察指標失準

**做法**:新增政策 `primary-signup-finalize`,綁在流程 `primary-signup` **順序 60(`primary-signup-complete-prompt`)
那一筆附加上**,「在執行階段時進行評估」開啟:

```python
plan = request.context.get("flow_plan")
if plan is None:
    return True

user = plan.context.get("pending_user")

# 只處理還留在申請中路徑的帳號,重跑也不會出事
if user and user.pk and user.path == "users/signup-pending":
    user.path = "users"
    # 永久標記:之後要統計「誰是怎麼註冊的」靠這個,不靠路徑
    user.attributes["signup_source"] = "email"
    user.save()

# 一定回 True:回 False 會讓順序 60 被跳過,使用者看不到「註冊完成」
return True
```

- 走到 60 代表已經點過信、設好密碼,這時移出才正確。40、50 還沒設密碼,使用者可能在那裡離開
- 政策裡直接 `user.save()`(有副作用的政策)可行 —— 已實測

**一次性搬移**:綁上去之前就開通的帳號(啟用中、還在 `users/signup-pending`)要手動搬:
路徑改成 `users`、屬性加 `signup_source: email`。條件一定要有「啟用中」,否則會把申請中帳號也搬走。
點了信卻沒設密碼就離開的人也在這群裡,他們已經證明過 email,一起搬沒有問題。

**驗證(已通過)**:註冊完成後,帳號路徑變成 `users`、屬性有 `signup_source: email`;
後台停用它、再用同一個 email 註冊 → 顯示「這個帳號目前無法使用…」。

## 3. Google 登入(✅ 已完成)

Google 登入會依「Authentik 裡有沒有同 email 的帳號」走不同流程,兩條都要處理:

| 情況 | 走哪條流程 | 處理 |
| --- | --- | --- |
| 已經有同 email 的帳號(含申請中帳號) | Source 的**認證流程** | 3-a |
| 完全沒有帳號 | Source 的**註冊流程** | 3-b |

兩條都**不要改 Authentik 內建的流程**(`default-source-authentication`、`default-source-enrollment`):
它們由啟用中的內建藍圖管理,改了可能被蓋回去。另建自己的流程,沿用內建流程的階段與政策。

### 3-a 碰到申請中帳號時直接開通

**問題**(artifact 故事四):Google Source 的比對模式是「連結到相同 email 的使用者」。
使用者申請到一半改按 Google,Authentik 會把 Google 連到停用的申請中帳號,然後因為帳號停用而登入失敗
(已實測:不會略過停用的帳號另建一個)。

Google 跟確認信一樣證明了 email 所有權,所以碰到**申請中帳號**時直接開通是安全的。
**只限申請中帳號**(路徑 `users/signup-pending`),被管理員停用的帳號不能因此重新打開。

**做法**:

1. 建流程 `primary-source-authentication`(使用目的:身分認證;認證要求照抄 `default-source-authentication`)
2. 附加 `default-source-authentication` 的 User Login 階段(`default-source-authentication-login`),
   流程本身的政策附加也照抄
3. 在那筆附加上綁政策 `primary-source-activate-pending`(在執行階段時進行評估):

```python
plan = request.context.get("flow_plan")
if plan is None:
    return True

user = plan.context.get("pending_user")

# 只開通申請中帳號:停用 + 路徑 users/signup-pending 同時成立
# 被管理員停用的帳號(路徑 users)不會進來,登入階段照樣擋下
if user and user.pk and not user.is_active and user.path == "users/signup-pending":
    user.is_active = True
    user.path = "users"
    # 永久標記:之後要統計「誰是怎麼註冊的」靠這個,不靠路徑
    user.attributes["signup_source"] = "google"
    user.save()

# 一定回 True:回 False 會讓登入階段被跳過
return True
```

4. **使用者目錄 → 聯合與社群登入 → Google → 編輯 → 認證流程**:改成 `primary-source-authentication`。
   出問題時改回 `default-source-authentication` 就能還原

- 用 Google 開通的申請中帳號沒有密碼;之後想用密碼登入就走忘記密碼

**驗證(已通過)**:新 email 先填註冊(不點信)→ 改按 Google 登入 → 成功登入,使用者清單仍只有一筆,
已啟用、路徑 `users`、`signup_source: google`;已連結 Google 的舊帳號照常登入;
另拿一個**被停用的一般帳號**按 Google → 仍然登不進去。

### 3-b 第一次就用 Google:路徑與標記

**問題**:完全沒有帳號的人走 Source 的註冊流程,路徑由 **Source 自己的「使用者路徑」**決定,
預設是 `goauthentik.io/sources/%(slug)s`,也就是 `goauthentik.io/sources/google`,不在 `users`。
而且 3-a 的政策碰不到這條路,所以也不會有 `signup_source`。

安全上沒有破洞(1-a、3-a 都只認 `users/signup-pending`),但客戶帳號會混進 Authentik 自己的 `goauthentik.io/`。

**只改路徑不夠**:改成 `users` 之後,「第一次就用 Google」和「業務建的、之後按過 Google」的帳號就長得一樣
(都在 `users`、都沒有 `signup_source`、都連結了 Google)。所以路徑和標記要一起做:

1. **Google Source → 使用者路徑**:改成 `users`(只影響之後新建的帳號,舊帳號手動改)
2. 新增政策 `primary-source-enrollment-mark`:

```python
plan = request.context.get("flow_plan")
if plan is None:
    return True

user = plan.context.get("pending_user")

# 綁在登入階段前面:這時 User Write 已經把帳號建好了,user.pk 才有值
if user and user.pk:
    # 這條流程只給 Google Source 用,所以直接寫 google
    user.attributes.setdefault("signup_source", "google")
    user.save()

# 一定回 True:回 False 會讓登入階段被跳過
return True
```

3. 建流程 `primary-source-enrollment`(使用目的:註冊;認證要求照抄 `default-source-enrollment`),
   照順序附加 `default-source-enrollment` 的階段(prompt → User Write → User Login),
   **流程本身和 prompt 那筆附加上的政策也照抄** —— 漏掉 prompt 上的政策,每個新用戶都會被問一次帳號名稱
4. 把第 2 步的政策綁在 **User Login 那筆附加**上(在執行階段時進行評估)。綁在 User Write 前面的話帳號還沒建,標不到
5. **Google Source → 註冊流程**:改成 `primary-source-enrollment`。出問題時改回 `default-source-enrollment`

- 值寫死 `google`。之後加其他 Source,要給它自己的註冊流程和政策,不要共用
- 不用 Source 的 Property Mapping 寫 `signup_source`:不確定它只在建帳號時套用,還是每次登入都套用;
  後者會讓業務建的帳號按一次 Google 就被標成 `google`。註冊流程只有建新帳號時才走,行為是確定的
- 改之前就用 Google 建的帳號不會自動補上,手動加 `signup_source: google`

**驗證(已通過)**:用一個還沒在 Authentik 裡的 Google 帳號登入 → 不會被問帳號名稱、直接登入;
新帳號路徑 `users`、`signup_source: google`;再登入一次照常登入,帳號仍只有一個。

## 4. 清理申請中帳號(選用)

申請中帳號沒有密碼、不會被冒用,1-b 和 3-a 也確保它不會擋住本人,所以**不急著刪**。
數量突然暴增可以當作機器人的警訊。保留 7~30 天後清除。

**查看數量**:**使用者目錄 → 使用者** → 左側 `users/signup-pending`。
申請的時間與來源 IP 也在 **事件** 裡,刪掉申請中帳號後仍查得到。

**定期清除**:Authentik 沒有內建「停用帳號 N 天後刪除」,在主機上排程:

```bash
# 刪除建立超過 14 天、仍未開通的申請中帳號
docker compose exec -T worker ak shell -c "
from datetime import timedelta
from django.utils.timezone import now
from authentik.core.models import User
qs = User.objects.filter(path='users/signup-pending', is_active=False,
                         date_joined__lt=now() - timedelta(days=14))
print(qs.count(), 'deleted')
qs.delete()
"
```

- ⚠️ `ak shell -c` 是否支援、`date_joined` 是否為建立時間,執行前先把 `qs.delete()` 拿掉試跑一次
- 執行位置:跑 Authentik 的主機、compose 檔所在資料夾;用 cron 每天一次

## 5. 擋機器人(選用)

每一次申請都會寄一封信。Workspace 每天約 2,000 封上限,機器人一次填幾千次就會用光,
連忘記密碼的信都寄不出去;大量退信也會傷害網域信譽。**防線要在寄信之前。**

**什麼時候該做**:`users/signup-pending` 的數量異常增加、事件裡出現大量同來源 IP 的申請,
或收到寄信額度/退信的警告。在那之前先不做。

**5-a Captcha 階段**(主要防線)

1. 申請 captcha 金鑰。建議 **Cloudflare Turnstile**(免費、多數情況不用使用者點圖片)
2. **流程與階段 → 階段 → 建立 → Captcha Stage**:名稱 `primary-signup-captcha`,填 site key / secret key
   ⚠️ Turnstile 需要在 Captcha 階段裡填對應的 JS URL 與驗證 API URL,建立時對照 Authentik 文件
3. 附加到 `primary-signup`,**順序 15**(填完 email 之後、建申請中帳號之前)

**5-b 限制頻率**(輔助)

- Email 階段本身有 `Account Recovery Max Attempts` / `Cache Timeout`(預設 5 分鐘 5 次),
  能擋「對同一個 email 一直重寄」,擋不住「幾千個不同 email」
- ⚠️ Authentik 的名譽分數主要記錄登入失敗,不太適合限制註冊頻率。如果 Authentik 前面是
  Cloudflare,用 Cloudflare 的 Rate Limiting 規則限制 `/if/flow/primary-signup/` 與
  `/api/v3/flows/executor/primary-signup/` 的請求頻率更直接

**驗證**:無痕視窗註冊時出現 captcha;captcha 沒過就不會寄信、也不會建申請中帳號。

---

## 相關

- artifact〈會員註冊流程〉—— 申請中帳號做法的完整理由(給非工程人員,文中稱「空白卡」)
- `docs/authentik-google-login.md` —— 一個 email 一個帳號、Google 比對模式
- `docs/authentik-email.md` —— SMTP 與寄信額度
- `docs/authentik-password-recovery.md` —— 同一個登入頁上的忘記密碼
