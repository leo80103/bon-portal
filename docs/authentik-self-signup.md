# Authentik:email 自行註冊(空白卡做法)

> **狀態:進行中。** 基本流程已建好並掛上登入頁,本文件後半是接下來要補的計畫。
> 標 ⚠️ 的是推論、還沒實測的地方,做完請把結果寫回來並拿掉標記。
>
> Portal 程式端零改動 ——「註冊」連結長在 Authentik 的登入頁上。

## 為什麼是空白卡

完整的取捨見 artifact〈會員註冊流程〉:https://claude.ai/artifact/NvhuZUnLJAMPkwi2Je8Dfo

一句話:**填 email 時先建一個「停用、沒有密碼」的帳號(空白卡),點了確認信的人才能設密碼。**

- 理想做法是「點信之前什麼都不建」,但 Authentik 的 Email 階段寄信時需要一個已存在的
  使用者(信裡的 token 綁在使用者上)。⚠️ 尚未用實驗確認,見「待驗證」
- 官方範例是「填 email 和密碼 → 建停用帳號 → 點信開通」。密碼在確認前就寫進去,
  會有「重新申請後密碼錯誤」與「別人先用你的 email 和他的密碼佔位」的問題
- 空白卡上沒有密碼,密碼只能由收得到信的人設定,上面兩個問題都不存在

## 現況(已完成)

**流程 `primary-signup`**(使用目的:註冊;身分認證:Require no authentication)

| 順序 | 階段 | 類型 | 重點設定 |
| --- | --- | --- | --- |
| 10 | `primary-signup-email-prompt` | Prompt | 欄位 `primary-signup-field-email`(鍵 `email`、類型電子郵件、必需);驗證政策 `primary-signup-username-from-email` |
| 20 | `primary-signup-pending-write` | User Write | 需要時建立;**建立停用狀態**;外部;路徑 `users/signup-pending`;群組空白 |
| 30 | `primary-signup-email` | Email | 成功時啟用使用者;全域 SMTP;token `minutes=30`;主旨「確認您的 email」;範本 Account Confirmation |
| 40 | `primary-signup-password-prompt` | Prompt | `default-password-change-field-password` / `-repeat`;驗證政策 `default-password-change-password-policy` |
| 50 | `primary-signup-password-write` | User Write | **不建立使用者**;建立停用狀態**關閉** |
| 60 | `primary-signup-login` | User Login | 預設值 |

**政策 `primary-signup-username-from-email`**:把 email 複製成帳號名稱(帳號名稱就是 email,
點信後只設密碼)。

**登入頁**:`primary-login-identification` → 註冊流程 = `primary-signup`。

> ⚠️ **目前正式環境已經開放註冊,但下面的 1(重複 email)還沒做。**
> 在那之前,既有的 email 可能被註冊出第二個帳號。如果短期內做不完,先把上面的「註冊流程」清空。
>
> 5(擋機器人)暫不做,接受機器人可能大量申請、用光寄信額度的風險;觀察到異常再補(見 5 的「什麼時候該做」)。

## 待驗證

- [ ] **正常註冊跑一次**:輸入 email → 使用者清單出現停用、路徑 `users/signup-pending` 的帳號
      → **用手機**點信 → 設密碼 → 自動登入 → 帳號變啟用 → 登出再用 email + 密碼登入
- [ ] **政策改 `prompt_data` 有沒有效**:放空白卡那步若出現 `username` 相關錯誤,就是
      `primary-signup-username-from-email` 在驗證政策裡改 `prompt_data` 沒傳到後面的階段
- [ ] **理想做法是否真的不可行**:暫時刪掉順序 20 的**附加**(不是刪階段),跑一次,
      看是否出現「No pending user」。有 → 文件說法成立;沒有 → 理想做法其實可行,重新評估

---

# 接下來的計畫

**必做:1、2、3** —— 不做的話,同一個 email 會長出兩個帳號,或帳號狀態判斷錯誤。
三項都是「Expression Policy 綁在附加上」,可以一起做、一起驗。

**選用:4、5** —— 目前不做,有需要再補。

## 1. 重複 email

**目標**(對照 artifact 故事二、三):

| 註冊時輸入的 email | 處理 |
| --- | --- |
| 沒人用過 | 照常建空白卡 |
| 已有**啟用中**的帳號 | 擋下,提示「這個 email 已經註冊過,請直接登入,或改用 Google 登入。」 |
| 已有**申請中的空白卡**(停用 + 路徑 `users/signup-pending`) | **沿用**那張空白卡,重寄確認信 |
| 已有**被停用**的帳號(不是空白卡) | 擋下,提示「這個帳號目前無法使用,請與我們聯繫。」—— 不能讓註冊重新打開被停用的帳號 |

**做法**:拆成兩條政策,因為「擋下並顯示訊息」和「沿用空白卡」要在不同時間點做。

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

# 停用中:只有空白卡可以繼續,被管理員停用的帳號一律擋下
if existing.path != "users/signup-pending":
    ak_message("這個帳號目前無法使用,請與我們聯繫。")
    return False

return True
```

- 用 `email__iexact`:大小寫不同視為同一個 email(Google 回來的大小寫不一定跟使用者打的一樣)
- ⚠️ 帳號名稱也是 email,而業務開的舊帳號帳號名稱不一定是 email。若有人的**帳號名稱**剛好等於
  別人的 email,User Write 會撞名報錯 —— 實測時注意

**1-b 沿用空白卡**:新增政策 `primary-signup-reuse-pending`,綁在流程 `primary-signup` 的
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
    # 把既有的空白卡設成「待處理的使用者」。
    # User Write 是「需要時建立」,看到已經有待處理的使用者就只更新、不新建;
    # 後面的 Email 階段也就寄給這張卡。
    plan.context["pending_user"] = pending

return True   # 一定回 True:回 False 會讓這個階段被跳過,空白卡就不會建
```

