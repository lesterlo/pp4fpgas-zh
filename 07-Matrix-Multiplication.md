# 第七章 矩陣乘法

本章節將討論稍複雜的設計—矩陣乘法。我們考慮兩種不同的版本。開始，我們用直接的方式實現，將兩個矩陣作為輸入，輸出結果是它們的乘積。我們稱這整體矩陣乘法。接下來，我們實現塊矩陣乘法。這裏，輸入到函數的矩陣被分塊，函數計算分塊的結果。

## 7.1 背景

矩陣乘法是一個二元運算，兩個矩陣計算結果為一個新的矩陣。矩陣乘法本身是採用構成矩陣的向量，進行的線性運算。最常見矩陣乘法稱為**矩陣積**。當矩陣**A**維度是n × m，矩陣**B**維度是m × p，那麼矩陣積**AB**則是一個n × p維的矩陣。

更準確的描述，我們這樣定義：

$$
\textbf{A} = \left[
\begin{matrix}
 A_{11}      & A_{12}      & \cdots & A_{1m}      \\
 A_{21}      & A_{22}      & \cdots &A_{2m}      \\
 \vdots & \vdots & \ddots & \vdots \\
 A_{n1}      &A_{n2}      & \cdots & A_{nm}     \\
\end{matrix}
\right]

\textbf{B} = \left[
\begin{matrix}
 B_{11}      & B_{12}      & \cdots &B_{1p}      \\
 B_{21}      & B_{22}      & \cdots &B_{2p}      \\
 \vdots & \vdots & \ddots & \vdots \\
 B_{m1}      &B_{m2}      & \cdots & B_{mp}     \\
\end{matrix}
\right]


\qquad(7.1)
$$

$$
\textbf{AB} = \left[
\begin{matrix}
 \textbf{(AB)}_{11}      &  \textbf{(AB)}_{12}      & \cdots & \textbf{(AB)}_{1p}       \\
 \textbf{(AB)}_{21}      &  \textbf{(AB)}_{22}      & \cdots & \textbf{(AB)}_{2p}   \\
 \vdots & \vdots & \ddots & \vdots \\
 \textbf{(AB)}_{n1}      &  \textbf{(AB)}_{n2}      & \cdots & \textbf{(AB)}_{np}  \\
\end{matrix}
\right]

\qquad(7.2)
$$


其中$$\textbf{AB}_{ij}$$ 運算定義為：$$\textbf{AB}_{ij} = \sum_{k=1}^mA_{ik} * B_{kj}$$

現在，我們看一個簡單的例子：

$$
\textbf{A} = \left[
\begin{matrix}
 A_{11}      & A_{12}     & A_{13}      \\
 A_{21}      & A_{22}      &A_{23}      \\
\end{matrix}
\right]

\textbf{B} = \left[
\begin{matrix}
 B_{11}      & B_{12}      \\
 B_{21}      & B_{22}      \\
 B_{31}      &B_{32}       \\
\end{matrix}
\right]
\qquad(7.3)
$$

這個矩陣乘法積結果是

$$
\textbf{AB} = \left[
\begin{matrix}
 A_{11}B_{11}+A_{12}B_{21}+A_{13}B_{31}      & A_{11}B_{12}+A_{12}B_{22}+A_{13}B_{32}      \\
 A_{21}B_{21}+A_{22}B_{21}+A_{23}B_{31}      & A_{21}B_{12}+A_{22}B_{22}+A_{22}B_{32}      \\
\end{matrix}
\right]
\qquad(7.4)
$$


```c
void matrixmul(int A[N][M], int B[M][P], int AB[N][P]) {
  #pragma HLS ARRAY_RESHAPE variable=A complete dim=2
  #pragma HLS ARRAY_RESHAPE variable=B complete dim=1
  /* for each row and column of AB */
  row: for(int i = 0; i < N; ++i) {
    col: for(int j = 0; j < P; ++j) {
         #pragma HLS PIPELINE II=1
         /* compute (AB)i,j */
         int ABij = 0;
         product: for(int k = 0; k < M; ++k) {
             ABij += A[i][k] * B[k][j];
      }
      AB[i][j] = ABij;
    }
  }
}

```

![圖7.1 一個通用的3層for循環實現了矩陣乘法。](images/placeholder.png)

最外層的**for**，標籤**rows** 和**cols**，通過迭代實現輸出矩陣**AB**的行和列的遍歷。最內層的循環，標籤**product**完成**A**矩陣對應行與矩陣**B**對應列的乘和累加，最後的結果是矩陣**AB**中對應位置的一個元素。

矩陣乘法是數值運算的基礎。計算大型矩陣之間的乘積會消耗大量的時間。因此，它在數值計算中一個非常重要的問題。根本上講， 矩陣運算是向量空間上的線性運算；矩陣乘法提供了一種線性變換的方式。這些應用有線性座標變換（例如，變換，旋轉在圖形領域），統計物理中的高維矩陣（例如，矩陣論）和拓撲操作（例如，確定從一個定點到另一個定點，路徑是否存在）。所以，它是一個值得被學習的問題，同時，有許多關於他的算法。這些算法目的是為了提高性能，減少算法對存儲的需求。

