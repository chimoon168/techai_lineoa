# B2E 人員權限快照事件（Kafka）— 下游串接規格

提供對象：LINE OA 下游系統
對應需求：ECB2E-10424

### 變更記錄

* v1.0 2026/09/14
  1. 首版：`MemberSnapshotSyncEvent` 完整欄位規格
  2. 新增 `LineOaIdentity` 節點（ECB2E-10424）
  3. message key 改為 `MemberId`，同一人員的快照保證依序抵達（ECB2E-10424）
  4. 明確說明不可用 `MemberUpdateOn` 當版本號丟棄訊息（身份／角色改名不會更新它）
  5. 補上順序保證的**界線**：partition key 只保證「送出順序＝抵達順序」，
     組裝與送出之間未序列化，仍可能舊蓋新；建議做法與已知限制見 §2.1–2.3

---

## 1. 這個事件是什麼

B2E 在**人員、角色、角色權限、人員角色、模組、功能、LINEOA 身份**任一異動時，
對「受影響的每一位人員」各發佈一筆**該人員當下的完整快照**。

三個關鍵特性：

1. **是完整快照，不是差異（diff）。** 每筆訊息都帶該人員當下的全部資料，
   收到後直接覆蓋你端的資料即可，不需要套用增量。
2. **不帶異動類型。** 沒有 CREATE/UPDATE/DELETE 旗標——請自行比對既有資料決定是否更新。
   「刪除」的情境由 `Status = 9` 表達。
3. **一次異動可能產生多筆訊息。** 例如修改一個角色，B2E 會對「擁有該角色的每一位人員」
   各發一筆。角色底下有 200 人就會有 200 筆。

另有**每日全量對帳排程**，會對所有未刪除人員各重送一筆快照，用於補漏。

## 2. Topic 與連線

| 項目 | 值 |
| ---- | -- |
| Topic | `PXB2E.Internal.MemberSnapshotChanged` |
| 格式 | JSON（UTF-8） |
| 欄位命名 | **PascalCase**（`MemberId`、`DisplayName`…） |

Broker 位址、帳號密碼與 consumer group 設定請另向 B2E 端索取（不寫在本文件）。

> ⚠️ **message key 變更（2026-09）**：本 topic 的 key 已由隨機 GUID 改為 `MemberId`。
> 若貴方（或其他既有消費者）曾把 key 當成「事件唯一識別」使用，**必須改用訊息內的 `Uuid`**。
> 切換期間 topic 中可能同時存在舊 key（GUID）與新 key（MemberId）的訊息並交錯抵達。

### 2.1 訊息順序與其限制（請完整讀完再決定實作方式）

**保證的部分**：message key ＝ `MemberId`，同一位人員的快照固定落在同一個 partition，
因此 **B2E 送出的順序 = 你收到的順序**。不同人員之間沒有順序保證（也不需要）。

**⚠️ 不保證的部分**：B2E 在「讀資料組裝快照」與「送出」之間**沒有跨請求的序列化**。
極端情況下仍可能發生：

```text
請求 A：讀取資料（舊）─────────────────────→ 送出 A（內容較舊）
請求 B：          讀取資料（新）→ 送出 B（內容較新）
                                    ↑            ↑
                              你先收到 B     再收到 A → 舊的蓋掉新的
```

多台 host 併發、或每日全量對帳與即時事件交錯時，都可能出現。

**因此**：不要把「後收到的一定比較新」當成前提來設計。

### 2.2 ⚠️ 也不要用 `MemberUpdateOn` 當版本號

看到 2.1 的限制後，直覺會想用時間戳判斷新舊——但 `MemberUpdateOn` 不適合，因為
**很多快照不是由「人員本身」的異動觸發的**，這些情況它**不會改變**：

| 觸發情境 | `MemberUpdateOn` 會變嗎 | 快照內容有變嗎 |
| -------- | ---------------------- | -------------- |
| 該人員的帳號資料被修改 | ✅ 會 | ✅ 會 |
| 該人員的 LINEOA 身份被指定／取消 | ✅ 會 | ✅ 會 |
| **該人員綁定的身份被改名** | ❌ **不會** | ✅ **會**（`IdentityName` 變了） |
| **該人員的角色被改名或調整權限** | ❌ **不會** | ✅ **會** |
| 每日全量對帳重送 | ❌ 不會 | 可能不變 |

