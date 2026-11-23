# B2E LINEOA 身份 — 下游串接規格

提供對象：LINE OA 下游系統
對應需求：ECB2E-10424

### 變更記錄

* v1.0 2026/09/14
  1. 首版：OIDC `line_oa_identity` claim、對外查詢 API、Kafka 快照 `LineOaIdentity` 節點

---

## 1. 這份資料是什麼

**LINEOA 身份**是 B2E 後台維護的「LINE OA 發送對象標籤」，用來決定一個員工在 LINE OA 上
可以使用哪些服務（Rich Menu、公告與審核通知的發送對象）。

它**不是 B2E 的權限角色**：身份不影響任何人在 B2E 後台能看到什麼、能做什麼。
B2E 的權限資料仍在 `systems` claim / 快照的 `Systems` 欄位，兩者互不相干。

身份由 B2E 後台「系統選單管理 → LINEOA身份維護」維護，並在「帳號及權限管理 → 人員角色」
頁指派給個別人員。**一名人員至多一個身份。**

## 2. ⚠️ 最重要的兩件事

### 2.1 「沒有身份」與「基本使用者」是兩種不同狀態

| 收到的值 | 語意 | 預期行為 |
| -------- | ---- | -------- |
| `null`（或欄位不存在） | **該帳號沒有使用 LINE OA 的資格** | 不應提供 LINE OA 服務 |
| `{ "identity_id": 10, "identity_name": "基本使用者" }` | 有資格，但只提供基本服務 | 給予基本服務 |

這兩者**不可互相代換**。B2E 後台的維護者可以刻意把某人的身份「取消」，那代表收回資格，
而不是降級成基本使用者。請務必分開處理。

### 2.2 身份 Id 可能被重新指定給不同的身份

B2E 後台允許刪除身份，被刪除的 `identity_id` **日後可能被建立成另一個意義完全不同的身份**
（例如今天的 `60` 是「客服」，刪掉後明天的 `60` 可能是「倉儲」）。

因此：

- **請勿長期快取 `identity_id` → 名稱／行為 的對應表。**
- 每次收到資料時以當下的 `identity_id` 與 `identity_name` 為準。
- 若你的系統要把身份對應到 Rich Menu 等設定，建議在 B2E 端異動身份時同步確認。

同理，身份的**名稱也可能被修改**（名稱改了 B2E 會重發快照，見第 5 節）。

## 3. 目前的身份清單

上線初始為以下 5 筆，`identity_id` 已與貴方約定，B2E 不會自行改號：

| identity_id | identity_name | 說明 |
| ----------- | ------------- | ---- |
| 10 | 基本使用者 | 綁定LINEOA後只提供基本服務。（**系統保留身份**，B2E 端不可停用或刪除） |
| 20 | 全聯採購 | 全聯商品部採購人員，推播通知，商品查詢，可銷量，加減量，鎖檔及商談回覆等作業。 |
| 30 | 全電商MD | 推播通知，商品查詢，可銷量，隱藏賣場開關等作業。 |
| 40 | 全聯行銷 | 推播通知，商品查詢，可銷量等作業。 |
| 50 | 廠服 | 發起商談，商品查詢，供應商查詢等作業。 |

後續可由 B2E 後台新增其他身份，新身份的 Id 由維護者指定或系統自動給號。

**關於身份 10 的特殊性**：當 B2E 端停用或刪除某個身份時，原本綁定該身份的帳號會被
**自動改為身份 10（基本使用者）**，而不是變成 `null`。換句話說，這些人仍保有 LINE OA 資格，
只是降為基本服務。貴方會透過快照或下次查詢收到變更後的值。

---

## 4. 取得方式一：OIDC（綁定流程當下）

貴方既有的 B2E OIDC 登入流程不需要任何改動，`id_token` 與 `/connect/userinfo` 會多一個
`line_oa_identity` claim。

### 4.1 id_token

```json
{
  "sub": "1234",
  "name": "王小明",
  "email": "xxx@pxmart.com.tw",
  "systems": [ ... ],
  "line_oa_identity": {
    "IdentityId": 20,
    "IdentityName": "全聯採購"
  }
}
```

