# 05｜SYSOP 管理後台 — 開發規格書

Nov 20, 2026 · chimoon 高千月 · 全電商技術處

| 項目 | 內容 |
| --- | --- |
| 版本 | v1.1（2026-11-20：補登入頁畫面，對齊 UAT `/login`） |
| 畫面原型 | `lineoa_backend/index.html`（登入）、`LINEOA圖文選單.html`、`每日語音列表.html`、`日結摘要列表.html`、`事件路由列表.html` |
| 基準程式 | TechAI20261116.zip |
| 技術 | Razor Pages、LuluSysop Policy |
| 相關文件 | 02 Rich Menu 配發、03 語音、04 日結、06 事件路由；參考：畫面規格 v0.5、測試規格 v2.0 §4 |

## 1. 登入與 Session

| 項目 | 規格 |
| --- | --- |
| 登入 | B2E OIDC SSO，callback `/signin-oidc-backoffice` |
| 准入條件 | B2E Status=1、Type=1、LINE OA Identity=90（小露SYSOP） |
| Session | 8 小時絕對到期；Cookie＋定期向 B2E 查核 |
| 與 LINE 綁定 | **無關**：不需先綁定 LINE；LINE 綁定也不會取得後台權限 |
| MFA | 信任證據需安全落實（待驗收） |

### 1.1 登入頁畫面（對齊 UAT）

UAT 網址：`https://techai-uat.pxec.com.tw/login?ReturnUrl=%2Frichmenus%2Frichmenu%2F`

| 區塊 | 內容 |
| --- | --- |
| 品牌區 | 🎙️「全電商技術處 小露AI」、副標「LINE 語音智能助理 · 技術處辦公室自動化」 |
| 卡片標題 | 登入；鎖頭圖示；「小露 AI 管理後台」；說明「請透過 B2E 企業帳號驗證身分，再進入小露 AI 管理後台。」 |
| 步驟提示 | ① 請先連接公司 VPN。② 按下「前往 B2E 登入」完成帳密驗證。③ SYSOP 身分核對成功後，自動返回小露後台。 |
| 主按鈕 | 「前往 B2E 登入」（綠色）→ OIDC Challenge 導向 B2E |
| 次要連結 | 「需要切換 B2E 帳號？」→ 先登出 B2E 再導向（OIDC `prompt=login` 或等效作法，以程式碼為準） |
| 頁尾說明 | 此頁不會要求輸入 B2E 密碼或驗證碼。登入與二次驗證均在 B2E 網站完成。小露登出不會同時登出 B2E。 |

**流程**

1. 未登入存取任一後台頁 → 導向 `/login?ReturnUrl={原路徑}`。
2. 按「前往 B2E 登入」→ B2E OIDC 登入（含二次驗證）→ callback `/signin-oidc-backoffice`。
3. 後端核對 Status=1、Type=1、Identity=90 → 簽發 Cookie，導回 `ReturnUrl`（僅允許站內相對路徑，防 open redirect）；無 ReturnUrl 預設進「每日語音」。
4. 核對失敗 → 回登入頁顯示原因（非 SYSOP 身份／帳號停用），不簽發 Cookie。
5. 登出只清除小露 Cookie，不登出 B2E。

## 2. 授權

- 所有後台頁與 API 套用 `LuluSysop` Policy。
- 各頁**重新檢查 B2E 身份**；B2E 將 SYSOP 改為其他身份或停用後，下次查核即失去權限，編輯中儲存被拒。
- 非 SYSOP 直接輸入網址或呼叫 API 一律拒絕（403）並寫稽核。

## 3. 管理功能

| 功能 | 內容 |
| --- | --- |
| 語音摘要列表／明細 | 依日期、LINE 帳號／群組篩選；明細顯示標題、摘要、逐字稿、來源時間 |
| 每日摘要 | 日結列表／內容；手動執行日結（見 04） |
| Rich Menu | 草稿、發布、身份選單指定（見 §4） |

列表／明細欄位沿用畫面規格 v0.5 §3–6，實作以程式碼為準。

## 4. Rich Menu 生命週期

| # | 步驟 | 內容 |
| --- | --- | --- |
| 1 | 建立草稿 | 類型：UNBOUND／BOUND_GENERAL／IDENTITY |
| 2 | 設計版面 | 2 格（半版）、4 格、6 格（全版）；上傳圖片 |
| 3 | 設定 Action | 每格：MESSAGE／URI／POSTBACK |
| 4 | 發布到 LINE | 呼叫 LINE API 建立，取得 `LineRichMenuId` |
| 5 | 指定身份選單 | 選擇 B2E LINE OA Identity，寫入 `sys_line_rich_menu_assignments` |
| 6 | 批次同步 | LineRichMenuSyncWorker 對現有綁定員工配發；異動重試 |

**欄位與檢核（沿用 UAT 現況與測試規格 v2.0 §4）**

| 欄位 | 規則 |
| --- | --- |
| 名稱 | 必填 |
| 選單列文字 | ≤14 字 |
| 預設展開 | 是／否 |
| 圖片 | PNG／JPEG ≤1 MB；半版 2500×843、全版 2500×1686 |
| MESSAGE | Label＋訊息文字 |
| URI | Label＋URL／LIFF |
| POSTBACK | Label＋Postback Data＋Display Text |

**操作規則**

- 儲存草稿不影響 LINE 端；儲存並發布才更新。
- 預設選單（UNBOUND）同時只能一筆。
- 身份選單停用後，該身份使用者回落 BOUND_GENERAL。
- 預設或作用中選單不可直接刪除。
- 身份下拉來源為 B2E LINE OA 身份清單，非寫死選項。

## 5. 驗收條件

- Identity=90 可登入；10／20／未設定身份不可登入或操作（403）。
- 8 小時後強制重新登入；期間 B2E 改身份，後台權限即時失效。
- 草稿 → 發布 → 指定身份 → 綁定員工看到新選單，全程可完成。
- 圖片尺寸／大小、選單列文字長度、必填欄位檢核生效。
- 對應測試：測試規格 v2.0 VOICE-09～11、RM-01～RM-16。

## 6. 待確認

1. MFA 信任證據落實方式與驗收標準（二次驗證在 B2E 完成，小露端如何確認）。
2. 「切換 B2E 帳號」的實作方式（`prompt=login` 或先導 B2E 登出）。
3. 定期查核 B2E 的頻率。
4. 同一身份可否有多筆作用中選單（目前預期：一筆）。
5. 後台稽核紀錄查詢畫面是否納入。