## 7.2 計算矩陣乘法

我們從最常用的方法用於計算矩陣乘法—三層嵌套**for**循環開始，開始計算過程的優化。圖7.1所示的代碼實現了這種運算。最外層的**for**循環，標籤為**rows**和**cols**，通過迭遍歷了數據矩陣**AB**的行和列。最內層的**for**循環計算了向量**A**一行與向量**B**一列的點積運算。每個點積運算通過一些列的計算得到的結果只是矩陣**AB**中的一個元素。從概念上説，我們執行了**P**矩陣向量乘法，每個對應矩陣**B**的一列。

在這個例子中，我們對**col**循環採用了**pipeline** directive，並且設定期望人任務間隔是1。這樣，最內層的**for**循環的完全展開的，我們期望的電路大概要執行M次乘累加操作並且執行的間隔是N × P個時鐘週期。正如第四章中討論到的，這是最合理的方式。我們選擇在函數不同的地方使用**pipeline** directive來實現在面積和速度之間的平衡。例如，當我們把相同的directive放在函數的最頂層（所有的**for**循環之外），那麼所有的循環都會被完全展開，那麼會有N × M × P次乘累加運算單元且運行間隔會是1。如果把directive放在內層**row** 循環，那麼會有M × P次乘累加單元並且執行間隔是N時鐘週期。通過對相應的矩陣進行分塊，上述設計要求和可以很容易滿足。當然也可對最內層循環實行流水線，那麼設計中只有個乘累加單元但是如果要達到 II =1且運行時鐘頻率很高是不可能的，因為變量$$\textbf{AB}_{ij}$$累加是一個遞歸。我們也可以對不同的循環進行部分展開，來實現其他的設計要求。最根本的平衡還是在設計結構的複雜度例如乘累加單元的數目和性能例如硬件執行的效率。最理想的方式，當資源翻倍以後，對應所用的時鐘週期減半，儘管這在實際中是很難實現的。

{% hint style='tip' %}
改變**pipeline** directive的位置。位置對資源的使用有什麼影響？對性能的改變有什麼作用？從函數間隔的角度考慮，那種選擇能實現最好的性能？那種能提供最小的資源使用率？你認為在哪裏放置directive最佳？如果矩陣的維度發生了變化，是不是你之前的選擇也要變化？
{% endhint %}

每個時鐘週期執行大量的運算，就需要為它們提供運算的對象，保存每個運算出處的結果。之前，我們使用了**array_partition**directive來提高在內存上可訪問的次數。當數組的分塊越來越小，每次存儲的訪問順序在編譯時就可以確定。所以數組分塊是最簡單和高效的方法用於提升在單位時間內，對內存的訪問。這個directive不僅僅是將存儲存儲空間的地址分為不同的區間，也可以將多個存儲空間合併以一個。這種變化存儲空間數據位寬變大用於保存數組，但是不能改變總存儲的比特數目。這其中的區別如圖7.2所示。

**array_reshape**和**array_partion** 都提高了一個時鐘週期內可以讀取的數組元素個數。它們支持相同的操作，可以循環和阻塞分塊或者根據多維度的數據進行不同維度的分塊。在使用**array_reshape** 的時候，所有的元素在變換後的數組中共用同一個地址，但是**array_partition** 變換後數組中地址是不相關的。看起來人們更喜歡使用**array_partition**因為它更靈活，每個獨立的小空間更小，可能會導致存儲資源使用不充分。**array_reshape** directive 會形成大的存儲塊，這樣可能更容易高效地映射到**FPGA** 的資源上。特別是Xilinx Virtex Ultrascale+器件最小的block RAM（**BRAM**）是18 Kbits，可以支持不同深度和寬度的組合。當數組分塊小於18kbits後，那麼**BRAM** 就沒有被充分使用。如果我們使用最原始的矩陣，4-bit數組，維度是 [1024][4] ,這個數組就可放到一個單獨的**BRAM** 中，它配置為4Kbit x 4。對這個數組的第二個維度進行分塊，那麼將使用4個 1Kbit x 4的存儲單元，每個都要比**BRAM**小很多。通過使用**array_reshape** 將數組變為1Kbit x 16，可以被一個**BRAM** 存儲。

![圖 7.2:三種實現二維矩陣的方式。左邊採用原始的方式，保存N*M個元素。中間數組使用了array_partition directive進行了變形，結果是使用了M個存儲單元，每個保存N個元素。右邊的，使用array_shape directive進行了變形，結果保存在一個存儲單元，帶有N個入口，每個入口保存原始矩陣的M個元素](images/matmul_array_reshape.jpg)

{% hint style='tips' %}

注意這裏的分塊是有效的因為分塊的維度（**A** 的列和 **B** 的行）是通過常量訪問的。另外，不可分塊的維度是通過相同變量訪問的。當對多維矩陣進行分塊是，這是一條很用用的經驗用於確定哪個維度可以進行分塊。

{% endhint %}

{% hint style='info' %}
去掉**array_reshape** directive。這對性能有什麼影響？這對資源使用率有什麼影響？通過改變**array_reshape**的參數，對這些數組有沒有影響？這種情況下，和只使用**array_reshape**有什麼區別？
{% endhint %}

