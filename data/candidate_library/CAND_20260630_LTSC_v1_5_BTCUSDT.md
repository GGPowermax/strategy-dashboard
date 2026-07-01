# 候選策略：LTSC v1.5 — BTCUSDT

**來源**：`mining_queue/MINE_20260630_0600_Twitter-Substack_LTSC_leverage_tax_saturation_cascade.md`
**Layer 1 回測日期**：2026-06-30 08:15
**狀態**：L2_REJECTED ❌

---

## 策略信號邏輯

**名稱**：Leverage Tax Saturation Cascade v1.5  
**方向**：做空（short perpetual）  
**時框**：4H  
**資產**：BTCUSDT（L1通過）｜ETHUSDT（L1失敗）｜SOLUSDT（未測試）

**進場邏輯（全部 AND）**：
```python
# 1. 累積 FR Z-score（fr_lookback_8h=9 → 18×4H = 72H）
fr_cum = fr.rolling(18).sum()
fr_cum_z = (fr_cum - fr_cum.rolling(730).mean()) / fr_cum.rolling(730).std()
entry_z: fr_cum_z > 1.5

# 2. Taker Sell Ratio（過去 3×4H = 12H）
sell_ratio_3bar = (taker_sell / (taker_buy + taker_sell)).rolling(3).mean()
entry_sell: sell_ratio_3bar > 0.505

# 3. 價格停滯門（未突破 5 日高點）
not_breakout: close < rolling_30bar_high * 0.995
```

**出場邏輯**：
- FR 回歸：fr_cum_z < 0.5（FR 正常化）
- 停損：close > entry_close × 1.05（+5% 停損）

---

## Layer 1 回測結果（BTCUSDT）

| 指標 | 值 | 門檻 | 通過 |
|------|-----|------|------|
| OOS Sharpe (ann.) | **1.817** | > 1.5 | ✅ |
| OIS ratio | **17.60** | > 0.5 | ✅ |
| Profit Factor | **1.565** | > 1.0 | ✅ |
| Param Stability | **100%** | ≥ 70% | ✅ |
| Max Drawdown | -3.93% | — | — |
| CAGR | +16.83% | — | — |
| Turnover/yr | 2.7次 | — | — |
| Cost drag (SR) | 0.049 | — | — |
| DSR | 0.971 | — | — |
| OOS obs | 3,240 bars | — | — |
| N folds | 18 | ≥ 6 | ✅ |

**最佳參數**：`{fr_cum_z_thr: 1.5, sell_ratio_thr: 0.505, fr_lookback_8h: 9, z_baseline_8h: 365, stoploss_pct: 0.05}`

---

## Fold 表現分布（BTCUSDT，18 folds，IS=1095 OOS=180 4H bars）

| Fold | IS_SR/obs | OOS_SR/obs | OOS_SR_ann | 備注 |
|------|-----------|------------|------------|------|
| 1 | -0.068 | nan | nan | 早期無信號 |
| 2 | -0.068 | nan | nan | 早期無信號 |
| 3 | -0.068 | nan | nan | 早期無信號 |
| 4 | -0.068 | nan | nan | 早期無信號 |
| 5 | -0.066 | 0.017 | +0.78 | |
| 6 | -0.027 | 0.047 | +2.19 | IS持倉延伸 |
| 7 | 0.015 | -0.023 | **-1.07** | ⚠️ FRAGILE |
| 8 | 0.003 | 0.098 | +4.59 | |
| 9–16 | ~0.01-0.03 | nan | nan | 中段無信號 |
| 17 | nan | 0.044 | +2.05 | |
| 18 | 0.018 | 0.201 | +9.41 | 近期強勢 |

**統計**：有交易 OOS fold：6/18 | 正SR：5/6 | 負SR：1/6（fold 7）

---

## 資料驗證

- ✅ OHLCV rows: 4,379（≥ 2000）
- ✅ 欄位：close, fr, taker_buy_vol, taker_sell_vol 全部存在
- ✅ 資料新鮮度：最後一筆距今 0.2H（< 24H）
- ⚠️ OI：未使用（策略設計 OI-free，使用 Taker Volume 代理）

