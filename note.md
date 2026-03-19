# Efficient Access Scheme for Multi-bank Based NTT Architecture Through Conflict Graph - Abstract

1. **背景**：數論轉換（NTT）硬件加速器是後量子加密（PQC）等多種加密系統中的關鍵組件。
2. **核心研究**：文章針對多銀行（multi-bank）NTT 架構，提出了關於構建**無衝突內存映射方案（CFMMS）**的新見解。
3. **主要貢獻**：
    * 提供了任意基數（arbitrary-radix）NTT 的並行循環結構，並提出了兩種**取點模式（point-fetching modes）**。
    * 將無衝突映射問題轉化為**衝突圖（conflict graph）**，並開發了一種新型啟發式演算法來探索 CFMMS 的設計空間。這種方法比傳統方案產生的訪問方案更高效。
4. **驗證結果**：為了驗證該方法，研究團隊為 Dilithium 算法設計了高性能的 NTT/INTT 核心。在類似的 FPGA 平台上，其面積-時間效率（area-time efficiency）顯著優於現有的先進研究成果。

- **關鍵字**：數論轉換（NTT）、任意基數（arbitrary radix）、多銀行（multi-bank）、無衝突內存映射方案（conflict-free memory mapping scheme）、Dilithium。

---

# Introduction - 簡介整理

### 1. NTT 的重要性與挑戰
* **應用廣泛**：關鍵組件於後量子密碼學（PQC）、同態加密（HE）及零知識證明。
* **核心功能**：將多項式從「係數表示」轉為「點值表示」，將計算複雜度從 $O(N^2)$ 降至 $O(N \log N)$。
* **面臨挑戰**：龐大的計算負擔限制了實際硬體實作，硬體加速器研究目前處於高峰。

### 2. 現有技術瓶頸 (研究動機)
* **追求高吞吐量**：主流技術使用多個內存庫（banks）搭配並行處理單元（PEs/BFUs）。
* **現有架構取捨**：
    * **Constant geometry NTT**：存取模式簡單但需要雙倍內存開銷。
    * **In-place NTT**：內存佔用小，但隨階段變化的跨度（strides）使高基數（higher radix）平行化變得極其困難。
* **現有 CFMMS 缺陷**：
    * 奇數長度向量仍有衝突、查找表/緩衝區佔用過高、或因資料同步導致管線停滯。
* **結論**：目前缺乏能同時支援「任意基數 In-place NTT」且能「多單元並行」的高效無衝突映射方案。

### 3. 本論文的主要貢獻
1. **提出新結構與模式**：定義了通用任意基數並行結構，並根據資料依賴性提出兩種取點模式：
    * **IEP (Inter-group Prioritization)**：組間優先。
    * **IAP (Intra-group Prioritization)**：組內優先。
2. **開發映射演算法**：
    * **IEP 模式**：修改線性轉換技術。
    * **IAP 模式**：創新將無衝突映射轉化為**「衝突圖 (Conflict Graph)」**，並使用啟發式演算法探索更佳方案。
3. **實體驗證**：
    * 為 Dilithium 設計了完全管線化 NTT/INTT 核心。
    * 面積-時間效率（Area-time efficiency）顯著優於現有先進研究。
    * 發現 IAP 模式比 IEP 模式消耗更少的 LUT 開銷。

---

# 關鍵概念補充

### 1. 記憶體交錯 (Memory Interleaving)
* **定義**：將連續的資料分散存儲在多個獨立記憶體庫（Memory Banks）的技術。
* **目的**：提升存取頻寬。讓多個硬體處理單元（PEs）能同時讀寫不同 Bank 的資料，避免排隊等待。
* **在 NTT 中的問題**：由於 NTT 的存取跨度（Strides）會隨階段改變，簡單的交錯規律容易導致多個單元同時存取同一個 Bank，造成「存取衝突」。