矩陣的形狀可能對你進行的優化有潛在的影響。有些應用可能使用很小的矩陣，例如2x2 或 4x4。在這種情況下，可能期望設計實現最高的性能，通過在函數的入口直接使用**pipeline** directive。當矩陣的維度變大，例如32 x 32時，這種方法很顯然不可實現因為單個器件上的資源是有限的。也可能沒有足夠的**DSP**資源在每個時鐘週期內完成大量的乘法運算，或者沒有足夠的外部存儲帶寬用芯片和外部數據的交互。在系統中許多**FPGA** 設計需要和外部數據速率相匹配，例如模數轉換（A/D）或 通信系統中的波特率。在這些設計中， 通常使用**pipeline** directive對最內層循環，目的是為了實現計算間隔和系統數據速率匹配。 在這些例子中，我們需要探索不同資源和速率之間的平衡，通過將**pipeline**放到內層循環或對循環部分展開。當處理非常大的矩陣包含成千上萬的元素時，我們需要考慮更復雜結構。下一章節我們將討論在大型設計中對矩陣進行縮放，稱之為**blocking** 和**tiling**

{% hint style='info' %}
去掉**array_reshape** directive。這對性能有什麼影響？這對資源使用率有什麼影響？通過改變**array_reshape** 的參數，對這些數組有沒有影響？這種情況下，和只使用**array_reshape** 有什麼區別？
{% endhint %}

##  7.3 塊矩陣乘法

**塊矩陣** 是被分割出來子矩陣。直觀的理解是，可以通過在水平方向和垂直方向畫線對矩陣元素進行分割。得到的塊作為原始矩陣的子矩陣。反過來，我們可以認為原始矩陣是塊矩陣的集合。這就自然的派生出許多算法在線性代數運算中，例如矩陣的乘法，通過把大型的矩陣切分成許多小的矩陣，然後基於塊進行運算。

例如，當我們討論等式7.3和7.4 描述的矩陣**A**和矩陣**B**之間的乘法，可以認為矩陣中每一個元素$$\textbf{A}_{11}$$ 或 $$\textbf{B}_{23}$$ 作為一個數或者一個複數。或者，我們也可以把它們當作是原始矩陣的一個塊。在這種情況下，如果每個獨立的塊維度是匹配的，我們可以對塊進行計算而不是對原始矩陣進行運算。例如，我們計算$$\textbf{AB}_{11}$$ ，通過計算兩個矩陣積和兩個矩陣加得到$$\textbf{A}_{11}\textbf{B}_{11} +\textbf{A}_{12} \textbf{B}_{21}  + \textbf{A}_{13} \textbf{B}_{31} $$

出於許多原因，矩陣分塊是一種非常實用的計算方式。其中一個主要的原因是根據塊，我們能發現更多的結果，探索更多的算法。實際上， 之前我們已經採用的優化對循環進行變形，例如循環展開，可以看做是一種形式簡單的分塊。另外我們採用矩陣分塊是基於矩陣天然的形式。如果矩陣有許多的0，那麼許多小的乘積就是0。如果在流水線運算中統計這些0會很困難，但是如果跳過一個塊0會比較簡單。許多矩陣是**block-diagonal**，對角線上的塊是非零，塊周圍的都是0。另外一個原因，分塊將結果分解成為許多小問題和少量數據集合的求解。這樣就增加了在本地完成數據的計算。在處理器系統中，可以很常見地看到，選擇合適的分區大小使得處理器存儲結構匹配，或者是使處理器與向量數據類型匹配。同理適用於FPGA，我們選怎合適的分區大小與片上的存儲資源相匹配，或者與片上乘加單元數目匹配。

直到現在，我們都假設在執行任務之前加速器總是有可用的數據。但是，在設計處理大型數據集，例如大型的矩陣，這就是不必要的假設。因為讓加速器能直接處理所有的數據是不可能的，我們能設計加速器，在他需要數據時才接收輸入數據。這樣就可以讓加速器更高效的使用片上存儲資源。我們稱這位**流水結構**（streaming architecture），因為每次傳輸輸入數據（也可能是輸出數據）按次進行而不是一次處理所有的。

流水結構在應用中非常普遍。在有些設計選擇時，我們將大型的計算分解為許多小的計算。例如，我們設計矩陣乘法系統，每次從外部讀取和處理一個數據塊。在其他的情況中，我們可能處理數據流因為數據是從物理環境中實時採樣而來，例如，從A/D轉換器。還有其他的例子，我們能處理的數據是來順序的來自於之前的計算或加速單元。實際上，我們在5.4節已經見過這樣的例子了。

流水的方式潛在的好處之一是減少了輸入數據和輸出數據對存儲的需求。這裏我們假設的是數據分塊，得到分塊的結果，完成了數據的處理，並且不需要存儲。當下一個數據到達，我們在小的存儲空間上對之前舊的數據進行改寫。

接下來，我們設計一個用於計算矩陣乘法的流水結構。我們將輸入的矩陣**A** 和**B** 進行了分塊，每個塊包含一定的行和列。使用這些分區塊，我們計算得到矩陣乘積**AB** 部分結果。接下來，我們採用流水的方式得到下一個塊，計算**AB** 另外一個部分的結果，直到整個矩陣乘法計算完成。