用它丟棄訊息，會把身份改名、角色改名這類快照全部丟掉，而且每日全量對帳也補不回來
（它的 `MemberUpdateOn` 同樣沒變）。

### 2.3 建議的做法

在 B2E 提供更強的順序保證之前（見下方「已知限制」），建議：

1. **直接以收到的快照覆蓋**你端資料，不自行判斷新舊——快照是完整的，覆蓋是冪等的。
2. **接受最終一致**：偶發的「舊蓋新」會讓你端資料短暫過時，
   由**每日全量對帳**收斂（對帳送的是當下最新快照）。
3. **對即時正確性要求高的動作**（例如即將依身份決定要不要推播機密通知），
   建議在動作前用**查詢 API**（`/api/external/1.0/line-oa-identity/member`）重查一次，
   不要單純信任快取。

> **已知限制**：B2E 端尚未提供「跨 host 的組裝／發佈序列化」或「涵蓋全部異動來源的快照版本號」。
> 若貴方對順序有更嚴格的需求，請與 B2E 端聯繫評估，不要自行以 `MemberUpdateOn` 之類的欄位替代。

### 2.4 重複訊息

同一筆快照可能因重試或每日全量對帳而重複收到。快照是完整的、直接覆蓋即可，
因此**重複套用本身是冪等的、不會造成錯誤**。

若要省下重複處理的成本，可用每則訊息的 `Uuid`（唯一值）做去重——
請把它當成效能最佳化，而不是正確性的必要條件。
⚠️ **`Uuid` 是「事件」的識別，不是「內容版本」**，不可用來比較新舊。

## 3. 訊息結構

```json
{
  "Uuid": "3f2a9c1b4e5d6f708192a3b4c5d6e7f8",
  "CreateOn": "2026-09-14T16:30:00",
  "TriggerSource": "RoleMember",
  "OperatorId": 1234,

  "MemberId": 5678,
  "Account": "someone@pxmart.com.tw",
  "Email": "someone@pxmart.com.tw",
  "DisplayName": "王小明",
  "Status": 1,
  "ActivationDate": "2024-01-01T00:00:00",
  "ResignationDate": null,
  "LastLoginTime": "2026-09-14T09:12:00",
  "MemberCreateOn": "2024-01-01T10:00:00",
  "MemberUpdateOn": "2026-09-14T16:29:58",
  "Department": "資訊部",
  "DepartmentNo": 3,
  "SystemTypes": "1,4",

  "Systems": [ ... ],

  "LineOaIdentity": { "IdentityId": 20, "IdentityName": "全聯採購" }
}
```

### 3.1 事件層欄位

| 欄位 | 型別 | 說明 |
| ---- | ---- | ---- |
| `Uuid` | string | 本則訊息的唯一識別，可用於去重 |
| `CreateOn` | datetime | **事件產生時間**（非人員建檔時間） |
| `TriggerSource` | string | 觸發來源，僅供除錯：`Member`／`Role`／`RolePermission`／`RoleMember`／`Module`／`Function`／`DailySync` |
| `OperatorId` | long | 操作者的人員 Id |

### 3.2 人員基本資料

| 欄位 | 型別 | 說明 |
| ---- | ---- | ---- |
| `MemberId` | long | **人員 Id（主鍵，必填）** |
| `Account` | string | 帳號（即公司信箱） |
| `Email` | string? | 電子信箱。目前帳號即公司信箱，`Account` 為 email 格式時同值，否則為 `null` |
| `DisplayName` | string | 姓名 |
| `Status` | int | **帳號狀態：`1` 啟用／`2` 停用／`9` 刪除（離職）** |
| `ActivationDate` | datetime? | 啟用日 |
| `ResignationDate` | datetime? | 離職日 |
| `LastLoginTime` | datetime? | 最後登入時間 |
| `MemberCreateOn` | datetime | 人員建檔日期 |
| `MemberUpdateOn` | datetime | 人員編輯日期。⚠️ **不可當成快照的版本號**——身份／角色改名等情境不會更新它（見 2.2） |
| `Department` | string? | 所屬單位名稱 |
| `DepartmentNo` | int? | 所屬單位編號（權威識別，請以編號比對而非名稱） |
| `SystemTypes` | string | 可用系統，多選 CSV，如 `"1,4"` |