### 2. 無衝突記憶體映射機制 (CFMMS)
* **定義**：一種決定資料在 Bank 中存放位置的「規則」或「映射函數」。
* **核心任務**：保證在任何並行運算週期中，所有需要同時被讀寫的資料點都位在**不同的實體 Bank** 中。
* **優化重點**：
    * **無衝突性**：必須在 NTT 的所有階段（Stages）都維持無衝突。
    * **硬體效率**：映射電路必須足夠簡單（如使用簡單的 XOR 邏輯），以減少延遲與面積佔用。
    * **靈活性**：本論文特別強調要能支援任意基數（Arbitrary Radix）的 NTT 架構。

---

# Background - 論文背景深度分析

### 1. NTT 算法基礎與原位運算 (In-place Property)
* **核心概念**：在晶格後量子密碼 (Lattice-based PQC) 中，NTT 是核心組件。論文特別強調了 **In-place NTT** 的特性：計算結果會直接存回原始操作數所在的內存位置。
* **優勢與挑戰**：
    * **記憶體效率**：In-place 的最大優勢是節省空間。對於 $N$ 點的向量，僅需要 $N$ 個字的存儲空間，這在硬體資源受限的 FPGA/ASIC 上至關重要。
    * **並行挑戰**：雖然節省空間，但因為數據不斷被覆蓋，且每一階段（Stage）存取數據的「跨度」（Strides）都在變化，這使得設計一個固定的高速並行電路變得非常複雜。如果映射不好，並行單元就會為了搶奪同一個記憶體庫（Bank）而排隊。

### 2. 記憶體交錯與多銀行架構 (Memory Interleaving & Multi-bank)
* **核心概念**：為了匹配處理單元（PEs/BFUs）的頻寬，系統採用多個獨立的 Bank。理想情況下，若有 $d$ 個並行單元，每個週期應能同時從不同 Bank 讀取 $d$ 個點。
* **吞吐量匹配**：這是硬體加速的基本邏輯。如果計算單元很快但記憶體存取跟不上，系統就會發生管線停滯（Pipeline stalls）。
* **衝突定義**：當兩個或多個處理單元在同一時刻試圖訪問同一個實體 Bank 時，就會發生「記憶體衝突」（Memory Conflict）。例如，在基數為 2（Radix-2）的 8 點 NTT 中，第 2 階段可能會出現多個數據點映射到同一個 Bank 的情況。

### 3. 跨度訪問模式與地址映射 (Strided Access Scheme)
* **核心概念**：論文將訪問方案定義為一個映射函數，將 $n$ 位的邏輯地址映射為 $m$ 位的 **Bank Index (BI)** 和 $t$ 位的 **Row Address (RA)**。
* **地址拆解**：這是硬體實現的底層邏輯。邏輯地址（數據在數學序列中的序號）必須轉換為實體的物理位置（在哪個 Bank，哪一行）。
* **映射函數的關鍵性**：無衝突內存映射方案 (CFMMS) 的目標就是設計一個函數，使得在任何給定的計算步驟中，所有並行需要的邏輯地址映射出來的 $BI$ 都是互不相同的。

### 4. 現有映射方案的演進與缺陷
論文回顧了三種主流方案：
1. **Low-order Interleaving**：直接取地址低位。雖然簡單，但在有跨度（Stride）的訪問中表現極差。
2. **Row Rotation (Skewed Scheme)**：通過循環移位來分散衝突。雖然能支持常數跨度，但在處理多維訪問或變動跨度時不夠靈活，且硬體實現上可能引入加法器延遲。
3. **Linear Transformation Technique (LTT)**：使用線性矩陣轉換（通常是 XOR 邏輯）來生成地址。雖然延遲極低，但在面對「任意基數」（Arbitrary Radix）和更複雜的並行模式時，依然難以找到最優解。

* **總結**：現有解決方案在靈活性或效率上存在瓶頸，這正是本篇論文提出「衝突圖（Conflict Graph）」法，以尋找更優映射規律的動機。

---

# NTT 補充概念：CT、DIT 與 DIF 差異解析