圖7.3提供描述了我們設計的流水結構。我們的結構中有變量**BLOCK_SIZE**用於指示每次執行從矩陣**A**取到的行，從矩陣**B**取到的列，**BLOCK__SIZE*BLOCK_SIZE** 的結果矩陣與**AB**矩陣對應。

圖7.3中的例子，設定BLOCK_SIZE = 2。所以每次我們執行時從矩陣**A** 取2行，從矩陣**B**取兩列，按照我們設計的流水結構。通過調用**blockmatmul**函數得到的結果針對**AB**的一個2x2的矩陣。

當我們定義4x4的矩陣是，我們需要執行4次完成計算。每次我們得到一組2x2的結果，它是**AB** 矩陣總結果的一部分。圖展示了了我們如何輸入行和列數據。在圖7.3 a)中，我們先輸入矩陣**A**的頭2行和矩陣**B** 的頭2列，。函數計算一個2x2的矩陣對應結果矩陣**AB**的開始2行和2列的數據。

圖7.3 b)中，我們仍然使用矩陣**A**的頭2行，但是這次我們輸入矩陣**B**最後的2列。我們不需要從矩陣**A**讀取新的行，因為它和之前執行的數據是相同的。我們這次得到的2x2矩陣額結果對應的是矩陣**AB** “右上角”的數據。

圖7.3 c)中，矩陣**A** 和矩陣**B**都需要輸入新的數據。這次我們輸入矩陣**A**最後2行的數據和矩陣**B**開始2列的數據。這次計算的結果對應矩陣**AB**“左下角”的值。

最後執行流水分塊矩陣乘法，如圖7.3 d)所示，使用上一次迭代矩陣**A**最後2行的數據。輸入矩陣**B**最後兩列的數據。得到結果對用的是矩陣**AB**“右下角”的元素。

在展示矩陣分塊成分的代碼之前，我們定義我們可能用到的數據類型。圖7.4展示了工程調用的頭文件。我們自定義用户數據類型**DTYPE**，用於指示在矩陣**A** 和**B**相乘過程總用到的，和對應**AB**的數據類型。

![圖 7.3:  一種可能的分塊方式是將矩陣分解為2個4x4的矩陣。整個AB 乘被分為4個矩陣乘法，分別是對A進行2x4個塊，對B進行4x2個塊](images/blockmm.jpg)

```C++
#ifndef _BLOCK_MM_H_
#define _BLOCK_MM_H_
#include "hls_stream.h"
#include <iostream>
#include <iomanip>
#include <vector>
using namespace std;

typedef int DTYPE;
const int SIZE = 8;
const int BLOCK_SIZE = 4;

typedef struct { DTYPE a[BLOCK_SIZE]; } blockvec;

typedef struct { DTYPE out[BLOCK_SIZE][BLOCK_SIZE]; } blockmat;

void blockmatmul(hls::stream<blockvec> &Arows, hls::stream<blockvec> &Bcols,
								 blockmat & ABpartial, DTYPE iteration);
#endif

```
![圖7.4：分塊矩陣乘法的頭文件。文件定義了函數內部使用的數據結構，最關鍵的是，函數blockmatmul的接口。](images/placeholder.png)

{% hint style='info' %}
使用用户自定義的數據類型是很好的編程實踐。在未來的設計迭代中，設計中只用修改一處源文件，這樣你能輕鬆的修改數據類型。根據設計的類型修改數據類型是非常常見的。例如，剛開始設計我們採用的是**float**或者**double**類型得到了正確結果。後期你可能使用定點化數據，這樣在出錯時能提供基本的框架。定點化數據類型減少了資源的使用，提高了性能， 這些是以損失部分數據精度為條件的。你可能會嘗試很多種定點類型的數據直到你找到在精度和準確率/錯誤率、性能和資源佔用率之間的平衡。
{% endhint %}

**SIZE** 確定了每次矩陣相乘的行數和列數。這樣就限制了矩陣類型為平方矩陣，但是我們可以通過修改代碼中不同的地方來處理不同類型的矩陣。我們把這作為一個練習留給讀者。
{% hint style='tip' %}
修改代碼，實現對各種維度矩陣的支持。
{% endhint %}

變量**BLOCK_SIZE**定義了每次運算從矩陣**A**取的行數和矩陣**B**取的列數。這也定義了一次流水，函數能處理的數量。我們每次從函數得到的輸出數據是矩陣**AB**的一個**BLOCK_SIZE** x **BLOCK_SIZE** 的。

數據類型**blockvec**是用於傳遞矩陣**A**的**BLOCK_SIZE**行和矩陣**B**的**BLOCK_SIZE**供函數每次執行。數據類型**blockmat**用於保存部分矩陣**AB**的結果。
最後，數據類型**blockmat**是一個**BLOCK_SIZE**x**BLOCK_SIZE** 的數組。它用於保存執行函數**matrmatmul**計算的結果。

函數原型本身的兩個輸入類型是**hls::stream<blockvec>&** 。它是一串**blockvec**類型的數據。記住，一個**blockvec**是由**BLOCK_SIZE**個元素構成的數組。

