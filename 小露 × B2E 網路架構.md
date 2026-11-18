技術處辦公室自動化 · 小露 LINEOA · 草案 v0.1 · 2026-10-07

# 小露 × B2E 網路架構

作者：chimoon 高千月 · 全電商技術處

評估另開一條「小露專屬」的 B2E 業務 API 通道，承載商品查詢、變價查詢、採購加減量、變價鎖檔，與既有 OIDC 綁定通道分離。

## 結論先講

- **OIDC 留在「身分」用途。**它必須對 Internet 開放（使用者手機的 LIFF 瀏覽器要直接連 `/connect/authorize`），白名單是 client_id / redirect_uri 這類應用層設定，不適合再掛業務寫入權限。
- **新開「B2E LINEOA 業務 API」通道，只走內網。**獨立 listener / 路徑前綴 `/api/lineoa/1.0/*`、獨立憑證（mTLS + API Key/HMAC）、防火牆只放行小露主機，不經過 OIDC。
- **使用者身分以「代理呼叫」帶入，由 B2E 重驗。**小露送出 MemberId，B2E 在每次請求當下檢查帳號狀態與 LINEOA 身份，寫入類作業（加減量、鎖檔）另加冪等鍵與稽核。

## 系統架構與網路分區

既有 LINE / 小露流量 OIDC 身分通道（既有） 小露 → B2E 業務通道（新） 身份同步 / 對帳（既有）

OIDC（③④）與新業務通道（⑤）在 B2E 端是兩個入口：前者要讓手機瀏覽器從 Internet 連得到，後者只接受小露 Adapter 從內網連入。小露位置以「同一機房／VPC 內網」為前提，若部署在雲端需另加 site-to-site VPN（見待確認 1）。

## 各通道網路設定

| # | 通道 | 來源 → 目的 | 協定 / Port | 曝露面 | 認證 / 防護 | 設定方 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | LINE Webhook 入站 | LINE Platform → WAF → 小露 API | HTTPS 443 | Internet 入站 | X-Line-Signature（HMAC）。LINE 未公開固定來源 IP，入站無法靠 IP 白名單 | 網管（WAF/DNS/憑證） |
| 2 | 小露出站 | 小露 → Egress → api.line.me、api-data.line.me、Google API | HTTPS 443 | Internet 出站 | Channel Access Token；Proxy 以 FQDN 白名單放行 | 網管 |
| 3 | OIDC 前端通道 | 手機 LIFF 瀏覽器 → B2E `/connect/authorize` | HTTPS 443 | Internet（B2E 既有公開入口） | PKCE、state；redirect_uri 白名單 | B2E（OIDC client 設定） |
| 4 | OIDC 後端通道 | 小露 API → B2E `/connect/token`、`/userinfo` | HTTPS 443 | 內網或公開入口皆可 | client_secret；只拿身分，**不帶業務 scope** | B2E |
| 5 | **小露業務通道（新）** | 小露 Adapter → B2E LINEOA 業務 API | HTTPS 443（內網 listener） | **僅內網**，不發布到 Internet | mTLS（client 憑證）＋ API Key/HMAC（timestamp、nonce 防重放）＋ 來源 IP 白名單 ＋ 限流 | 網管（FW 規則、DNS）、B2E（API、金鑰）、資安（憑證） |
| 6 | 身份快照 | 小露 Worker ← Kafka broker | Kafka（SASL/TLS，port 依 B2E 提供） | 僅內網 | Consumer 帳號、group 由 B2E 發放 | B2E |
| 7 | 身份對帳 | 小露 → `/api/external/1.0/line-oa-identity/member` | HTTPS 443 | 僅內網（建議） | B2E 發的 32 字元 Bearer token | B2E |

## 為什麼不與 OIDC 混用

| 比較 | A. 借用 OIDC（加 scope / client_credentials） | B. 另開小露專屬業務通道（建議） |
| --- | --- | --- |
| 網路曝露 | OIDC 必須對 Internet 開放，業務授權跟著掛在公開入口上 | 業務 API 只開內網 listener，Internet 看不到 |
| 影響範圍 | OIDC 服務其他系統；小露的 scope、token 壽命、限流調整都會牽動共用設定 | 與 OIDC 解耦，小露的限流、停用、金鑰輪替只影響自己 |
| 使用者在場與否 | 背景作業（推播後的鎖檔、對帳）手上沒有使用者 token | 系統憑證 ＋ 代理 MemberId，前景、背景都能呼叫 |
| 權限語意 | OIDC 的 `systems` claim 是 B2E 後台權限；LINEOA 身份規格明確寫「不是 B2E 權限角色」 | B2E 端依 `LineOaIdentityId` 對應功能，規則集中在業務 API |
| 出事時 | 要關小露只能動 OIDC client，可能誤傷綁定流程 | 撤銷 Adapter 憑證或 FW 規則即可斷開，綁定不受影響 |

## 新通道承載的 4 項功能

| 功能 | 建議路徑 | 類型 |
| --- | --- | --- |
| 查詢商品 基本資料 / 狀態 / 可銷量 | `GET /products/{pxNo}` | 讀 |
| 查詢變價 銷售門市 / 取貨門市 / 活動起迄 / 限量 | `GET /price-changes?pxNo=` | 讀 |
| 採購加減量 ＝供應商送審 + autopass | `POST /purchase-adjustments` | 寫 |
| 變價鎖檔 | `POST /price-changes/{id}/lock` | 寫 |

路徑皆在 `/api/lineoa/1.0` 之下。依身份規格，加減量與鎖檔目前只屬「20 全聯採購」；30 全電商MD、40 全聯行銷可查商品與可銷量；變價查詢的開放對象待確認

## 請求要帶的東西

```
POST /api/lineoa/1.0/purchase-adjustments
(mTLS client cert: lulu-adapter)
X-Client-Id:      lulu-lineoa
X-Timestamp:      1791340800
X-Nonce:          7f3c…
X-Signature:      HMAC-SHA256(secret,
                    method+path+ts+nonce+sha256(body))
X-Acting-Member:  1234        ← 綁定的 B2E MemberId
X-Line-User-Id:   U1a2b…      ← 稽核用
Idempotency-Key:  7c1e…       ← 寫入類必帶
```

B2E 收到後依當下 `USR_Member` 重驗：在職、身份非 null、身份允許此功能；不信任小露端快取（身份規格 §2.3 亦建議敏感動作前重查）。

## 待確認事項

1. **小露部署位置**：與 B2E 同機房／VPC，或在 GCP？若跨雲，⑤⑥⑦ 需走 site-to-site VPN 或專線，不建議改成公開 API。— 網管
2. **B2E 是否已有內部 API Gateway** 可掛 `/api/lineoa/*`，或由 B2E 另起內網 listener／子網域（例如 `b2e-lineoa.internal`）。— B2E 開發、網管
3. **認證強度**：mTLS ＋ HMAC 雙層，或僅 HMAC ＋ IP 白名單；憑證與 secret 的存放與輪替週期。— 資安
4. **寫入類的 create_by**：加減量、鎖檔在 B2E／SCM 記錄為「操作人 MemberId」或「系統帳號＋代理人」；autopass 是否需要另一層主管確認。— PM、SCM
5. **LINE 端確認步驟**：寫入前是否強制在 LINE 顯示確認卡片（數量、門市、起迄），避免誤觸。— PM
6. **限流與時效**：每人／每分鐘上限、B2E 回應逾時（LINE reply token 約 1 分鐘有效，查詢要在時限內回覆，否則改用 push）。— B2E、小露開發