在深入了解 NTT 硬體架構前，必須釐清 Cooley-Tukey (CT)、DIT 與 DIF 關係。**Cooley-Tukey (CT)** 是一種利用分治法加速離散傅立葉轉換的演算法大類，而 **DIT** 和 **DIF** 是 CT 演算法的兩種最常見具體實現方式。本論文主要使用的是 **DIT NTT**。

### 1. 什麼是 NTT (數論轉換)?
* **本質**：NTT 是「在有限整數環 (Finite Field) 上執行的 FFT」。所有加、減、乘法最後都會對一個特定的質數 $q$ 進行取模運算 (Modulo $q$)。
* **優勢**：因為全是整數運算，絕對精確、無捨入誤差。能將多項式相乘的計算複雜度從 $O(N^2)$ 降到 $O(N \log N)$，是後量子密碼學的硬體加速核心。

### 2. DIT (時間抽取法) 與 DIF (頻率抽取法) 的核心差異
兩者都使用「蝴蝶運算 (Butterfly Operation)」將大問題拆成小問題，但「拆解」和「計算」順序相反。

#### A. DIT (Decimation-in-Time，時間抽取法) - *本論文採用的方式*
* **分解邏輯**：按輸入索引的「奇/偶」拆分。
* **教科書定義 (1965 Cooley-Tukey 原始版)**：在最原始、最經典的原地運算（In-place）DIT 中，因為精神是「物理分家」（將陣列按奇/偶數位置拆開），為了在同一塊記憶體做到這件事，演算法的第一步就是強制把輸入資料的位址做「位元反轉（Bit-reversal）」。因此，傳統教科書會說 DIT 是 **「亂序進 (Bit-reversed)、正序出 (Natural)」**。這正是本論文引用的傳統定義。
* **本論文的改良 (現代常見實作)**：作者為了避免位元反轉的巨大硬體開銷，透過調整迴圈結構與旋轉因子，將演算法改良為我們現在硬體實作中常見的 **「正序進 (Natural Order)、亂序出 (Bit-reversed Order)」**。
* **蝴蝶運算結構**：與旋轉因子 ($\omega$) 的乘法，發生在加/減法「之前」。

#### B. DIF (Decimation-in-Frequency，頻率抽取法)
* **分解邏輯**：按陣列的「前半/後半」拆分。
* **順序特徵**：傳統上允許**輸入數據**是**自然順序 (Natural order)**；計算完後，**輸出數據**變成**位元反轉 (Bit-reversed order)**。
* **蝴蝶運算結構**：與旋轉因子 ($\omega$) 的乘法，發生在加/減法「之後」（通常作用於減法分支）。

### 3. 綜合比較與硬體考量

| 比較項目 | DIT (Decimation-in-Time) | DIF (Decimation-in-Frequency) |
| :--- | :--- | :--- |
| **陣列拆分方式** | 按索引的 **奇/偶 (Even/Odd)** 拆分 | 按陣列的 **前半/後半 (Top/Bottom)** 拆分 |
| **標準輸入順序** | 位元反轉 (Bit-reversed) | 自然順序 (Natural) |
| **標準輸出順序** | 自然順序 (Natural) | 位元反轉 (Bit-reversed) |
| **旋轉因子($\omega$)乘法位置**| 在加/減法 **之前** | 在加/減法 **之後** |

**💡 結合論文觀點的硬體設計考量：**
在硬體設計中，資料搬移（如位元反轉排序）非常耗時且耗資源。雖然標準的 DIT 要求輸入必須是「位元反轉」順序，但本論文的作者希望透過巧妙設計迴圈結構與**無衝突記憶體映射 (CFMMS)**，來達到在硬體上「不需要事先做位元反轉排序 (bit-reversed-free)」就能直接做 DIT NTT 計算，進而壓榨出更高的效能與節省面積。這也是他們提出「衝突圖演算法」的關鍵挑戰之一。

---

# Section 3: GENERAL PARALLEL IN-PLACE NTT 深度架構分析

本節建立了一個通用的硬體加速框架，使其能支援「任意基數 (Radix-$R$)」且具備「平行度 $d$」的 In-place NTT 運算。