> **離職／停用處理**：`Status` 為 `2`（停用）或 `9`（刪除／離職）時，
> 建議一併解除該員工的 LINE 綁定或停止推播。

### 3.3 `LineOaIdentity`（ECB2E-10424 新增）

| 欄位 | 型別 | 說明 |
| ---- | ---- | ---- |
| `IdentityId` | long | 身份 Id（下游識別鍵） |
| `IdentityName` | string | 身份名稱（可能被修改） |

⚠️ **`null` ＝ 該帳號沒有使用 LINE OA 的資格**，與身份 `10`「基本使用者」
（有資格、僅基本服務）是**不同狀態**，請分開處理。

身份清單、Id 重用注意事項與其他取得方式，詳見 **`B2E_LINEOA身份API規格文件.md`**。

### 3.4 `Systems`（B2E 權限，非 LINE OA 相關）

人員在各系統的角色與可存取選單。**若你只需要 LINEOA 身份，此節點可以忽略。**

```json
"Systems": [
  {
    "SystemType": 1,
    "SystemName": "B2E",
    "Roles": [
      { "RoleId": 30, "RoleName": "商開", "Departments": [ { "DepartmentNo": 1, "DepartmentName": "全聯商品部" } ] }
    ],
    "Modules": [
      {
        "ModuleId": 106, "Title": "系統選單管理", "Icon": "fa fa-cog", "DisplayOrder": 6,
        "Functions": [ { "FunctionId": 10607, "Title": "LINEOA身份維護", "Action": "/PermissionsSettings/line-oa-identity", "DisplayOrder": 8 } ]
      }
    ]
  }
]
```

| 欄位 | 說明 |
| ---- | ---- |
| `SystemType` | 系統別：`0` 系統／`1` B2E／`2` SCM／`3` 3PL／`4` LOS |
| `Roles` | 該系統下**啟用中**的角色；無則為 `null` |
| `Roles[].Departments` | 角色的部門清單。**空陣列＝通用角色**（不受部門限制），永遠是陣列不會是 `null` |
| `Modules` / `Functions` | 可存取的模組與功能；`ModuleId`／`FunctionId` 為**對外權限識別碼**（非內部流水號） |

- 人員沒有任何啟用角色或選單時，整個 `Systems` 為 `null`（不是空陣列）。
- 各系統下 `Roles`／`Modules` 無資料時為 `null`。

---

## 4. 建議的處理流程

```text
1. 收到訊息 →（可選）以 Uuid 去重，純為省成本
2. 直接以 MemberId 為鍵 upsert 你端的員工快取（sys_employees）
   ⚠️ 不要用 MemberUpdateOn 或 Uuid 判斷新舊而丟棄訊息（見 2.2、2.4）
   ⚠️ 極少數情況可能被較舊的快照覆蓋，由每日全量對帳收斂（見 2.1、2.3）
3. 依 Status 判斷是否停用／解除綁定
4. 依 LineOaIdentity 更新該員工的 LINE OA 身份
   → null 代表無資格，請一併處理（不是「維持原樣」）
```

⚠️ 第 4 步的 `null` 特別容易被寫成「沒帶就不動」。快照是完整的，`null` 是明確的值，
代表「這個人現在沒有資格」，必須實際清掉你端的身份設定。

---

## 5. 與其他取得方式的關係

| 來源 | 時機 | 適用 |
| ---- | ---- | ---- |
| OIDC `line_oa_identity` claim | 使用者綁定／登入當下 | 綁定流程 |
| **本事件（Kafka）** | B2E 端資料異動時 | 日常同步 |
| 查詢 API | 主動查詢 | 對帳、單筆重查 |

三者資料一致，都以 B2E 當下的資料為準。詳見 `B2E_LINEOA身份API規格文件.md`。

---

## 6. 聯絡

B2E 端負責人：全電商 資訊部（ECB2E-10424）
