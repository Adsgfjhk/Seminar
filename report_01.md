# 專題演講筆記：系統級診斷與容錯 (System-Level Diagnosis)

## 演講資訊
* **講題**：System-level Diagnosis - An Introduction and Recent Results (研究領域介紹：網路診斷與容錯)
* **講者**：Dajin Wang 王大進
* **單位**：School of Computing, Montclair State University, New Jersey, USA

---

## 講者學校簡介 (Montclair State University - Quick Facts)
* **創立年份**：1908 年
* **校園規模**：主校區約 250 英畝（$\approx 1\text{ km}^2$）
* **學院結構**：13 所學院 (Colleges/Schools)
* **學術項目**：超過 300 個主修、副修與專業領域
* **學生總數**：約 23,000+ 人
  * 大學部：約 18,000 人
  * 研究所：約 4,500 人

---

## 1. 互連網路拓撲結構範例 (Interconnection Network Examples)
大規模多處理器系統常採用特定的拓撲結構來連接各節點（處理器/計算單元）：

* **Hypercube（超立方體）**：以二進位編碼為頂點，各相鄰頂點漢明距離為 1（例：3-cube / 8 個節點）。
* **$k$-ary $n$-cube**：超立方體的推廣形式（圖示為 $Q_2^4$，$4$ 個節點維度的環狀/網格互連）。
* **Star Graph（星型圖）**：基於排列（Permutation）的對稱網路結構。
* **Bubble-sort Graph（冒泡排序圖）**：基於相鄰元素對換的頂點置換圖（圖示為 $B_4$）。

---

## 2. 研究背景與動機 (Background & Motivation)

### 規模擴張帶來的可靠性挑戰
* 越來越多高速、大規模的多處理器（Multiprocessor）與多電腦（Multicomputer）系統被部署與使用。
* 系統規模與日俱增，加上網路攻擊活動猖獗，計算節點 (Nodes) 的**故障/失效 (Faults/Faulty nodes) 不可避免**。

### 失效機率的數學現實 (可靠度崩跌)
在**無備援 (Non-redundant)** 的系統架構下，單一處理器的高可靠度無法保證大規模系統的整體可用性：
* **假設**：單一處理器的可靠度為 $99.99\%$（即 $0.9999$）
* **100 顆處理器系統**：
  $$R = 0.9999^{100} = 99.01\%$$
* **1,000 顆處理器系統**：
  $$R = 0.9999^{1000} = 90.48\%$$
* **10,000 顆處理器系統**：
  $$R = 0.9999^{10000} = 36.79\%$$
> **結論**：隨著節點數擴展到萬級以上，整個系統正常運行的機率將大幅降至 $36.79\%$，因此必須具備容錯與多節點故障診斷的能力。

---

## 3. PMC 系統級故障診斷模型 (Preparata-Metze-Chien Model)

> **經典文獻**：F.P. Preparata, G. Metze, R.T. Chien, *"On the connection assignment problem of diagnosable systems"*, IEEE Trans. Comput. 16(1967), 448-454.

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

> ⚠️ **關鍵難點**：若執行測試的節點本身已故障，則其產生的測試結果是**隨機/任意 (Arbitrary)** 的，完全不可信。

---

## 4. 症候群 (Syndrome) 與可區分性 (Distinguishability)

### 症候群表容量公式 (Syndrome Table Size)
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

## 5. 系統診斷度 ($t$-Diagnosability)

### 定義
* **$t$-Diagnosable（$t$-可診斷）**：若在全系統故障節點數**不超過 $t$ 個**的前提下，任意兩個相異的故障集合 $F_1, F_2$（滿足 $|F_1| \le t, |F_2| \le t$）皆為可區分 (Distinguishable)，Arbiter 能唯一且正確識別所有故障節點。
* **診斷度 (Diagnosability)**：系統所能保證正確診斷的最大允許故障節點數 $t$。