在**Vivado®HLS** 中，通過調用類**hls::stream<>** 創建**FIFO**類型的數據結構，可以很好的進行仿真和綜合。數據通過**write()** 函數順序的寫入，通過**read()** 函數讀出。這個庫之所以被創建，是因為流水在硬件設計中，是一種常見的數據傳遞形式，並且這種操作可以用**C**語言進行多種方式的描述，例如，通過數組。實際上， **Vivado®HLS** 在處理負責的訪問或多維數組時，很難推測出流水行為的。內建的流庫方便程序員準確的指定流訪問的順序，避免在接口有任何限制。
{% hint style='info' %}
**hls::stream** 只能通過引用的方式在函數之間進行傳遞，例如我們在圖7.5中在函數**blockmatmul**中用法。
{% endhint %}

圖7.5中的代碼展示了通過流水的方式分塊矩陣乘法。代碼由3部分構成，分別標記為**loadA**、**partialsum** 和**writeoutput**。

代碼的第一部分，標籤為**loadA**，只要當**it %(SIZE/BLOCK_SIZE) == 0** 程序才會執行。這樣可以幫我節約時間，通過複用之前函數運行用到矩陣 **A** 的數據。

記住，每次函數**blockmatmul**執行的時候，我們從矩陣**A**取**BLOCK_SIZE**行，從矩陣**B**取**BLOCK_SIZE**列。矩陣**A**每個**BLOCK_SIZE**行對應多個**BLOCK_SIZE**列。變量**it**記錄函數**blockmatmul**被調用的次數。所以，在每次執行函數時，我們可以判斷是不是需要從矩陣**A**取新行。當不需要輸入新行，這樣就節省了我們的時間。當需要取新的數據，通過從**Arows**取數據存到一個靜態的二維矩陣**A[BLOCK_SIZE]** **[SIZE]** 中。

```c
#include "block_mm.h"
void blockmatmul(hls::stream<blockvec> &Arows, hls::stream<blockvec> &Bcols,
        blockmat &ABpartial, int it) {
  #pragma HLS DATAFLOW
  int counter = it % (SIZE/BLOCK_SIZE);
  static DTYPE A[BLOCK_SIZE][SIZE];
  if(counter == 0){ //only load the A rows when necessary
    loadA: for(int i = 0; i < SIZE; i++) {
      blockvec tempA = Arows.read();
      for(int j = 0; j < BLOCK_SIZE; j++) {
        #pragma HLS PIPELINE II=1
        A[j][i] = tempA.a[j];
      }
    }
  }
  DTYPE AB[BLOCK_SIZE][BLOCK_SIZE] = { 0 };
  partialsum: for(int k=0; k < SIZE; k++) {
    blockvec tempB = Bcols.read();
    for(int i = 0; i < BLOCK_SIZE; i++) {
      for(int j = 0; j < BLOCK_SIZE; j++) {
        AB[i][j] = AB[i][j] +  A[i][k] * tempB.a[j];
      }
    }
  }
  writeoutput: for(int i = 0; i < BLOCK_SIZE; i++) {
    for(int j = 0; j < BLOCK_SIZE; j++) {
      ABpartial.out[i][j] = AB[i][j];
    }
  }
}
```
![圖7.5： 函數blockmatmul從矩陣A輸入一系列大小為BLOCK_SIZE的行，從矩陣 B 輸入一系列大小為BLOCK_SIZE 的列，創建一個BLOCK_SIZWxBLOCK_SIZE 分佈結果，作為矩陣AB 的一部分。代碼的第一部分（標記為loadA ）保存 A 的行到局部存儲，第二部分是嵌套的partialsum for循環執行部分結果的計算，最後的一部分（標記為 writeoutput ）將之前計算返回的分佈結果放到合適格式中。](images/placeholder.png)

需要對**stream**類進行一些解釋説明才能完全掌握這段代碼，並且使用它。**stream** 類型變量**Arows**由許多**blockvec**類型的元素構成。**blockvec** 是一個**BLOCK_SIZE**x**BLOCK_SIZE**的矩陣。每個**Arows**中的元素都是一個數組，用於保存矩陣**A**中**BLOCK_SIZE**行的數據。所以在每次調用**blockmatmul**函數時，**Arow** 流會有**SIZE**個元素，每個用於保存**BLOCK_SIZE**行。語句**tempA = Arows.read()** 從**Arows** 流中取一個元素。然後，將這些對應的元素賦值給局部矩陣**A** 中對應的位置。

{% hint style='info' %}
**stream** 類對操作符 << 進行了重載，它等價於函數**read()** 。所以，語句** tempA = Arows.read()** 和 ** tempA << Arows** 執行時等價的。
{% endhint %}
剩餘部分的計算式是部分求和。大部分都是在函數**blockmatmul** 中進行。