### 1. 數學基石：分治法與並行化的起源 (Equations 1 & 2)
* **Equation (1) - 基數-R 的初步拆解 (Decimation)**：
  $$A_i = \sum_{j=0}^{N/R-1} a_{Rj} \phi_{2N}^{Rj \cdot i} + \omega_N^{1 \cdot i} \sum_{j=0}^{N/R-1} a_{Rj+1} \phi_{2N}^{Rj \cdot i} + \dots$$
  * **分析**：此公式將輸入序列 $a$ 根據索引對 $R$ 取模的結果拆分為 $R$ 個子序列。
  * **硬體意義**：這是 **Multi-bank 架構** 的理論基礎。為了讓 $R$點蝴蝶運算能在一週期內完成，這 $R$ 個子序列（$a_{Rj}, a_{Rj+1}, \dots$）理想上應存放在不同的實體 Bank 中。

* **Equation (2) - 索引拆解與 Elimination Property**：
  透過定義 $i = a_H \cdot \frac{N}{R} + a_L$，將索引拆解為高位與低位。
  * **分析**：作者證明了括號內的子 NTT 運算僅與低位索引 $a_L$ 有關。
  * **硬體意義**：這簡化了 **地址產生器 (AGU)** 的設計。因為不同子序列的存取模式在數學上是同步的，硬體只需要產生一組基礎地址，再透過固定的偏移量（Offset）即可同時存取 $R$ 個點。

### 2. 遞迴與跨度：衝突的根本來源 (Equation 3 & Figure 3)
* **Equation (3) - 遞迴形式與 In-place 特性**：
  $$A_{i + m \cdot \frac{N}{R}} = \sum_{r=0}^{R-1} \omega_N^{r(i+m \frac{N}{R})} \left( \sum_{j=0}^{N/R-1} a_{Rj+r} \phi_{2N/R}^{j i} \right)$$
  * **分析**：此公式定義了第 $p$ 階段的結果如何直接產生第 $p+1$ 階段的輸入。
  * **關鍵項 $m \cdot \frac{N}{R}$**：這直接定義了 **「存取跨度 (Stride)」**。跨度隨階段變化的特性，導致了記憶體衝突的發生。

* **Figure 3: Dataflow 視覺化驗證**：
  觀察 8 點 Radix-2 NTT 的資料流：
  * **第一階段 (Stage 1)**：跨度為 $N/2 = 4$。蝴蝶運算存取 $\{a_0, a_4\}, \{a_1, a_5\} \dots$。
  * **第二階段 (Stage 2)**：跨度縮小為 $N/4 = 2$。蝴蝶運算存取 $\{a_0, a_2\}, \{a_1, a_3\} \dots$。
  * **結論**：Equation (3) 中隨階段變化的跨度，導致了在固定 Bank 映射下，某些階段會發生多個 PE 爭奪同一個 Bank 的情況。

### 3. 並行展開與取點模式 (Algorithm 2 & 3)
* **IAP 模式 (Intra-group Prioritization - 組內優先)**：
  * **邏輯**：固定 Equation (3) 中的 Group 索引，優先在組內連續取出 $d$ 組蝴蝶運算所需的資料。
  * **特性**：雖然存取規律隨階段變化的劇烈程度較高（導致衝突難解），但其位址映射電路 (Address Mapper) 的硬體實作最為精簡。
  * **地位**：這是本論文透過「衝突圖」試圖優化的核心對象。

* **IEP 模式 (Inter-group Prioritization - 組間優先)**：
  * **邏輯**：同時從 $d$ 個不同的 Groups 中各取出一組蝴蝶運算的點。
  * **特性**：存取模式較接近線性，可用傳統線性轉換 (LTT) 處理，但處理跨組資料的硬體開銷 (LUT) 較大。

