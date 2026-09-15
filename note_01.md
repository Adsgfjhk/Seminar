# 專題演講筆記：系統級診斷與容錯 (System-Level Diagnosis)

## 演講資訊
* **講題**：System-level Diagnosis - An Introduction and Recent Results (研究領域介紹：網路診斷與容錯)
* **講者**：Dajin Wang 王大進
* **日期**：2026/09/15

---

## 1. Interconnection Network Examples
大規模多處理器系統常採用特定的拓撲結構來連接各節點（處理器/計算單元）：

* **Hypercube（超立方體）**：以二進位編碼為頂點，各相鄰頂點距離為 1。
* **$k$-ary $n$-cube**：超立方體的推廣形式。
* **Star Graph（星型圖）**：基於排列（Permutation）的對稱網路結構。
* **Bubble-sort Graph（冒泡排序圖）**：基於相鄰元素對換的頂點置換圖。

---

## 2. 研究背景與動機 (Background & Motivation)

### 規模擴張帶來的挑戰
* 越來越多高速、大規模的Multiprocessor與Multicomputer系統被使用。
* 系統規模與日俱增，加上網路攻擊活動頻繁，計算節點 (Nodes) 的**故障/失效 (Faults/Faulty nodes) 不可避免**。

### 失效機率的數學現實 (可靠度崩跌)
在**無備援 (Non-redundant)** 的系統架構下，單一處理器的高可靠度無法保證大規模系統的整體可用性：
* **假設**：單一處理器的可靠度為 $99.99\%$（即 $0.9999$）
* **100 顆處理器系統**：
  $$R = 0.9999^{100} = 99.01\%$$
* **1,000 顆處理器系統**：
  $$R = 0.9999^{1000} = 90.48\%$$
* **10,000 顆處理器系統**：
  $$R = 0.9999^{10000} = 36.79\%$$

---

## 3. Preparata-Metze-Chien Model


### 核心運作機制
* **互測機制**：節點對鄰近節點互相測試 (Mutually check status)。
* **中央仲裁器 (Centralized Arbiter)**：收集全系統的所有測試結果向量（稱為 **Syndrome，症候群**），並由 Arbiter 解讀該 Syndrome，判定各節點究竟為正常 (Fault-free) 還是故障 (Faulty)。

### 測試結果規則 (Test Outcomes)
設節點 $i$ 測試相鄰節點 $j$，記為 $a_{ij}$：

| 測試者 $i$ (Tester) | 被測者 $j$ (Tested) | 測試結果 $a_{ij}$ | 說明 |
| :--- | :--- | :---: | :--- |
| **正常 (Fault-free)** | **正常 (Fault-free)** | `0` | 結果可靠，判定正常 |
| **正常 (Fault-free)** | **故障 (Faulty)** | `1` | 結果可靠，判定故障 |
| **故障 (Faulty)** | **正常 (Fault-free)** | `x` (`0` 或 `1`) | **結果任意、不可靠** |
| **故障 (Faulty)** | **故障 (Faulty)** | `x` (`0` 或 `1`) | **結果任意、不可靠** |


---

## 4. Syndrome 與 Distinguishability

### Syndrome Table Size 
若給定一組故障節點集合 $F_1 = \{f_{11}, f_{12}, \dots, f_{1x}\}$：
* 故障節點發出的所有測試結果均有 $2$ 種可能（$0$ 或 $1$）。
* 該故障集合所能產生的可能 Syndrome 總數（Table Size）為：
  $$\text{Syndrome Table Size}(F_1) = 2^{\deg(f_{11}) + \deg(f_{12}) + \dots + \deg(f_{1x})}$$
  *(其中 $\deg(v)$ 為頂點 $v$ 發出的測試邊數/度數)*

### 可區分性 (Distinguishable)
* 設有兩組不同的潛在故障集合 $F_1$ 與 $F_2$。
* **定義**：若 $F_1$ 的 Syndrome 表與 $F_2$ 的 Syndrome 表**沒有交集 (Do not overlap)**，則稱 $F_1$ 與 $F_2$ 是**可區分的 (Distinguishable)**。
* 若兩集合的 Syndrome 發生重疊，則中央仲裁器看到該特定測試輸出時，將無法判定到底是 $F_1$ 還是 $F_2$ 發生故障。

---

## 5. t-Diagnosability診斷度

### 定義
* **$t$-Diagnosable**：若在全系統故障節點數**不超過 $t$ 個**的前提下，任意兩個相異的故障集合 $F_1, F_2$（滿足 $|F_1| \le t, |F_2| \le t$）皆為可區分 (Distinguishable)，Arbiter 能唯一且正確識別所有故障節點。
* **診斷度 (Diagnosability)**：系統所能保證正確診斷的最大允許故障節點數 $t$。

### 超立方體的診斷度實例
* **$n$ 維超立方體 ($Q_n$)** 是 **$n$-diagnosable**（診斷度為 $n$）。
* 例如：3 維超立方體 $Q_3$（8 個節點，每點度數為 3）具備 **3-diagnosable** 特性。

---

## 6. 條件診斷度 (Conditional Diagnosability)

### 經典診斷度限制與問題
* 傳統診斷度 $t$ 對故障點的分佈**未做任何限制（允許無條件、任意分佈）**。
* 在極端情況下，一個正常節點的所有鄰居節點可能同時故障，導致該節點完全被孤立且失去所有可信測試來源（瓶頸往往取決於圖的最小度數）。

### 條件診斷度假設 (Conditional Assumption)
* **核心假設**：**任一節點的鄰居節點不可能同時全部故障 (cannot be all faulty simultaneously)**。
* **物理合理性**：在實際分散式系統中，某一節點的所有周邊鄰居在同一瞬間全部故障的機率極低。
* **效果**：排除此種極端情況後，大幅減少了不確定測試結果的干擾，使系統允許檢測出的故障節點數大幅提升（通常可提高數倍）。