流類型變量**Bclos**用法和變量 **Arows** 相似。但是，它不是存儲矩陣**A**的行，而是存儲**B**矩陣參與當前運算的列數據。每次調用**blockmatmul**函數都會從矩陣**B**取到新的列數據。所以我們不需要像矩陣 **A** 那樣，每次取數據時進行條件判斷。函數本身執行步驟和圖7.1中的**matmul**很相似，但是我們只計算**BLOCK_SIZE** x **BLOCK_SIZE** 的結果與矩陣**AB**對應。因此我們每次通過迭代矩陣**A**中**BLOCK_SIZE**行和矩陣**B**中**BLOCK_SIZE**列。但是每行和每列都有**SIZE**個元素，因此循環的邊界全取決於外層**for**循環。

函數最後的部分實現數據到本地矩陣**AB**的賦值，對應的是**BLOCK_SIZE**x**BLOCK_SIZE**的**AB**輸出矩陣的部分結果。

程序的三個部分中，中間部分用於計算部分求和，是對運算要求最高的部分。通過觀察代碼，可以發着這部分嵌套了３層**for**循環，迭代的次數為**SIZE**x**BLOCK_SIZE**x**BLOCK_SIZE**。第一部分迭代的次數為**SIZE**x**BLOCK_SIZE**，最後一部分迭代的次數為**BLOCK_SIZE**x**BLOCK_SIZE**。所以，我們集中對程序中間部分進行優化，例如**分部求和**（partialsum）嵌套的**for**循環。

嵌套循環優化最開始的出發點都是對最內層的**for**循環進行流水化。如果這樣操作需要耗費大量的資源，那麼設計人員可以把**pipeline**directive放到稍外層的**for**循環。是否導致設計耗費資源是由**BLOCK_SIZE**決定的，如果資源佔用率很小，那麼把**pipeline**directive放在現在的位置是可行的。甚至把**pipeline**directive放在最外層的**for**循環也是可行的。這樣，內層的2層**for**循環都會展開，可能會導致資源使用量大幅上升。但是，這樣性能也是有提升的。

{% hint style='tip' %}
改變**BLOCK_SIZE**是如何影響性能和資源佔用率的？改變常量**SIZE**影響是什麼？移動**pipeline**directive 在**partiaslum**函數的嵌套的三層**for**循環之間，性能和資源佔用率會發生什麼變化？
{% endhint %}

函數開始部分的**dataflow** directive 實現函數部分之間的流水線，例如**loadA for**循環，**partialsum** 嵌套的**for**循環、**writeoutput for** 循環。使用這個directive可以減少函數**blockmatmul**運行的間隔。但是，代碼三個部分中最大運行間隔是最終的限制條件。也就是説，函數**blockmatmul**運行間隔的最大值，我們稱之為interval**(blockmatmul)** 是要大約等於interval**(loadA)** ,interval**(partialsum)** ，interval**(writeoutput)** 中的最大值的。

$$
\begin{aligned}
interval\textbf{(blockmatmul)} \geq \\
 max(interval\textbf{(loadA)},interval\textbf{(partialsum)},  interval\textbf{(writeoutput)})
\end{aligned}
\quad(7.5)
$$

當我們對函數**blockmatmul**進行優化的時候，我們需要牢牢記住等式7.5。例如，假設interval**(partialsum)** 比函數部分中其他兩個都大。針對interval **(loadA)** 和interval **(writeoutput)** 的優化都是無用的，因為interval **(blockmatmul)** 不會變小。所以，設計者需要將所有的注意力放在對interval**(partialsum)** 的優化上，目標性能的優化就在嵌套的三層**for**循環上。

在進行性能優化時意識到這一點是非常重要的。開發人員可以對函數中其它兩個部分進行資源優化。通常情況下這種減少資源佔用率的優化都會提高執行間隔和延遲。在這個例子中，這種增加間隔不會對函數**blockmatmul** 整個性能有影響。實際上最理想的方式，對函數的三個部分同時進行優化，它們有相同的運行間隔，這樣我們就可以輕鬆的在運行間隔和資源佔用上實現平衡（實際可能不總是這樣）。

函數**blockmatmul**的測試平台如圖7.6和7.7。我們將它分成為兩個圖片，這樣可讀性更強， 因為這段代碼實在太長了。到目前為止，我們都是沒有看到測試平台的。我們展示這個測試平台有一下幾點原因。首先，它讓我們知道函數**blockmatmul** 的工作流程。特別是，它將輸入的矩陣進行分塊，然後一個塊接一個塊的輸入給函數**blockmatmul**。其次，它演示瞭如何在複雜的使用場景中使用**stream**模板進行仿真。最後，他給讀者一個如何建立合適測試平台的概念。

函數**matmatmul_sw** 通過簡單的三層**for** 循環實現了矩陣乘法。它將2個二維的矩陣作為輸入，輸出是一個二維矩陣。這和我們之前在圖7.1中看到的函數**matrixmul**很像。我們用這個函數的結果和硬件版本分塊矩陣乘法進行做比較。

讓我們先從圖7.6開始，觀察測試平台的前半部分。開頭的代碼完成對函數其餘部分變量的初始化。變量**fail**用於記錄矩陣乘法是否計算正確。在函數的後面會檢查這個變量。變量**strm_matrix1**和**strm_matrix2**是**hls:stream<>** 類型，分別用於保存矩陣**A**的行和矩陣**B** 的列。這些流類型的變量中每個元素是一個< blockvec >。參考圖7.4所示的頭文件**block_MM.h**。我們反覆調用定義的**blockvec**作為數組數據，我們用**blockvec**保存一行和一列數據。
{% hint style='info' %}