---

## 旗標

| 旗標 | 說明 |
|------|------|
| **FRAGILE** | Fold 7 OOS_SR_ann = -1.07 < 0，單 fold 負 SR。6 個有交易 fold 中唯一負例，整體仍達標。 |
| **LOW_SIGNAL_FREQUENCY** | 18 folds 中 12 個 OOS 窗口 0 新交易（IS 持倉延伸至 OOS 貢獻部分回報）。年化換手僅 2.7 次。 |
| **OIS_UNSTABLE_NUMERIC** | OIS=17.6 因 IS_SR_mean=0.0022 趨近 0 導致比值放大。方向性正確（OOS > IS），但絕對值無意義。 |
| DSR_UNRELIABLE | 低頻策略（< 100 trades/yr）的必然旗標，不影響通過判斷。 |

---

## ETHUSDT 結果（失敗記錄）

| 指標 | 值 |
|------|-----|
| OOS Sharpe (ann.) | 0.772 ❌（門檻 > 1.5）|
| OIS ratio | nan（UNSTABLE_IS_BASELINE）|
| Profit Factor | 1.161（通過）|
| 旗標 | UNSTABLE_IS_BASELINE, OVERFIT_SUSPECT |
| 失敗原因 | ETH IS_SR 全程接近或低於 0；BTC FR 結構更集中，ETH 信號更雜亂 |

---

## Decomposer 三柱（沿用 Track A）

**柱 1 [P2 行為金融 — 沉沒成本陷阱]（中強）**  
Longs 累積支付高 FR 後因沉沒成本謬誤持續持倉。BTC 72H 累積 FR Z=1.726 > 1.5 閾值，年化 ~10%（1x）。行為偏誤部分驗證（FR 擁擠確認；心理機制為推論）。

**柱 2 [P1 微結構 — Taker Sell Ratio 分佈信號]（中）**  
Longs 被困但未主動清算時，Taker Sell Ratio 溫和上升 51–55%（緩慢分佈非爆量清算）。最近 3 個 4H bar 平均 sell ratio = 52.7%（門檻 50.5%）。

**柱 3 [P5 成本永續性 — FR 年化侵蝕]（強）**  
累積 FR 年化 > 10% 時高槓桿 longs 數學上無法持續。BTC 0.009121%/8H = 9.99%/yr；48H 價格波動 1.44%（停滯）。數學驗證完整。

**有效獨立柱：~2.5（P2+P5 共享「高 FR 持續」激活，算 1.5 票）**

---

## Layer 2 複審待辦

待 Track C 執行（Layer 2 設定：IS=1095, OOS=365 bars, 3 folds, 7bps+15bps）

重點檢查：
1. FRAGILE fold 7 — L2 更嚴格成本 + 更長 OOS 是否加重
2. SOLUSDT — 未在 L1 測試，L2 應補測
3. 信號頻率 — 2.7 次/年在 L2 15bps slippage 下成本結構是否可持續
4. 近期集中 — 2025H2–2026 表現較強，是否有 regime 依賴

---

## Layer 2 複審結果
**狀態**：L2_REJECTED ❌
**複審日期**：2026-06-30 10:00

### Layer 2 Harness 數據（IS=1095, OOS=365, 8 folds, 成本=22bps）

| 指標 | L2 結果 | 門檻 | 狀態 |
|------|---------|------|------|
| Mean OOS Sharpe (ann.) | **0.910** | > 1.8 | ❌ |
| Mean OIS ratio | **-1.103** | > 0.6 | ❌ |
| Mean Profit Factor | **1.110** | > 1.1 | ✅（barely） |
| Worst MDD | **-4.87%** | < 25% | ✅ |
| Overfit folds (SR<0) | **0/8** | ≤ 2/3 | ✅ |
| Trades/yr | **2.2** | ≥ 12 | ⚠️ LOW_N |