- 它是**巢狀 JSON 物件**，不是字串，可直接反序列化。
- **未設定或身份已停用時，此 claim 不會出現在 token 中**（等同 `null`，語意見 2.1）。
- 既有的 `systems` claim 結構與內容**完全沒有變動**。

### 4.2 GET/POST /connect/userinfo

以 `access_token` 呼叫，回應會包含同一份資料（此處未設定時為明確的 `null`）：

```json
{
  "sub": "1234",
  "name": "王小明",
  "email": "xxx@pxmart.com.tw",
  "systems": [ ... ],
  "line_oa_identity": { "IdentityId": 20, "IdentityName": "全聯採購" }
}
```

`id_token` 與 `userinfo` 兩者的 `line_oa_identity` 內容保證一致。

---

## 5. 取得方式二：Kafka 快照（帳號異動時）

B2E 既有的人員權限快照事件新增了 `LineOaIdentity` 節點。

- **Topic**：`PXB2E.Internal.MemberSnapshotChanged`
- **觸發時機**：人員、角色、人員角色、LINEOA 身份任一異動時，對受影響的人員各發一筆
  **當下完整快照**（不是差異）。另有每日全量對帳排程重送全部人員，可用於補漏。
- **既有欄位完全未變**，本次僅新增一個同層節點。

```json
{
  "MemberId": 1234,
  "Account": "xxx@pxmart.com.tw",
  "DisplayName": "王小明",
  "Status": 1,
  "Systems": [ ... ],
  "LineOaIdentity": { "IdentityId": 20, "IdentityName": "全聯採購" }
}
```

- 未設定或身份已停用時，`LineOaIdentity` 為 `null`。
- 欄位為 PascalCase（與既有快照欄位一致）。

**會收到快照的情況包含**：該人員被指定／變更／取消身份、其綁定的身份被改名、
其綁定的身份被停用或刪除（此時值會變成身份 10）、以及每日全量對帳。

---

## 6. 取得方式三：查詢 API（重查與對帳）

### 6.1 認證

```
Authorization: Bearer {token}
```

- token 由 B2E 提供（32 字元），屬**系統對系統**的長效憑證，與 OIDC 的 `access_token` 不同——
  背景對帳時手上不會有使用者的 token，請使用本憑證。
- 未帶或無效憑證回 **401**。

### 6.2 GET /api/external/1.0/line-oa-identity/member

查詢單一人員目前的 LINEOA 身份。

**參數**（`account` 與 `employee_number` 擇一必填）

| 參數 | 型別 | 說明 |
| ---- | ---- | ---- |
| `account` | string | B2E 帳號（即公司信箱） |
| `employeeNumber` | string | 員工編號 |

**回應**

```json
{
  "success": true,
  "data": {
    "found": true,
    "identity": { "identity_id": 20, "identity_name": "全聯採購" }
  }
}
```

**三種情形請分開處理：**

| 情形 | `found` | `identity` | 意義 |
| ---- | ------- | ---------- | ---- |
| 查無此帳號／員編 | `false` | `null` | B2E 沒有這個人 |
| 有此人，但無資格 | `true` | `null` | **該帳號沒有使用 LINE OA 的資格** |
| 有此人且有身份 | `true` | 物件 | 正常情形 |

**錯誤回應**（未提供任何查詢條件）

```json
{ "success": false, "message": "請提供帳號或員工編號" }
```

---

## 7. 建議的串接方式

1. **綁定當下**：走既有 OIDC 流程，直接讀 `id_token` 的 `line_oa_identity`，不必額外打 API。
2. **日常異動**：訂閱 Kafka topic 接收快照，收到後以 `MemberId` 或 `Account` 更新貴方端的
   員工快取（`sys_employees`）。
3. **對帳／重查**：以查詢 API 定期核對，或在懷疑資料不同步時單筆重查。

三種來源的資料保證一致，都以 B2E 當下的 `USR_Member.LineOaIdentityId` 為準。

---

## 8. 聯絡

B2E 端負責人：全電商 資訊部（ECB2E-10424）