變量**stream**在**HLS**的命名空間中。所以，我們使用對應的命名空間，並且加載**hls::stream**，或者直接簡化用**stream**。但是，實際使用更傾向在**stream**前保留**hls::** ，可以提箱讀者這裏的stream是和**Vivado®HLS**相關的而不是來自於其他**C**庫函數。同樣，這樣可以避免潛在與其他新命名空間產生衝突。
{% endhint %}

接下來的代碼開始定義變量**strm_matix1_element**和**strm_matrix2_element**。這兩個變量用來作為每個**blockvec**變量分別寫入流變量**strm_matrix1**和**strm_matrix2**的中間過度變量。變量**block_out**用於保存函數**blockmatmul**的輸出結果。注意這個變量使用**blockmat**類型，它是一個大小為**BLOCK_SIZE**x**BLOCK_SIZE**的二維矩陣，在頭文件**block_mm.h**定義（見圖7.4）。最後定義的是矩陣**A**、**B**、**matrix_swout**和**matrix_hwout**。它們都是大小為**SIZE**x**SIZE**的二維矩陣，其中數據類型都是**DTYPE**。

{% hint style='info' %}
你可以對流使用初始化進行命名。這是一個很好的方式，因為它輸出的錯誤信息更準確。如果沒有名字，錯誤信息只會提供一個通用的參考到流對應的數據類型。如果你有多個流，在聲明的時候都使用相同的數據類型，那麼你很難指出錯誤信息到底指向那個流。給流變量命名是通過給變量一個名字參數，例如**hls::stream\<blockvec\> strm_matrix1("strm_matrix1")**
{% endhint %}

接下來嵌套的**for**循環**initmatrices**設定四個二維矩陣**A**、**B**、**matrix_sout**和**matrix_hout**。變量**A**和**B**是輸入矩陣。它們是由在[0,512)之間的隨機數構成。我們選擇512沒有其他的原因，因為9比特可以保存對應所有的值。需要注意的是，**DTYPE** 是作為**int**類型，所以它的範圍是明顯大於[0,512)，在後面的優化過程中我們通常用範圍更小的定點數。矩陣**matrix_swout**和**matrix_hwout**都被初始化為0。他們後續通過調用函數**matmatmul**和**blockmatmul**進行賦值。
第二部分的測試平台如圖7.7所示。這是**main**函數中最後的一部分了。

圖中的第一部分是一系列複雜的嵌套的**for**循環。所有這些**for**循環的目的是為了實現對輸入矩陣**A**、**B** 的處理，使它們能流入到函數**blockmatmul**中。函數**blockmatmul**保存結果在**matrix_hwout**中。

外部的兩層**for**循環每一步都以塊的形式實現輸入矩陣。你可以發現每次迭代都是以**BLOCK_SIZE**作為步進。接下來的兩個**for**循環把矩陣**A**的行寫到**strm_matrix1_element**，把矩陣**B**的列寫到**strm_martix2_element**中。在執行上述步驟時都是一個元素一個元素進行的，通過使用變量**k**訪問單個的值從行（列）然後將它們寫入到一維數組中。注意，**strm_matrix1_element** 和**strm_martix2_element**都是**blockvec**數據類型，即長度為**BLOCK_SIZE**的一維數組。它們用於保存行或列的**BLOCK_SIZE**個元素。內層循環迭代**BLOCK_SIZE**次。流類型變量**strm_matrix1**和**strm_matrix2**被寫入**SIZE**次。換句話説，就是對整行（列）進行緩存，緩存中每個元素保存**BLOCK_SIZE**個值。

{% hint style='info' %}
類**stream**通過重載**>>** 操作符，它等價於函數 **write（data)** 。這和對 **read()** 函數重載，使其等價於操作符**<<** 。因此，表達式 **strm_matrix1.write(strm_matrix1_element)** 和 **strm_matrix1_element >> strm_matrix1** 執行是相同的。
{% endhint %}

```c
#include "block_mm.h"
#include <stdlib.h>
using namespace std;

void matmatmul_sw(DTYPE A[SIZE][SIZE], DTYPE B[SIZE][SIZE],
      DTYPE out[SIZE][SIZE]){
 DTYPE sum = 0;
 for(int i = 0; i < SIZE; i++){
  for(int j = 0;j<SIZE; j++){
   sum = 0;
   for(int k = 0; k < SIZE; k++){
    sum = sum + A[i][k] * B[k][j];
   }
   out[i][j] = sum;
  }
 }
}

int main() {
 int fail = 0;
 hls::stream<blockvec> strm_matrix1("strm_matrix1");
 hls::stream<blockvec> strm_matrix2("strm_matrix2");
 blockvec strm_matrix1_element, strm_matrix2_element;
 blockmat block_out;
 DTYPE A[SIZE][SIZE], B[SIZE][SIZE];
 DTYPE matrix_swout[SIZE][SIZE], matrix_hwout[SIZE][SIZE];

 initmatrices: for(int i = 0; i < SIZE; i++){
  for(int j = 0; j < SIZE; j++){
   A[i][j] = rand() % 512;
   B[i][j] = rand() % 512;
   matrix_swout[i][j] = 0;
   matrix_hwout[i][j] = 0;
  }
 }
}
 //the remainder of this testbench is displayed in the next figure
```
![圖7.6： 分塊矩陣乘法測試平台的第一部分。函數被分為兩個部分， 因為太長不能在單頁上一次顯示。測試平台剩下的部分在圖7.7中。其中有一個“軟件”版本的矩陣乘法、變量的聲明和初始化。](images/placeholder.png)

