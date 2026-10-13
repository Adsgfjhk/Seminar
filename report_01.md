# 專題演講報告

## 1. 演講基本資訊

* **日期**： `2026/10/06`
* **演講講者：** 卓世明 
* **演講題目：** AI 果園防猴系統

---

## 2. 演講簡介

這場演講針對台灣農業環境中長期存在之高複雜度非結構化場域問題——臺灣獼猴（*Macaca cyclopis*）造成之農損衝突，提出基於邊緣計算（Edge Computing）與閉迴路控制（Closed-Loop Control）之自主防護系統。傳統物理性防護（如電圍網、防猴圍網）建置維護成本高昂且受地形限制，而開迴路聲光驅逐刺激則因靈長類動物之高階認知學習能力，極易誘發神經適應與習慣化效應（Habituation Effect），導致長期防禦效能急遽衰退。

卓教授團隊展示了一套概念驗證 $\rightarrow$ 場域實戰 $\rightarrow$ 多模態空地立體協同的遞進式研發路徑，並為我們示範了研究方法論：
1. **論文一（PoC 最小可行閉環，Eng. Proc. 2025）：** 於受控環境（模擬果園與遙控猴偶）接通感知（IPCAM RTSP） $\rightarrow$ 決策（YOLOv5n 推論） $\rightarrow$ 座標校正（線性回歸） $\rightarrow$ 致動（雙軸步進馬達驅動 650 nm 雷射）的 38 ms 低延遲邊緣控制管線。
2. **論文二（落地實測與抗習慣化，CMC 2025 SCI Q2）：** 前進宜蘭頭城 5 公頃真實陡坡果園，部署 Jetson Orin Nano 邊緣系統，升級為 YOLOv12n 偵測模型、引進結合外觀特徵嵌入之 BoT-SORT 多目標追蹤，並建構以深度確定性策略梯度（DDPG）為基礎的 LoAH 5 自適應決策代理人，在長達 21 天之實地 field test 中維持 88% 之入侵頻率降低率（IFRR）。
3. **論文三（多模態雙防線，撰寫中）：** 針對野外濃密植被遮蔽與過曝等邊界案例（Edge Cases），設計 Raspberry Pi 5 搭載 CSI RGB 與 USB TIR 熱成像感測器之微秒級硬體時間戳對齊與單應性矩陣（Homography）空間融合架構，並協同空中自主巡弋四軸無人機（UAV），達成全天候三維立體防線。

---

## 3. 技術說明

本演講涵蓋之關鍵技術跨足電腦視覺、邊緣運算、機電控制、增強學習與多感測器融合：

### 3.1 影像物件偵測與三階段訓練架構（Object Detection & Semi-Supervised Pipeline）
* **輕量化偵測模型：** 系統從早期的 YOLOv5n 升級至最新的 **YOLOv12n**。在邊緣設備 Jetson Orin Nano（FP16）上具備高達 480 FPS、單張延遲僅 28.6 ms、整機平均功耗僅 12.5 W 的卓越性能，最終達到 0.923 的 mAP@0.5。
* **三階段訓練框架（解決少標註與災難性遺忘）：**
  * **Stage 1（高精度監督學習）：** 僅標註獼猴，將人類、貓狗與背景雜訊設為負樣本，人工標註量減少約 60%。
  * **Stage 2（半監督學習）：** 以 Stage 1 高精度（Precision 0.924）之教師模型針對未標註影像產生偽標籤（Pseudo-label），僅保留信賴度 $\ge 0.8$ 之高品質樣本，避免學生模型產生確認偏誤（Confirmation Bias）。
  * **Stage 3（增量微調）：** 導入雷射光點新類別，凍結骨幹網路前 10 層並採用低學習率微調偵測頭，保留舊特徵的同時達到 0.943 的光點 Recall。

### 3.2 多目標追蹤技術（Multi-Object Tracking, MOT）
* **BoT-SORT vs. ByteTrack：** 果園環境中猴群外觀極為相似且移動迅速、遮擋頻繁。相較於僅依賴運動預測的 ByteTrack，**BoT-SORT** 結合了深度外觀特徵 Re-ID，使同類個體間的餘弦距離保持在 0.4–0.5。
* **實測表現：** 在密集植被場景下，BoT-SORT 的軌跡中斷次數與 ID 交換（ID Switches）顯著低於 ByteTrack，軌跡平均壽命由 0.81 秒大幅提升至 1.88 秒，確保雷射能穩定追隨特定個體。

### 3.3 座標轉換與敏感度校正（Coordinate Calibration）
* **映射轉換模型：** 系統需即時將 2D 像素座標 $(x, y)$ 換算為雙軸伺服馬達/雲台的水平與垂直轉向角度 $(\text{pan}, \text{tilt})$。
  * 論文一採用一次多項式線性回歸擬合：
    $$\text{angle} = (-0.05074) \cdot x + 98.5273 \quad (\text{MSE} = 0.0813)$$
  * 論文二升級為二次多項式曲面校正，在全域均勻取樣（$N = 181$ 點）時，像素誤差降至 1.2 px。
* **敏感度分析：** 演講特別點出係數擾動測試（$\pm 5\%$ 係數變動使 RMSE 激增至 2.5–3.8 px），證明非線性校正項對系統命中的重要性。