### 充分條件定理 (Sufficient Condition for $t$-Diagnosability)
對於一個網路圖 $G = (V, E)$，若要滿足 $t$-可診斷性，充分條件為：
1. **節點總數限制**：$|V| \ge 2t + 1$
2. **最小度數限制**：$\kappa(G) \ge t$ （其中 $\kappa(G)$ 為圖 $G$ 的最小度數，$\min\text{ degree of } G$）

### 超立方體的診斷度實例
* **$n$ 維超立方體 ($Q_n$)** 是 **$n$-diagnosable**（診斷度為 $n$）。
* 例如：3 維超立方體 $Q_3$（8 個節點，每點度數為 3）具備 **3-diagnosable** 特性。

---

## 6. 三角形網路 ($K_3$) 診斷實例與反例證明

考慮由節點 $a, b, c$ 構成的完全三節點圖（測試向量順序：$a\to b, b\to a, a\to c, c\to a, b\to c, c\to b$）。

### 1-Diagnosable (可診斷, $t=1$)
當故障節點最多為 1 個時，各故障情況的 Syndrome 集合彼此完全獨立無重疊：
* 全正常：`(0, 0, 0, 0, 0, 0)`
* 僅 $a$ 故障：`(x, 1, x, 1, 0, 0)`
* 僅 $b$ 故障：`(1, x, 0, 0, x, 1)`
* 僅 $c$ 故障：`(0, 0, 1, x, 1, x)`

### NOT 2-Diagnosable (不可診斷, $t \neq 2$) 反例證明
若允許故障節點數為 2，考慮以下兩種故障集合：
* 故障集合 $F_1 = \{a, b\}$（$c$ 正常）
* 故障集合 $F_2 = \{a, c\}$（$b$ 正常）

對照兩者的 Syndrome 表可發現存在**重疊輸出**：
* 在 $F_1 = \{a, b\}$ 中，因為 $a, b$ 皆故障，可能產生症候群：
  $$\mathbf{s} = (0, 1, 1, 1, 1, 1)$$
* 在 $F_2 = \{a, c\}$ 中，因為 $a, c$ 皆故障，同樣能產生症候群：
  $$\mathbf{s} = (0, 1, 1, 1, 1, 1)$$
* 在 $F_3 = \{b, c\}$ 中，亦會與其他組合產生如 `(1, 1, 1, 1, 0, 1)` 等重疊症候群。

> **證明結論**：當中央仲裁器觀察到 Syndrome 為 `(0, 1, 1, 1, 1, 1)` 時，無法判定到底是 $\{a, b\}$ 故障還是 $\{a, c\}$ 故障。因此 3-Cycle **並非 2-diagnosable**（故其診斷度 $t = 1$）。

---

## 7. 條件診斷度 (Conditional Diagnosability)

### 經典診斷度限制與問題
* 傳統診斷度 $t$ 對故障點的分佈**未做任何限制（允許無條件、任意分佈）**。
* 在極端情況下，一個正常節點的所有鄰居節點可能同時故障，導致該節點完全被孤立且失去所有可信測試來源（瓶頸往往取決於圖的最小度數）。

### 條件診斷度假設 (Conditional Assumption)
* **核心假設**：**任一節點的鄰居節點不可能同時全部故障 (cannot be all faulty simultaneously)**。
* **物理合理性**：在實際分散式系統中，某一節點的所有周邊鄰居在同一瞬間全部故障的機率極低。
* **效果**：排除此種極端情況後，大幅減少了不確定測試結果的干擾，使系統允許檢測出的故障節點數大幅提升（通常可提高數倍）。

### 超立方體下的條件診斷度定理
* **定理 (Lai et al., 2005)**：
  在 PMC 模型下，對於維度 $n \ge 5$ 的 $n$ 維超立方體 $Q_n$：
  * **傳統診斷度 (Unconditional)**：$t = n$
  * **條件診斷度 (Conditional)**：$t_c = \mathbf{4n - 7}$
* **對比**：以 $n=5$ 為例，傳統診斷度僅為 $5$，但在條件診斷度模型下可容忍高達 $4(5) - 7 = 13$ 個故障節點。