### 4. 映射函數的定義與挑戰
最後，本節將上述數學邏輯轉化為硬體實體位置的映射：
1. **邏輯位址 (Logic Address $a$)**：由 Equation (3) 產生的索引。
2. **映射函數 $f(a) \to (BI, RA)$**：
   * **$BI$ (Bank Index)**：資料在哪個 Bank。
   * **$RA$ (Row Address)**：資料在 Bank 的哪一行。
3. **目標**：確保在任何平行取點週期中，所有被選中的 $R \times d$ 個點，其映射出的 $BI$ 互不相同。

---

# Section 4: CONFLICT-FREE MEMORY MAPPING 深度拆解分析

本節核心任務是設計一個映射函數，將 $n$ 位元的**邏輯位址 $a$** 映射到實體的 **Bank Index ($BI$)** 與 **Row Address ($RA$)**。

### 1. IEP 模式下的映射方案 (Equation 4 & Figure 4, 5)
在 IEP (組間優先) 模式下，存取模式較為規律。作者採用了線性轉換技術 (LTT) 的擴充版。

* **Equation (4) - 線性轉換映射架構**：
  $$RA = R \times a^{\intercal}$$
  $$BI_i = T_{N,M} \times a^{\intercal} \oplus h_{n,m}(i)$$
  * **$R$ (Row Matrix)**：$(n-m) \times n$ 矩陣，決定 Bank 內的行位址。
  * **$T_{N,M}$ (Transformation Matrix)**：$m \times n$ 矩陣，決定進入哪個 Bank。
  * **$h_{n,m}(i)$ (Offset Function)**：針對並行度 $i$ 引入的偏移量，確保並行存取不衝突。
* **硬體實現 (Figure 5)**：由 **XOR Array** (實現矩陣乘法) 和 **Left Rotation** 單元組成的可配置映射器。優點是穩健，缺點是對於任意基數的擴充性會導致關鍵路徑延遲增加。

### 2. IAP 模式下的核心創新：衝突圖演算法 (Algorithm 4, Equation 5, Figure 6, 7)
這是論文的核心貢獻。對於存取規律雜亂的 IAP 模式，作者改用**圖論**解決。

* **衝突圖構建 (Figure 6)**：
  * **頂點 (Vertex)**：代表邏輯位址點。
  * **邊 (Edge)**：代表兩個點在同週期內需被同時存取，兩者之間連線代表衝突風險。
* **啟發式搜尋演算法 (Algorithm 4)**：
  1. **Step 1: Coresidual Check (餘項檢查)**：識別位址中具備天然無衝突潛力的位元特徵（Toggling bits）。
  2. **Step 2: Performing Permutation (執行排列)**：將存取模式劃分為 $R \times d$ 個群組並進行置換。
  3. **Step 3: Developing Group Conflict Graph (Figure 7)**：針對群組建立圖形，將搜尋空間從 $N$ 縮小到與並行度 $d$ 相關。
* **最終映射函數 (Equation 5)**：
  $$BI_l = r_l \oplus (\bigoplus_{k=0}^{u} a_{k \cdot m + l} \oplus a_{k \cdot m + l + w})$$
  * **精妙之處**：將複雜的圖論問題轉化為極簡的 **XOR 縮減 (XOR reduction)** 電路，不再需要複雜的矩陣乘法。

### 3. 映射器的硬體模型與優勢 (Figure 8 & 9)
* **硬體模型 (Figure 8)**：展示了 IAP 模式下的模組化設計（Modular Accumulation Unit 和 PM 置換單元），使用 MUX 取代複雜解碼器。
* **效能優勢 (Figure 9)**：證明了 IAP 模式在 LUT 開銷上遠低於 IEP，特別是在大並行度 $d$ 下，擴展性極佳，且顯著降低了硬體面積。

### 4. 針對 Dilithium 的案例研究與優化 (Algorithm 5)
* **Algorithm 5 (Improved Barrett Reduction)**：針對 Dilithium 的特定質數 $q=8380417$ 進行優化。透過硬體友善的位移 (Shift) 與加法 (Addition) 取代昂貴的 23 位元乘法，大幅提升運算核心的速度並節省 DSP 資源。