- ⚠️ `request.context["flow_plan"]` 在附加上的政策可用、且修改會留在流程裡 —— 依 Authentik 文件
  判斷,未實測
- 驗證:同一個 email 註冊兩次 → 使用者清單**只有一筆** → 收到兩封信,點**任一封**都能設密碼

## 2. 開通後移出「申請中」並加上永久標記

**問題**:路徑在**建立**時決定,開通後不會自動改。帳號一直留在 `users/signup-pending` 的話:

- 之後被管理員停用,看起來就跟空白卡一樣 → 1-a 會讓它重新註冊、3 會讓 Google 重新打開它
- 「申請中的空白卡有幾張」這個觀察指標失準

**做法**:新增政策 `primary-signup-finalize`,綁在流程 `primary-signup` **順序 60(`primary-signup-login`)
那一筆附加上**,「在執行階段時進行評估」開啟:

```python
plan = request.context.get("flow_plan")
user = plan.context.get("pending_user")
if user and user.pk:
    user.path = "users"
    # 永久標記:之後要統計「誰是 email 自行註冊的」靠這個,不靠路徑
    user.attributes["signup_source"] = "email"
    user.save()
return True
```

- 走到 60 代表已經點過信、設好密碼,這時移出才正確
- ⚠️ 政策裡直接存檔使用者(有副作用的政策)可行,但未實測

**驗證**:註冊完成後,帳號路徑變成 `users`、屬性有 `signup_source: email`。

## 3. Google 登入碰到空白卡時直接開通

**問題**(artifact 故事四):Google Source 的比對模式是「連結到相同 email 的使用者」。
使用者申請到一半改按 Google,Authentik 會把 Google 連到停用的空白卡,然後因為帳號停用而登入失敗。

Google 跟確認信一樣證明了 email 所有權,所以碰到**空白卡**時直接開通是安全的。
**只限空白卡**(路徑 `users/signup-pending`),被管理員停用的帳號不能因此重新打開。

**做法**:不要改 Authentik 內建的 `default-source-authentication`(由啟用中的內建藍圖管理),
另建自己的流程:

1. 建流程 `primary-source-authentication`(使用目的:身分認證)
2. 附加一個 User Login 階段(可沿用 `primary-signup-login`)
3. 在那筆附加上綁政策 `primary-source-activate-pending`(在執行階段時進行評估):

```python
plan = request.context.get("flow_plan")
user = plan.context.get("pending_user")
if user and user.pk and not user.is_active and user.path == "users/signup-pending":
    user.is_active = True
    user.path = "users"
    user.attributes["signup_source"] = "google"
    user.save()
return True
```

4. **使用者目錄 → 聯合與社群登入 → Google → 編輯 → 認證流程**:改成 `primary-source-authentication`

- ⚠️ 不確定 Source 的 email 比對會不會**直接略過**停用的使用者(那樣就不是「連到空白卡後登入失敗」,
  而是另建新帳號 → 同 email 兩個帳號)。先實測目前的實際行為再動手
- 用 Google 開通的空白卡沒有密碼;之後想用密碼登入就走忘記密碼

**驗證**:新 email 先填註冊(不點信)→ 改按 Google 登入 → 成功登入,使用者清單仍只有一筆,
已啟用、路徑 `users`;另拿一個**被停用的一般帳號**按 Google → 仍然登不進去。

## 4. 清理空白卡(選用)

空白卡沒有密碼、不會被冒用,1-b 和 3 也確保它不會擋住本人,所以**不急著刪**。
數量突然暴增可以當作機器人的警訊。保留 7~30 天後清除。

**查看數量**:**使用者目錄 → 使用者** → 左側 `users/signup-pending`。
申請的時間與來源 IP 也在 **事件** 裡,刪掉空白卡後仍查得到。

**定期清除**:Authentik 沒有內建「停用帳號 N 天後刪除」,在主機上排程:

```bash
# 刪除建立超過 14 天、仍未開通的空白卡
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
3. 附加到 `primary-signup`,**順序 15**(填完 email 之後、建空白卡之前)

**5-b 限制頻率**(輔助)

- Email 階段本身有 `Account Recovery Max Attempts` / `Cache Timeout`(預設 5 分鐘 5 次),
  能擋「對同一個 email 一直重寄」,擋不住「幾千個不同 email」
- ⚠️ Authentik 的名譽分數主要記錄登入失敗,不太適合限制註冊頻率。如果 Authentik 前面是
  Cloudflare,用 Cloudflare 的 Rate Limiting 規則限制 `/if/flow/primary-signup/` 與
  `/api/v3/flows/executor/primary-signup/` 的請求頻率更直接

**驗證**:無痕視窗註冊時出現 captcha;captcha 沒過就不會寄信、也不會建空白卡。

---

## 相關

- artifact〈會員註冊流程〉—— 空白卡做法的完整理由(給非工程人員)
- `docs/authentik-google-login.md` —— 一個 email 一個帳號、Google 比對模式
- `docs/authentik-email.md` —— SMTP 與寄信額度
- `docs/authentik-password-recovery.md` —— 同一個登入頁上的忘記密碼