### Fold 明細（8 folds，OOS=365 bars each）

| Fold | OOS 期間 | IS_SR | OOS_SR | OIS | PF | MDD% | Trades |
|------|---------|-------|--------|-----|----|------|--------|
| 1 | 2024-12-29→2025-02-28 | -2.758 | nan | nan | nan | 0.00 | 0 |
| 2 | 2025-02-28→2025-04-30 | -2.758 | nan | nan | nan | 0.00 | 0 |
| 3 | 2025-04-30→2025-06-30 | -2.758 | 1.060 | -0.385 | 1.139 | -4.08 | 2 |
| 4 | 2025-06-30→2025-08-30 | 0.613 | 0.759 | 1.239 | 1.082 | -4.87 | 1 |
| 5 | 2025-08-30→2025-10-29 | 0.726 | nan | nan | nan | 0.00 | 0 |
| 6 | 2025-10-30→2025-12-29 | 0.726 | nan | nan | nan | 0.00 | 0 |
| 7 | 2025-12-29→2026-02-28 | 0.439 | nan | nan | nan | 0.00 | 0 |
| 8 | 2026-02-28→2026-04-30 | nan | nan | nan | nan | 0.00 | 0 |

**說明**：8 個 OOS fold 中僅 2 個產生交易（共 3 筆）。「無交易」fold 的 OOS_SR=nan 從均值計算中排除，Mean OOS SR 僅由 2 個有交易 fold 支撐，統計意義極低。

### 主管拒絕原因

1. **OOS Sharpe 嚴重不達標**：Mean OOS SR = 0.910，遠低於 L2 門檻 1.8。對比 L1 OOS SR = 1.817，L2/L1 降效比 = 50.1%（警戒線 70%），明顯降效。

2. **OIS ratio 失效（負值）**：Mean OIS = -1.103，因 IS_SR 在前 3 折均為 -2.758（大量 IS 期間零信號 + 成本拖累），分子/分母意義顛倒。負 OIS 代表「IS 越差，OOS 越好」，這是純雜訊特徵而非策略 edge。

3. **信號匱乏（最根本問題）**：8 個 OOS 窗口（共計約 2.9 年）中僅 3 筆交易，Trades/yr = 2.2。**策略本質上在大多數市場條件下不觸發**。Layer 1 能通過，是因為 18 個較短 fold 中碰巧捕捉到 2 個高強度交易窗口。Layer 2 使用更長 OOS 窗口（365 vs 180 bars），稀疏度問題完全暴露。

4. **Regime 集中（近期偏誤）**：L1 的亮眼折 17 OOS_SR=+2.05、折 18 OOS_SR=+9.41 均集中在 2025H2–2026。L2 更長時間跨度顯示這是局部 regime，而非普遍 edge。

5. **策略結構性缺陷**：fr_cum_z 使用 730-bar（約 5 個月）Z 基線，在 4H 時框下 BTC 結構性熊市（2024H2–2025Q1）期間，累積 FR 長期偏低，導致 IS_SR 為負且 OOS 零信號。策略只在 FR 正態且出現極端擁擠時才有效，這種市場條件極為罕見。

### 後續建議（供 Ken 參考）

- **根本問題**：LTSC v1.5 的三個進場條件同時滿足的概率在 BTC 4H 時框極低（約每年 2-3 次），且集中在特定 regime（FR 長期為正 + 擁擠環境）。這在統計上不具備足夠的樣本量。
- **方向一**：放寬一個條件（移除 Taker Sell Ratio 條件，或降低 fr_cum_z 門檻至 1.0），提升信號頻率至 ≥ 12 次/年再重測。
- **方向二**：改用月線或週線時框，OR 擴展至多幣種（同邏輯跑 ETH+BTC+SOL 合倉），人工提高信號池。
- **方向三**：SOLUSDT 雖 L1 未測，但鑑於 BTC 在 L2 已無統計力，建議先解決信號頻率問題再跨幣種擴展。