### 3.4 強化學習抗習慣化策略（DDPG Reinforcement Learning）
* **習慣化困境：** 傳統固定掃描或單純隨機路徑在 21 天內驅離效果會雪崩式下滑（固定模式第 21 天降至 -15%，隨機模式降至 18%）。
* **DDPG 連續控制架構：** 系統以深度確定性策略梯度（Deep Deterministic Policy Gradient）演算法，觀察猴群軌跡、移動速度、離園距離與密度等連續狀態，輸出動態雷射角度、發射功率與閃爍頻率。
* **獎勵函數設計：**
  $$R = 1.0 \cdot R_{\text{驅離}} + 0.5 \cdot R_{\text{安全}} - 0.8 \cdot R_{\text{習慣化}}$$
  透過嚴厲懲罰固定重複的刺激路徑，迫使系統持續探索全新刺激組合（LoAH 5 等級），使 21 天實驗的入侵頻率降低率維持在 88% 至 91%。

### 3.5 雙光譜時空對齊與空地協同（Multi-Modal Fusion & Drone Surveillance）
* **時間同步（Temporal Alignment）：** 地面 Raspberry Pi 5 同時接入 30 fps CSI（可見光 RGB）與約 8.5 fps USB 3（熱成像 TIR），利用統一系統時間戳（Timestamp）、排除校正快門凍結（FFC）影格，達成最近幀配對時間誤差 $\le 16.7\text{ ms}$。
* **空間配準（Spatial Homography）：** 透過專用主動加熱棋盤格（Active Heating Checkerboard）校準，計算可見光與熱光譜的單應性矩陣（Homography Matrix）。
* **空地立體聯防：** 地面雙鏡頭藉由熱輪廓降低遮蔽漏判，藉由體溫特徵排除強光陰影誤判；空中自主無人機則補足視角死角，以高分貝仿猛禽喊話器或空中爆鳴進行動態驅趕。

---

## 4. 心得報告

本場演講不僅是一場結合邊緣運算、深層特徵追蹤與連續空間強化學習的研究，更為剛踏入學術研究領域的我示範了該如何去做研究的研究架構方法，卓教授團隊展現了如何將泥濘果園中的實體生態衝突抽象化為最小可行閉迴路（Minimal Viable Closed-Loop）系統——從第一篇論文運用猴偶、受控室內光照與 YOLOv5n 確立即時感知到致動的可行性，到第二篇論文再鎖定多目標干擾、光學幾何誤差與動物習慣化等遇到的問題，進一步升級為 YOLOv12n、BoT-SORT、DDPG 演算法並導入嚴謹的 RM-ANOVA 統計檢定，具體體現了上一篇論文的 Limitation 即為下一篇論文的 Contribution的學術思維，讓我深刻感覺到研究絕非盲目追逐參數量龐大的大模型，而是必須具備將實際工程缺點變成可被驗證的研究問題，並在硬體功耗、推論延遲與生物行為動態之間取得系統平衡。

---

## 5. 關鍵字
 
* `臺灣獼猴防護 (Macaque Crop-Depredation Deterrence)`
* `YOLOv12n 輕量化神經網路 (Lightweight Deep Learning)`
* `邊緣計算 (Edge Computing, Jetson Orin Nano)`
* `多目標追蹤 (Multi-Object Tracking, BoT-SORT)`
* `重識別特徵提取 (Re-Identification, Re-ID)`
* `深度確定性策略梯度 (DDPG, Deep Reinforcement Learning)`
* `抗習慣化決策機制 (Anti-Habituation Deterrence, LoAH 5)`
* `多光譜影像融合 (RGB-TIR Multi-Modal Fusion)`
* `單應性矩陣空間配準 (Homography Registration)`
* `時空錨定同步 (Spatial-Temporal Synchronization)`
* `空地立體協同防禦 (Autonomous Aerial-Ground Synergy)`

---

## 6. 參考文獻

1. **Cho, S.-M., Wang, S.-W., Chiu, M.-C., & Chen, S.-C.** (2025). AI-Powered Real-Time Object Detection and Laser Deterrence System for Orchard Protection Against Macaques. *Engineering Proceedings*.
2. **Cho, S.-M., Wang, S.-W., Chiu, M.-C., & Chen, S.-C.** (2025). Anti-Habituation Laser Deterrence System Combining Multi-Object Tracking and Deep Reinforcement Learning in Orchard Environments. *Computers, Materials & Continua (CMC)*, SCI Q2.
3. **Cho, S.-M., et al.** (In Preparation). Multi-Spectral Spatial-Temporal Image Fusion and Autonomous Aerial UAV Deterrence System for Agricultural Wildlife Management.
4. **Aharon, N., Orfaig, R., & Bobrovsky, B. Z.** (2022). BoT-SORT: Robust Associations Multi-Pedestrian Tracking. *arXiv preprint arXiv:2206.01256*.
5. **Lillicrap, T. P., Hunt, J. J., Pritzel, A., Heess, N., Erez, T., Tassa, Y., Silver, D., & Wierstra, D.** (2015). Continuous control with deep reinforcement learning. *arXiv preprint arXiv:1509.02971*.
6. **行政院農業部林業及自然保育署 / 林業試驗所** (2021). 野生動物危害農作物防治技術與田野防禦試驗報告.