```c
// The beginning of the testbench is shown in the previous figure
int main() {
  int row, col, it = 0;
  for(int it1 = 0; it1 < SIZE; it1 = it1 + BLOCK_SIZE) {
    for(int it2 = 0; it2 < SIZE; it2 = it2 + BLOCK_SIZE) {
      row = it1; //row + BLOCK_SIZE * factor_row;
      col = it2; //col + BLOCK_SIZE * factor_col;

      for(int k = 0; k < SIZE; k++) {
        for(int i = 0; i < BLOCK_SIZE; i++) {
          if(it % (SIZE/BLOCK_SIZE) == 0) strm_matrix1_element.a[i] = A[row+i][k];
          strm_matrix2_element.a[i] = B[k][col+i];
        }
        if(it % (SIZE/BLOCK_SIZE) == 0) strm_matrix1.write(strm_matrix1_element);
        strm_matrix2.write(strm_matrix2_element);
      }
      blockmatmul(strm_matrix1, strm_matrix2, block_out, it);

      for(int i = 0; i < BLOCK_SIZE; i++)
        for(int j = 0; j < BLOCK_SIZE; j++)
          matrix_hwout[row+i][col+j] = block_out.out[i][j];
      it = it + 1;
    }
  }

  matmatmul_sw(A, B, matrix_swout);

  for(int i = 0; i<SIZE; i++)
    for(int j = 0; j<SIZE; j++)
      if(matrix_swout[i][j] != matrix_hwout[i][j]) { fail=1; }

  if(fail==1) cout << "failed" << endl;
  else cout << "passed" << endl;

  return 0;
}

```
![圖7.7：分塊矩陣乘法的第二部分。第一部分在圖7.6中。這部分演示如何將流類型數據送入函數blockmatmul，代碼測試這個函數結果和更簡單的三重for循環得出的結果是否匹配。](images/placeholder.png)

這段代碼最後一部分從高亮的**if**語句開始。這和矩陣**A**的值相關。本質上，這裏是為了我們不用經常向**strm_matrix1**中寫入相同的值。矩陣**A**的值可以對應多次**blockmatmul**函數的調用。圖7.3是關於這的討論。這裏的**if**語句高亮是為了強調，不能在這裏一次次寫入相同值。因為函數**blockmatmul**只在有必要的時候讀這些數據。所以當我們連續寫數據，代碼執行不正常，因為流寫入的數據是多於讀出的數據。

在測試平台中調用**blockmatmul**函數輸入數據。在函數調用以後，它收到部分計算結果**block_out**。接下來的兩層**for**循環把這些結果填入到數組**matrix_hwour**正確的位置。

在執行一些列複雜的**for**循環之後，分塊矩陣乘法實現了。測試平台用於保證代碼寫入是正確的。它通過比較調用函數**blockmatmul**計算的輸出結果和調用**matmatmul_sw**計算的輸出結果。在函數調用以後，測試平台迭代遍歷二維矩陣**matrix_hwout**和**matrix_swout**，保證每個元素都是相等的。如果其中一個或多個元素不相同，它會置**fail**為1。測試平台結束時會打印輸出**failed**或**passed**。

{% hint style='info' %}
需要注意的是，你不能直接比較函數**blockmatmul**用於實現矩陣乘法的性能，例如圖7.1中的代碼。因為它調用了函數**blockmatmul**函數很多次用於實現整個矩陣乘法。比較需要在相同條件下進行
{% endhint %}

{% hint style='tip' %}
為了實現完整的矩陣乘法，寫一個函數用來確定**blockmatmul**需要被調用多少次。這個函數應該是通用的，它應該不需要假定特殊的值如**BLOCK_SIZE**或矩陣的大小（例如，**SIZE**）。
{% endhint %}

{% hint style='tip' %}
比較分塊矩陣乘法和矩陣乘法之間的資源佔用率。當增大矩陣的大小，佔用率是如何變化的？分塊大小是不是在資源佔用率中起到了重要作用？通常的趨勢是怎樣的？
{% endhint %}

{% hint style='tip' %}
比較分塊矩陣乘法和矩陣乘法之間的性能。當增大矩陣的大小，性能如何變化？分塊大小是否在性能中起到了重要作用？選擇兩種有相同資源佔用率的架構，這兩種架構之間性能差異有多大？
{% endhint %}

## 7.4 總結 ##

矩陣乘法提供了一種不同的方式來實現矩陣乘法。函數通過流的方式輸入矩陣的部分，然後計算矩陣的部分結果。函數通過多次計算，完成對整個矩陣乘法的計算。
