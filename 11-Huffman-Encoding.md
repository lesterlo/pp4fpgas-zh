# 第十一章 霍夫曼編碼
## 11.1 背景

無損數據壓縮是高效數據存儲中的一個關鍵要素，而霍夫曼編碼則是其中最流行的可變長度編碼算法[[33](./BIBLIOGRAPHY.md#33)]。 給定一組數據符號以及它們出現的頻率，霍夫曼編碼將以更短的代碼分配給更頻繁（出現的）符號這種方式生成碼字來最小化平均代碼長度。由於它保證了最優性，霍夫曼編碼已被各種應用廣泛採用[[25](./BIBLIOGRAPHY.md#25)]。 在現代多級壓縮設計中，它經常用作系統的後端，來提高特定領域前端的壓縮性能，如GZIP[[23](./BIBLIOGRAPHY.md#23)]，JPEG[[57](./BIBLIOGRAPHY.md#57)]和MP3[[59](./BIBLIOGRAPHY.md#59)]。 儘管算術編碼[[61](./BIBLIOGRAPHY.md#61)]（霍夫曼編碼的一個廣義版本，它將整個消息轉換為單個數字）可以對大多數場景實現更好的壓縮，但是由於算術編碼專利問題[[38](./BIBLIOGRAPHY.md#38)]，霍夫曼編碼通常成為許多系統的首選算法。

Canonical霍夫曼編碼與傳統的霍夫曼編碼相比有兩大優勢。 在基本霍夫曼編碼中，編碼器將完整霍夫曼樹結構傳遞給解碼器。 因此，解碼器必須遍歷樹來解碼每個編碼的符號。 另一方面，Canonical霍夫曼編碼僅將每個符號的位數傳送給解碼器，然後解碼器重構每個符號的碼字。這使得解碼器在內存使用和計算需求中更加高效。 因此，我們專注於Canonical霍夫曼編碼。

在基本的霍夫曼編碼中，解碼器通過遍歷霍夫曼樹，從根開始直到它到達葉節點來解壓縮數據。這種方式有兩個主要缺點：第一、它需要存儲整個霍夫曼樹，從而增加了內存使用量。此外，為每個符號遍歷樹的計算成本是很高的。Canonical霍夫曼編碼通過使用標準的規範格式創建代碼來解決這兩個問題。使用Canonical編碼的好處在於：我們只需要傳輸每個霍夫曼碼字的長度。一個Canonical霍夫曼代碼具有兩個額外的屬性：首先，較長代碼比相同長度前綴的較短代碼具有更高的數值。其次，具有相同長度的代碼隨着符號值的增加而增加。這意味着如果我們知道每個代碼長度的起始符號，我們可以很容易重建Canonical霍夫曼編碼。霍夫曼樹本質上等同於“排序”版本的原始霍夫曼樹，以便更長的碼字在樹的最右邊的分支上，樹的同一級別上的所有節點按照符號的順序排序。


![圖11.1：Canonical霍夫曼編碼過程。 符號被過濾、排序，並且被用來建造霍夫曼樹，而不是將整個樹傳遞給解碼器（就像“基本的” 霍夫曼編碼所完成的那樣），這樣編碼使得只有樹的符號長度是解碼器所需要的。 請注意，最終Canonical樹與初始樹在流程開始創建時是不同的。](images/canonical_huffman_flow.jpg)

圖11.1展示了創建Canonical霍夫曼編碼的過程。filter模塊僅傳遞非零頻率的符號。sort模塊根據它們的頻率按升序重新排列符號。接下來，create tree模塊使用以下三個步驟構建霍夫曼樹：

1. 使用兩個最小頻率節點作為初始子樹並通過求它們頻率的和來生成新的父節點；
2. 它將新的中間節點添加到列表中，並再次分類;
3. 它從列表中選擇兩個最小元素並重復這些步驟直到剩下最後一個元素為止。

結果是霍夫曼樹中的每個葉節點代表可以被編碼的符號，並且每個內部節點被標記為該子樹節點的頻率。通過將樹中的左邊和右邊與位0和1相關聯，我們可以根據從根節點到達它的路徑來確定每個符號唯一的碼字。例如，A的碼字是00，而B的碼字是1110。
這樣就完成了基本的霍夫曼編碼過程，但是對於創建Canonical霍夫曼樹這並不是必須的。

為了創建霍夫曼樹，我們做了幾個額外的轉換。 首先，compute bit len模塊計算每個碼字的位長度，然後對每個長度的頻率進行計數，結果是一個碼字長度的直方圖（見第8.2節）。 在這個例子中，我們有三個符號（A，D，E），代碼長度為2。因此，計算的直方圖表中包含了位置2中的數值3。接下來，truncate tree模塊重新平衡霍夫曼樹以避免出現過長的碼字。這可以在編碼時間稍微增加的代價下提高解碼器的速度。 這在圖11.1的示例中不是必需的。我們設定了樹的最大高度為27。 最後，canonize tree模塊創建兩個排序表，第一個表格包含了符號和按符號排序的符號長度。第二個表格包含了符號和按長度排序的符號長度。 這些表格簡化了為每個符號創建Canonical霍夫曼碼字的過程。

create codeword模塊通過遍歷已排序的表創建Canonical霍夫曼碼字表。從排序表中的第一個碼字開始，它被分配了適當長度全零碼字。每個具有相同位長度的後續符號被賦值為隨後的碼字，它是在前一個碼字上簡單地加1而形成的。在我們的例子中，符號A，D和E的位長度均是l = 2，而分配的碼字分別是A = 00，D = 01，E = 10。 請注意符號是按字母順序考慮的，這是使樹規範化所必需的。這個過程一直持續直到我們得到一個需要一個更大長度的碼字，在這種情況下，我們不僅增加了先前的碼字，而且還向左移位以生成正確長度的碼字。在這個例子中，下一個符號是C，長度為3，其接收的碼字C =（10 + 1）<< 1 = 11 << 1 = 110。接下來，下一個符號是B，長度為4.再一次，我們增加和移位一次。於是B的代碼字為B =（110 + 1）<< 1 = 1110.符號F的最終碼字為F= 1110 + 1 = 1111。詳見第11.2.7章。

Canonical霍夫曼代碼的創建包括許多複雜的和內在的順序計算。 例如，create_tree模塊需要跟蹤被創建的子樹的正確順序、需要仔細的內存管理，可以被利用的並行性非常有限。下面我們來討論使用Vivado HLS進行Canonical霍夫曼編碼設計的硬件架構和實現。

```c

#include "huffman.h"
#include "assert.h"
void huffman_encoding(
    /* input */ Symbol symbol_histogram[INPUT_SYMBOL_SIZE],
    /* output */ PackedCodewordAndLength encoding[INPUT_SYMBOL_SIZE],
    /* output */ int *num_nonzero_symbols) {
    #pragma HLS DATAFLOW

    Symbol filtered[INPUT_SYMBOL_SIZE];
    Symbol sorted[INPUT_SYMBOL_SIZE];
    Symbol sorted_copy1[INPUT_SYMBOL_SIZE];
    Symbol sorted_copy2[INPUT_SYMBOL_SIZE];
    ap_uint<SYMBOL_BITS> parent[INPUT_SYMBOL_SIZE-1];
    ap_uint<SYMBOL_BITS> left[INPUT_SYMBOL_SIZE-1];
    ap_uint<SYMBOL_BITS> right[INPUT_SYMBOL_SIZE-1];
    int n;

    filter(symbol_histogram, filtered, &n);
    sort(filtered, n, sorted);

    ap_uint<SYMBOL_BITS> length_histogram[TREE_DEPTH];
    ap_uint<SYMBOL_BITS> truncated_length_histogram1[TREE_DEPTH];
    ap_uint<SYMBOL_BITS> truncated_length_histogram2[TREE_DEPTH];
    CodewordLength symbol_bits[INPUT_SYMBOL_SIZE];

    int previous_frequency = -1;
 copy_sorted:
    for(int i = 0; i < n; i++) {
        sorted_copy1[i].value = sorted[i].value;
        sorted_copy1[i].frequency = sorted[i].frequency;
        sorted_copy2[i].value = sorted[i].value;
        sorted_copy2[i].frequency = sorted[i].frequency;
        // std::cout << sorted[i].value << " " << sorted[i].frequency << "\n";
        assert(previous_frequency <= (int)sorted[i].frequency);
        previous_frequency = sorted[i].frequency;
    }

    create_tree(sorted_copy1, n, parent, left, right);
    compute_bit_length(parent, left, right, n, length_histogram);

#ifndef __SYNTHESIS__
    // Check the result of computing the tree histogram
    int codewords_in_tree = 0;
 merge_bit_length:
    for(int i = 0; i < TREE_DEPTH; i++) {
        #pragma HLS PIPELINE II=1
        if(length_histogram[i] > 0)
            std::cout << length_histogram[i] << " codewords with length " << i << "\n";
        codewords_in_tree += length_histogram[i];
    }
    assert(codewords_in_tree == n);
#endif

        truncate_tree(length_histogram, truncated_length_histogram1, truncated_length_histogram2);
    canonize_tree(sorted_copy2, n, truncated_length_histogram1, symbol_bits);
    create_codeword(symbol_bits, truncated_length_histogram2, encoding);

    * num_nonzero_symbols = n;
}
```

![圖11.2: 顯示了整個“top” Huffman_encoding函數，它設置了數組和其他在各個子函數之間傳遞的變量，並實例化了這些函數。](images/placeholder.png)

有些額外的數據複製可能是不必要的，這是因為我們使用了dataflow指令，這給子函數間變量的傳遞起到了限制作用。特別是，部分函數之間，生產者和消費者的數據關係，有一些嚴格的規則， 這就要求我們複製一些數據。 例如，我們創建了數組parent，Left和Right的兩個副本。我們也與數組截短位長度相同。 前者是在top霍夫曼編碼函數的for循環中完成的;後者是在canonize tree函數內完成的。

{% hint style='tip' %}
dataflow指令限制函數中的信息流動。許多限制強化了子函數之間嚴格的生產者和消費者關係。一個這樣的限制是，一個數組應該只由一個函數寫入，並且它只能由一個函數讀取。即它只能作為從一個函數的輸出到另一個函數的輸入。如果多個函數從同一數組中讀取，Vivado HLS將綜合代碼，但會發出警告並且不會使用數據流水線式架構。因此，使用數據流模式通常需要將數據複製到多個數組中。如果一個函數試圖從一個數組讀寫，而這個數組同時被另一個函數訪問，則會出現類似的問題。在這種情況下，必須保留函數內部的數據一個額外的內部副本。我們將討論這兩個要求以及如何遵守它們，正如我們在本章其餘部分所述的那樣。
{% endhint %}

## 11.2 實現
Canonical霍夫曼編碼過程本質上是被分成子函數。因此，我們可以逐一處理這些子函數。 在我們這樣做之前，我們應該考慮這些函數的每個接口。
圖11.4顯示了函數及其輸入和輸出的數據。 為了簡單起見，它只顯示與數組的接口，由於它們很大，我們可以假設它們存儲在RAM塊中（BRAMs）。 在我們描述這些函數以及他們的輸入和輸出之前，我們需要討論huffman.h中定義的常量，自定義數據類型和函數接口。
圖11.3顯示了這個文件的內容。

INPUT_SYMBOL_SIZE參數定義了作為編碼輸入的符號的最大數量。在這種情況下，我們將其設置為256，從而啟用8位ASCII數據的編碼。 TREE_DEPTH參數指定初始霍夫曼樹生成期間單個代碼字長度的上限。當霍夫曼樹在truncate_tree函數中重新平衡時，CODEWORD_LENGTH參數指定了目標樹的高度。 CODEWORD_LENGTH_BITS常量決定編碼碼字長度所需的位數，等於log2[CODEWORD_LENGTH]，在這種情況下等於5。

```c
#include "ap_int.h"

// input number of symbols
const static int INPUT_SYMBOL_SIZE = 256;

// upper bound on codeword length during tree construction
const static int TREE_DEPTH = 64;

// maximum codeword tree length after rebalancing
const static int MAX_CODEWORD_LENGTH = 27;

// Should be log2(INPUT_SYMBOL_SIZE)
const static int SYMBOL_BITS = 10;

// Should be log2(TREE_DEPTH)
const static int TREE_DEPTH_BITS = 6;

// number of bits needed to record MAX_CODEWORD_LENGTH value
// Should be log2(MAX_CODEWORD_LENGTH)
const static int CODEWORD_LENGTH_BITS = 5;

// A marker for internal nodes
const static ap_uint<SYMBOL_BITS> INTERNAL_NODE = -1;

typedef ap_uint<MAX_CODEWORD_LENGTH> Codeword;
typedef ap_uint<MAX_CODEWORD_LENGTH + CODEWORD_LENGTH_BITS> PackedCodewordAndLength;
typedef ap_uint<CODEWORD_LENGTH_BITS> CodewordLength;
typedef ap_uint<32> Frequency;

struct Symbol {
	 ap_uint<SYMBOL_BITS> value;
	 ap_uint<32> frequency;
};

void huffman_encoding (
	Symbol in[INPUT_SYMBOL_SIZE],
	PackedCodewordAndLength encoding[INPUT_SYMBOL_SIZE],
	int *num_nonzero_symbols
);
```
如圖11.3：頂層函數huffman_encoding的參數、自定義數據類型和函數接口。

![圖11.4：Canonical霍夫曼編碼的硬件實現框圖。灰色塊表示由不同子函數生成和消耗的重要輸入和輸出數據。白色塊對應於函數（計算核心）。注意，數組初始位長度出現了兩次（圖中initial bit length模塊），以使圖形更加清晰。](images/che_dataflow.jpg)

我們創建一個自定義數據類型Symbol來保存對應於輸入值和它們的頻率的數據。 該數據類型在需要訪問這些信息的編碼過程中被用於filter, sort，以及其它一些函數。數據類型有兩個字段：數值和頻率。 在這裏我們假定被編碼的數據塊包含不超過2^32個符號。
最後，huffman.h 文件具有huffman_encoding函數接口。 這是Vivado HLS工具指定的頂層函數。它有三個參數, 第一個參數是大小為INPUT_SYMBOL_SIZE的符號數組。該數組表示被編碼塊中數據頻率的直方圖。 接下來的兩個參數是輸出。 encoding參數
輸出每個可能符號的碼字。Num_nonzero_symbols參數是來自輸入數據的非零符號的數量 ，這與filter操作後剩餘的符號數量相同。
系統的輸入是一個Symbol數組，這保存了數組IN中的符號值和頻率。每個符號保存10-bit value和32-bit frequency。該數組的大小設置為常數INPUT_SYMBOL_SIZE，在本例中為256。filter模塊從in數組讀取，並將其輸出寫入到filtered數組。這是一個Symbols數組，它保存了sort模塊輸入的非零元素個數。 sort模塊將按頻率排序的符號寫入兩個不同的數組中 - 一個用於create tree模塊，另一個用於canonize tree模塊。create tree模塊從排序後的數組中創建一個霍夫曼樹並將其存儲到三個數組中（(parent, left, and right）;這些數組擁有霍夫曼樹每個節點的所有信息。使用霍夫曼樹信息，compute bit len模塊計算每個符號的位長並將該信息存儲到initial bit len數組。我們將最大條目數設置為64，覆蓋最高64位的頻率數，這對於大多數應用來説是足夠的，因為我們的霍夫曼樹能夠重新平衡它的高度。truncate tree模塊重新平衡樹高，並將每個碼字的位長度信息複製到兩個單獨的truncated bit length數組中。它們每個都有完全相同的信息，但它們必須被複制以確保Vivado HLS工具可以執行流水線功能;稍後我們將會更詳細地討論它。 canonize tree模塊遍歷sort模塊中的每個符號，並使用truncated bit length數組分配適當的位長度。canonize模塊的輸出是一個數組，其中包含每個符號碼字的位長度。最後，create codeword模塊為每個符號生成canonical碼字。

```c
#include "huffman.h"
// Postcondition: out[x].frequency > 0
void filter(
            /* input  */ Symbol in[INPUT_SYMBOL_SIZE],
            /* output */ Symbol out[INPUT_SYMBOL_SIZE],
            /* output */ int *n) {
#pragma HLS INLINE off
    ap_uint<SYMBOL_BITS> j = 0;
    for(int i = 0; i < INPUT_SYMBOL_SIZE; i++) {
#pragma HLS pipeline II=1
        if(in[i].frequency != 0) {
            out[j].frequency = in[i].frequency;
            out[j].value = in[i].value;
            j++;
        }
    }
    * n = j;
}
```
![圖11.5：filter函數在輸入數組in中迭代，並將具有非零頻率字段的任何符號條目添加到輸出數組out。此外，它記錄非零頻率元素的數量並將其傳遞到輸出參數n。](images/placeholder.png)

### 11.2.1 過濾
霍夫曼編碼過程的第一個函數是filter，如圖11.5所示。 這個函數的輸入是Symbol數組。 輸出是另一個Symbol數組，它是輸入數組in的一個子集。Filter函數刪除任何頻率等於0的條目。函數本身只是簡單的迭代in數組，如果其frequency域不為零，則將每個元素存儲到out數組中。 另外，該函數計算輸出的非零條目數，作為輸出參數n傳遞，使其他函數只能處理“有用的”數據。

{% hint style='tip' %}
Vivado HLS可以自動內聯函數以便生成更高效的架構。 大多數情況下，這發生在小函數上。 inline指令允許用户明確指定Vivado HLS是否應該內聯特定的功能。在這種情況下，INLINE off 確保該功能不會被內聯，並且會在生成寄存器傳輸級（RTL）設計中作為模塊出現。 在這種情況下，禁用內聯可以使我們能夠獲得該函數的性能和資源使用情況，並確保它將作為頂層數據流設計中的一個流程來實施。
{% endhint %}

### 11.2.2 分類
如圖11.6所示，sort函數根據對輸入符號的frequency值進行排序。該函數由兩個for循環組成，標記為copy_in_to_sorting和radix_sort。copy_in_to_sorting循環把輸入的數據從in數組搬移到sorting數組中，這確保了in數組是隻讀的，以滿足在頂層使用的dataflow指令的要求。 sorting函數在整個執行過程中讀取和寫入sorting數組。即使對於這樣的簡單循環，使用pipeline指令生成最有效的結果和最準確的性能估計是很重要的。
radix_sort 循環實現核心基數排序算法。 通常，基數排序算法通過一次處理一個數字或一組比特來對數據進行排序。 每個數字的大小決定了排序的基數。 我們的算法在32比特的Symbol.frequency變量時處理4比特。因此我們使用基數r = 2^4 = 16排序。 對於32位數字中的每個4比特數字，我們進行計數排序。radix_sort循環執行這8個計數排序操作，以4為單位迭代到32。基數排序算法也可以從左到右（首先是最低有效位）或從右到左（首先是最高有效位）操作。算法從最低有效位運行到最高有效位。 在代碼中，基數可以通過設置RADIX和BITS_PER_LOOP參數來配置。

{% hint style='tip' %}
如果我們增加或減少基數會發生什麼？這將如何影響執行的計數排序操作的次數？這將如何改變資源使用，例如數組的大小呢？
{% endhint %}

代碼在sorting[]和previous_sorting[]中存儲當前排序的狀態。 每次迭代radix_sort_loop循環，sorting[]的當前值被複制到previous_sorting[]，然後這些值在被複制回sorting[]時被排序。digit_histogram[]和digit_location[]數組用於radix_sort_loop來實現對特定數字的計數排序。 兩個array_partition數組聲明這兩個數組應該完全映射到寄存器中。 這些數組很小而且使用頻繁，因此不佔用很多資源就可以提供良好的性能。最後，current_digit[]存儲了基數排序的當前迭代中為每個項目排序的數字。
該代碼還包含兩個assert（）調用，用於檢查有關num_symbols符號輸入。由於該變量決定了in數組中有效元素的數量，因此它必須以數組的大小為界。這樣的斷言通常是良好的防禦性編程實踐，以確保滿足該函數的假設。 在Vivado HLS中他們也有一個額外的目的。 由於num_symbols符號決定了許多內部循環執行的次數，Vivado HLS可以基於這些斷言推斷循環的計數。 另外，Vivado HLS還使用這些斷言在電路實現時最小化變量的位寬。

```c
#include "huffman.h"
#include "assert.h"
const unsigned int RADIX = 16;
const unsigned int BITS_PER_LOOP = 4; // should be log2(RADIX)
typedef ap_uint<BITS_PER_LOOP> Digit;

void sort(
    /* input */ Symbol in[INPUT_SYMBOL_SIZE],
    /* input */ int num_symbols,
    /* output */ Symbol out[INPUT_SYMBOL_SIZE]) {
    Symbol previous_sorting[INPUT_SYMBOL_SIZE], sorting[INPUT_SYMBOL_SIZE];
    ap_uint<SYMBOL_BITS> digit_histogram[RADIX], digit_location[RADIX];
#pragma HLS ARRAY_PARTITION variable=digit_location complete dim=1
#pragma HLS ARRAY_PARTITION variable=digit_histogram complete dim=1
    Digit current_digit[INPUT_SYMBOL_SIZE];

    assert(num_symbols >= 0);
    assert(num_symbols <= INPUT_SYMBOL_SIZE);
 copy_in_to_sorting:
    for(int j = 0; j < num_symbols; j++) {
#pragma HLS PIPELINE II=1
        sorting[j] = in[j];
    }

 radix_sort:
    for(int shift = 0; shift < 32; shift += BITS_PER_LOOP) {
    init_histogram:
        for(int i = 0; i < RADIX; i++) {
#pragma HLS pipeline II=1
            digit_histogram[i] = 0;
        }

    compute_histogram:
        for(int j = 0; j < num_symbols; j++) {
#pragma HLS PIPELINE II=1
            Digit digit = (sorting[j].frequency >> shift) & (RADIX - 1); // Extrract a digit
            current_digit[j] = digit;  // Store the current_digit for each symbol
            digit_histogram[digit]++;
            previous_sorting[j] = sorting[j]; // Save the current sorted order of symbols
        }

        digit_location[0] = 0;
    find_digit_location:
        for(int i = 1; i < RADIX; i++)
#pragma HLS PIPELINE II=1
            digit_location[i] = digit_location[i-1] + digit_histogram[i-1];

    re_sort:
        for(int j = 0; j < num_symbols; j++) {
#pragma HLS PIPELINE II=1
            Digit digit = current_digit[j];
            sorting[digit_location[digit]] = previous_sorting[j]; // Move symbol to new sorted location
            out[digit_location[digit]] = previous_sorting[j]; // Also copy to output
            digit_location[digit]++; // Update digit_location
        }
    }
}
```
![圖11.6：sort函數根據輸入符號的頻率值對其進行基數排序。](images/placeholder.png)

{% hint style='tip' %}
之前我們已經看到loop_tripcount指令向Vivado HLS提供 循環次數信息。使用assert（）語句有許多相同的用途，有一些優點和缺點。使用assert（）語句的一個優點是它們檢查仿真過程，並且這些信息可以用來進一步優化電路。然而，loop_tripcount指令隻影響性能分析，不是用於優化。另一方面，assert（）語句只能用於對變量值進行限制，但不能用於設置期望值或平均值，只能通過loop_tripcount指令完成。在大多數情況下，建議首先通過assert（）語句提供最差情況邊界，然後在必要時添加loop_tripcount指令。
{% endhint %}

radix_sort循環的主體被分成四個子循環，分別標記為init_histogram，compute_histogram,，find_digit_location和re_sort。init_histogram和compute_histogram結合基於當前考慮的數字來計算輸入的直方圖。這會產生每次每個數字在digit histogram[]中出現的數量。compute_histogram循環還存儲當前正在為current_digit[]中每個符號排序的數字。
接下來，find_digit_location循環計算所得直方圖值的前綴之和，把結果存在digit location[]。在計數排序的情況下，digit_location[]包含新排序數組中每個數字的第一個符號的位置。最後，re_sort循環根據這些結果對符號重新排序，將每個元素放置在新排序數組中的正確位置。它使用current_digit[]中存儲的密鑰從digit_location[]中選擇正確的位置。每次通過re_sort循環時，該位置都會遞增，以將下一個具有相同數字的元素放入排序數組中的下一個位置。總體而言，每次都是通過radix_sort循環迭代來實現對一個數字的計數排序。計數排序是一個穩定的排序，具有相同數字的元素保持相同的順序。在基於每個數字的穩定排序之後，數組以正確的最終順序返回。
前面我們在8.2和8.1章中討論過直方圖和前綴求和算法。在這種情況下，我們使用簡單的代碼和digit_histogram[]和digit_location[]完整的分解，就可以實現1的循環II來計算直方圖和前綴和，因為箱子的數量非常小。re_sort循環的優化與此類似。 由於唯一的循環是通過相對較小的digit_location[]數組，實現1的循環II也很簡單。 注意這種方法的工作原理主要是因為我們配置的RADIX相對較小。較大的RADIX值時，將digit_histogram[]和digit_location[]作為存儲是更好的，這可能需要額外的優化來實現1的循環II。
在此代碼的上下文中可能有意義的另一種替代方法是將digixPrase[]和digialPosith[]完全的分解與init_histogram和find_digit_location循環完全的展開結合起來。這些循環訪問這些小數組中的每個位置並使用最少量的邏輯進行操作。在這種情況下，儘管展開循環可能會導致為每個循環體複製電路，但實現此電路所需的資源較少，因為陣列訪問將處於固定索引。 但是，改變更大的值的BITS_PER_LOOP參數是禁止的，因為每個額外的位都會使RADIX參數翻倍，展開循環的成本也翻了一番。 這對於參數化代碼是一種常見的情況，不同的優化對於不同的參數值是有意義的。

{% hint style='tip' %}
在第8.2和第8.1章中提到的前綴求和和直方圖循環中，執行優化時，性能和利用率結果會發生什麼變化？在這種情況下，優化是必要的嗎？
{% endhint %}

{% hint style='tip' %}
re_sort循環是否能夠實現一個週期的特定啟動間隔？ 為什麼或為什麼不？
{% endhint %}

{% hint style='tip' %}
對於大數據集（n>256），圖11.6中代碼的近似延時（以n為單位）是多少？ 代碼的哪些部分支配週期數量？ 這將如何隨着RADIX參數的變化而改變？
{% endhint %}

請注意，re_sort循環不僅將排序的數組存儲在sorting[]中，而且還存儲排序的數組在out[]。 雖然這可能看起來多餘，但我們需要確保只寫入out[]以遵從頂層dataflow指令的要求。在這種情況下，out[]將會用部分排序的結果重寫多次，但只有最終結果傳遞給後面的函數。

{% hint style='tip' %}
dataflow指令對於執行任務級別流水線優化有幾個要求。其中之一就是需要單一的生產者和消費者之間的數據任務。 由於我們想在霍夫曼編碼過程執行如圖11.4所示的任務級別的流水線操作，我們必須確保每項任務都符合這一要求。 在sort函數中，這其中一個任務，它只能消耗（讀取但不寫入）輸入參數數據，並且只產生（寫入但不讀取）輸出
參數數據。 為了滿足這個要求，我們創建了內部sorting數組，它讀取和寫入整個函數。我們從函數開頭的參數in中複製輸入數據，並將最終結果寫入函數結尾處的輸出參數。 這確保我們遵循生產者/消費者dataflow指令的要求。
{% endhint %}

### 11.2.3 創建樹
霍夫曼編碼過程中的下一個功能形成了代表霍夫曼編碼的二叉樹。這在圖11.8所示的create_tree函數中實現。in[]包含num_symbols符號元素，按最低頻率到最高頻率排序。該函數創建一個這些符號的二叉樹,存儲在三個名為parent，left和right的輸出數組中。left和right數組表示樹中每個中間節點的左側和右側子結點。 如果子結點是一個葉結點，那麼左邊或右邊數組的相應元素將包含孩子的符號值，否則包含特殊符號INTERNAL_NODE。同樣，父數組保存每個中間結點的父結點的索引。樹的根結點的父結點被定義為索引零點。樹也是有序的，就像父母一樣總是比孩子有更高的指數。因此，我們可以很好地實現樹的自下而上和自上而下遍歷。
圖11.7顯示了這些數據結構的一個例子。 六個符號按其頻率排序並存儲在數組中。 霍夫曼樹的結果存儲在parent，left和right三個數組中。 另外，每個中間節點的頻率都存儲在frequency數組中。為了便於説明，我們直接表示左右數組的節點號（例如，n0，n1等）。這些將在現實中保存一個特殊的內部結點值。

{% hint style='tip' %}
雖然在這樣的樹中存儲複雜的數據結構可能是奇怪的，但在不允許數據分配的嵌入式編程中，這是非常普遍的[[53](./BIBLIOGRAPHY.md#53)]。事實上，malloc（）和free（）的 c 庫實現通常以這種方式實現低級內存管理，以便操作系統返回的更大的內存分配（通常稱為頁面）中創建小的分配，這使得操作系統能夠有效地管理內存的大量分配，並使用通常處理大型數據塊的處理器頁表和磁盤存儲器來協調虛擬內存。 4千字節是這些頁面的典型大小。有關使用數組實現數據結構的更多想法，請參見[[58](./BIBLIOGRAPHY.md#58)]。
{% endhint %}

在霍夫曼樹中，每個符號都與樹中的葉結點相關聯。樹中的中間節點通過將兩個符號以最小的頻率分組並使用它們作為新中間節點的左右結點來創建。 該中間結點的頻率是每個子結點的頻率之和。此過程通過迭代地創建具有最小頻率的兩個節點的中間節點，它可以包括其他的中間節點或葉節點。當所有中間結點都被併入二叉樹中時，樹的創建過程就完成了。
在代碼中有很多方法可以表示這個過程。例如，我們可以顯式地創建一個表示樹中每個節點按頻率排序的數組。 在這種情況下，選擇結點添加到樹中很簡單，因為它們將始終位於已排序數組的相同位置。 另一方面，將新創建的結點插入到列表中相對比較複雜，因為數組必須再次排序，還需要移動周圍的元素。或者，我們可以向數組中的數據結構添加指針式數組索引，以便在不實際移動數據的情況下邏輯地排序數據。 這會減少數據複製，但是會增加訪問每個元素的成本並需要額外的存儲空間。在數據結構設計中，許多常規算法權衡適用於HLS的上下文中，也適用於處理器中。
![圖11.7：符號數組in被用來創建霍夫曼樹，樹以圖形方式顯示，以及用於表示樹的四個數組的相應的值(intermediate,left, right, and parent)](images/huffman_create_tree.jpg)

然而，在這種情況下，我們可以做一些額外的簡化觀察。 最重要的觀察是，新的中間結點總是按頻率順序創建的。我們可以創建一箇中間節點，其頻率小於某個葉節點的頻率，但是我們將永遠不會創建一個比已經創建的中間節點更小的中間節點。這表明我們可以通過將節點存儲在兩個單獨的數組中來維護排序的數據結構：一個排序的符號數組和一個排序的中間節點數組。當我們使用每個列表中的最低頻率元素時，我們只需要追加到中間節點列表的末尾。還有一點額外的複雜性：因為我們可能需要從兩個數組中刪除零個，一個或兩個元素，但事實證明這比複製節點數組複雜得多。

{% hint style='tip' %}
從概念上講，這個算法與10.3節中討論的合共排序算法非常相似。關鍵的區別在於元素從排序後的數組中移除時的操作。 在合共排序中，最少的元素只是簡單地插入到合適的數組位置。 在這種情況下，兩個最小元素被識別，然後合併到一個新的樹結點中。
{% endhint %}

create_tree函數的實現代碼如圖11.8所示。 第一個代碼塊定義我們在函數中使用的局部變量。 frequency[]存儲每個被創建的中間節點的頻率， in_count跟蹤哪些符號已被賦予樹中的父節點，tree_count跟蹤哪些新創建的中間結點已經被賦予父結點。 通過主循環的每次迭代都會創建一個沒有父級的新中間節點，所以tree_count和i之間的所有中間節點都尚未被分配到樹的父節點中。

```c
#include "huffman.h"
#include "assert.h"
void create_tree (
    /* input */ Symbol in[INPUT_SYMBOL_SIZE],
    /* input */ int num_symbols,
    /* output */ ap_uint<SYMBOL_BITS> parent[INPUT_SYMBOL_SIZE-1],
    /* output */ ap_uint<SYMBOL_BITS> left[INPUT_SYMBOL_SIZE-1],
    /* output */ ap_uint<SYMBOL_BITS> right[INPUT_SYMBOL_SIZE-1]) {
    Frequency frequency[INPUT_SYMBOL_SIZE-1];
    ap_uint<SYMBOL_BITS> tree_count = 0;  // Number of intermediate nodes assigned a parent.
    ap_uint<SYMBOL_BITS> in_count = 0;    // Number of inputs consumed.

    assert(num_symbols > 0);
    assert(num_symbols <= INPUT_SYMBOL_SIZE);
    for(int i = 0; i < (num_symbols-1); i++) {
#pragma HLS PIPELINE II=5
        Frequency node_freq = 0;

        // There are two cases.
        // Case 1: remove a Symbol from in[]
        // Case 2: remove an element from intermediate[]
        // We do this twice, once for the left and once for the right of the new intermediate node.
        assert(in_count < num_symbols || tree_count < i);
        Frequency intermediate_freq = frequency[tree_count];
        Symbol s = in[in_count];
        if((in_count < num_symbols && s.frequency <= intermediate_freq) || tree_count == i) {
            // Pick symbol from in[].
            left[i] = s.value; // Set input symbol as left node
            node_freq = s.frequency; // Add symbol frequency to total node frequency
            in_count++; // Move to the next input symbol
        } else {
            // Pick internal node without a parent.
            left[i] = INTERNAL_NODE; // Set symbol to indicate an internal node
            node_freq = frequency[tree_count]; // Add child node frequency
            parent[tree_count] = i; // Set this node as child's parent
            tree_count++; // Go to next parentless internal node
        }

        assert(in_count < num_symbols || tree_count < i);
        intermediate_freq = frequency[tree_count];
        s = in[in_count];
        if((in_count < num_symbols && s.frequency <= intermediate_freq) || tree_count == i) {
            // Pick symbol from in[].
            right[i] = s.value;
            frequency[i] = node_freq + s.frequency;
            in_count++;
        } else {
            // Pick internal node without a parent.
            right[i] = INTERNAL_NODE;
            frequency[i] = node_freq + intermediate_freq;
            parent[tree_count] = i;
            tree_count++;
        }
        // Verify that nodes in the tree are sorted by frequency
        assert(i == 0 || frequency[i] >= frequency[i-1]);
    }

    parent[tree_count] = 0; //Set parent of last node (root) to 0
}
```
![圖11.8：霍夫曼樹的創建完整的代碼。 該代碼將已排序的符號數組in作為輸入，該數組中的元素數量為n，並輸出left, right和 parent三個數組的霍夫曼樹。](images/placeholder.png)

主循環包含兩個相似的代碼塊。每個代碼塊將[in_count].frequency中下一個可用符號的頻率與下一個[tree_count]可用的的中間結點的頻率相比較，然後選擇兩個最低頻率合併為一個新的中間結點的葉子。第一個代碼塊對新節點的左側子節點執行這個過程，並將結果存儲在left[i]中。第二個代碼塊選擇新節點的右側子節點執行這個過程，並將結果存儲在right[i]中。這兩種情況我們都需要小心確保這個比較是有意義的。 在第一次循環迭代中，tree_count==0和i==0，所以沒有有效的中間節點需要考慮，我們必須始終選擇一個輸入符號。在循環的最後迭代期間，所有的輸入符號可能都會被使用，因此in_count==num symbols，我們必須始終使用一箇中間節點。

循環的迭代次數以一種有趣的方式依賴於num_symbols的輸入。由於每個輸入符號都變成了二叉樹中的一個葉節點，我們知道會有num_symbols-1箇中間節點會被創建，因為這是二叉樹的基本屬性， 在循環結束時，我們將創建num_symbols-1個新節點，每個節點有兩個子節點。這些子結點的num_symbols將是輸入符號，num_symbols-2將是中間節點。在沒有父節點的情況下，將有一箇中間節點作為樹的根。這最後的節點在最後一行代碼中被賦予一個父索引0，這就完成了霍夫曼樹的創建。

樹中的中間節點的子節點可以是一個符號節點，也可以是中間節點。在創建霍夫曼樹時，這些信息並不重要，儘管稍後我們在遍歷樹時會很重要。如果相應的子節點是內部節點的話，為了存儲這個差別，一個特殊的值INTERNAL_NODE被存儲在left[]和right[]數組中。請注意，這個存儲本質上需要在數組中再加一位來表示。 結果，left[]和right[]數組會比你想象的大一點。

{% hint style='tip' %}
在圖11.8中，對於大數據集（n> 256），代碼的近似延時（以n為單位）是多少？
代碼的哪些部分支配週期數量？
{% endhint %}

### 11.2.4 計算比特長度
Compute_bit_length函數為每個符號確定了樹的深度。深度很重要，因為它決定了用於編碼的每個符號的位數。計算樹中每個節點的深度是使用下面的循環完成的：

$$
\begin{aligned}
\text{depth(root)} = 0 \\
\forall n! = \text{root}, \text{depth}(n) = \text{depth(parent}(n) + 1)      \\
\forall n, \text{child\_depth}(n) = \text{depth}(n) + 1
\end{aligned}
\qquad(11.1)
$$

可以通過遍歷從根節點開始的樹並按順序探索每個內部節點來計算這種循環。當我們遍歷每個內部節點時，我們可以計算節點的深度和任何子節點的相應的深度（增加1）。事實證明，我們實際上並不關心內部節點的深度，而只關心子節點的深度。下面的代碼對這種循環進行了計算：

$$
\begin{aligned}
\text{child\_depth(root)}  = 1 \\
\forall n! = \text{root}, \text{child\_depth}(n) = \text{chil\_depth(parent}(n) + 1)  
\end{aligned}
\qquad(11.2)
$$

這個函數的代碼如圖11.9所示。 函數的輸入參數表示霍夫曼樹的parent[], left[]和right[]。num_symbols包含輸入符號的數量，它比樹的中間節點的數量多一個。length_histogram[]數組的每個元素都存儲具有給定深度的符號數量。因此，如果有深度為3的5個符號，則length_histogram[3] = 5。
child_depth[]在遍歷樹時存儲每個內部節點的深度。每個內部結點的深度在traverse_tree 循環中被確定後，length_histogram[]被更新。Internal_length_histogram[]用於確保我們的功能符合dataflow指令的要求，其中輸出數組length_histogram[]永遠不會被讀取，init_histogram初始化這兩個數組。

{% hint style='tip' %}
init_histogram循環具有II = 1的pipeline指令。是否有可能滿足這個II？如果我們將II增加到更大的值，會發生什麼？ 如果我們不應用這個指令，會發生什麼？
{% endhint %}

樹中的內部節點從根節點遍歷，根節點具有最大的索引，直到索引為零。由於節點的數組是以自下而上的順序創建的，因此該反向順序導致樹的自頂向下遍歷，使得能夠在通過節點的單個路徑中計算每個節點的遞歸。對於每個節點，我們確定它的孩子的深度。然後，如果該節點存在有任何符號的孩子，那麼我們確定有多少孩子，並相應地更新直方圖。內部節點的子節點由特殊值INTERNAL_NODE來表示。

```c
#include "huffman.h"
#include "assert.h"
void compute_bit_length (
    /* input */ ap_uint<SYMBOL_BITS> parent[INPUT_SYMBOL_SIZE-1],
    /* input */ ap_uint<SYMBOL_BITS> left[INPUT_SYMBOL_SIZE-1],
    /* input */ ap_uint<SYMBOL_BITS> right[INPUT_SYMBOL_SIZE-1],
    /* input */ int num_symbols,
    /* output */ ap_uint<SYMBOL_BITS> length_histogram[TREE_DEPTH]) {
    assert(num_symbols > 0);
    assert(num_symbols <= INPUT_SYMBOL_SIZE);
    ap_uint<TREE_DEPTH_BITS> child_depth[INPUT_SYMBOL_SIZE-1];
    ap_uint<SYMBOL_BITS> internal_length_histogram[TREE_DEPTH];
 init_histogram:
    for(int i = 0; i < TREE_DEPTH; i++) {
        #pragma HLS pipeline II=1
        internal_length_histogram[i] = 0;
    }

    child_depth[num_symbols-2] = 1; // Depth of the root node is 1.

traverse_tree:
    for(int i = num_symbols-3; i >= 0; i--) {
#pragma HLS pipeline II=3
        ap_uint<TREE_DEPTH_BITS> length = child_depth[parent[i]] + 1;
        child_depth[i] = length;
        if(left[i] != INTERNAL_NODE || right[i] != INTERNAL_NODE){
            int children;
            if(left[i] != INTERNAL_NODE && right[i] != INTERNAL_NODE) {
                // Both the children of the original node were symbols
                children = 2;
            } else {
                // One child of the original node was a symbol
                children = 1;
            }
            ap_uint<SYMBOL_BITS> count = internal_length_histogram[length];
            count += children;
            internal_length_histogram[length] = count;
            length_histogram[length] = count;
        }
    }
}
```
![圖11.9：用於確定每個比特長度上符號數量的完整代碼。](images/placeholder.png)

{% hint style='tip' %}
對於大數據集（n> 256），圖11.9中代碼的近似延時（以n為單位）是多少？ 代碼的哪些部分支配週期數量？
{% endhint %}

{% hint style='tip' %}
此代碼有多個循環。 例如，一次循環發生是因為直方圖計算。 在這種情況下，該循環用II的3合成。如果你在pipeline指令中，目標是較低的II，會發生什麼？ 你可以重寫代碼來消除循環並達到較低的II嗎？
{% endhint %}

### 11.2.5 截斷樹

霍夫曼編碼過程的下一部分是重組具有大於MAX_CODEWORD_LENGTH所指定的深度的節點。這是通過尋找任何具有更大深度的符號，並將它們移動到比所指定的目標更小的級別來完成的。有趣的是，這完全可以通過操作符號深度直方圖來完成，只要直方圖以與原始樹上相同的模式一致的方式被修改。
輸入直方圖包含在input_length_histogram中，它是前面章節中compute_bit_length()函數導出的。有兩個相同的輸出數組truncated_length_histogram1和truncated_length histogram2。它們在稍後的過程中傳遞給兩個單獨的函數(canonize_tree and create_codewords)，因此我們必須使兩個數組遵守單個生產者，單個消費者的dataow指令的約束。

代碼如圖11.10所示。 copy_input循環從輸入數組input_length_ histogram複製數據。 move_nodes循環包含修改直方圖的大部分處理。 最後，input_length_histogram函數將內部結果複製到函數結尾的其他輸出。

{% hint style='tip' %}
copy_in for循環並未被優化。如果我們在這個循環中使用pipeline或unroll指令,延遲和啟動間隔會發生什麼？設計的總延時和啟動間隔會發生什麼變化？
{% endhint %}

該函數繼續在第二個節點移動 for循環中執行大部分循環計算。for循環是通過從最大的索引（TREE_DEPTH-樹的最大深度）迭代遍歷截斷長度直方圖數組開始的。這樣繼續下去直到找到一個非零元素或i達到MAX_CODEWORD_LENGTH的值。如果我們找不到一個非零元素，那意味着霍夫曼樹的初始輸入沒有深度大於目標深度的節點。換句話説，我們可以退出這個函數不用執行任何截斷。如果有一個大於目標深度的值，那麼該函數繼續重新組織樹，使所有節點的深度都小於目標深度，這是通過重新排序 while循環中的操作完成的。當有節點移動時，節點移動 for循環從深度最大的節點開始，然後繼續，直到所有節點重新排列的深度都小於目標深度。每個節點移動for循環的迭代都在一個深度的移動結點上進行。

```c
#include "huffman.h"
#include "assert.h"
void truncate_tree(
    /* input */ ap_uint<SYMBOL_BITS> input_length_histogram[TREE_DEPTH],
    /* output */ ap_uint<SYMBOL_BITS> output_length_histogram1[TREE_DEPTH],
    /* output */ ap_uint<SYMBOL_BITS> output_length_histogram2[TREE_DEPTH]
) {
    // Copy into temporary storage to maintain dataflow properties
 copy_input:
    for(int i = 0; i < TREE_DEPTH; i++) {
        output_length_histogram1[i] = input_length_histogram[i];
    }

    ap_uint<SYMBOL_BITS> j = MAX_CODEWORD_LENGTH;
 move_nodes:
    for(int i = TREE_DEPTH - 1; i > MAX_CODEWORD_LENGTH; i--) {
        // Look to see if there is any nodes at lengths greater than target depth
    reorder:
        while(output_length_histogram1[i] != 0) {
#pragma HLS LOOP_TRIPCOUNT min=3 max=3 avg=3
            if (j == MAX_CODEWORD_LENGTH) {
                // Find deepest leaf with codeword length < target depth
                do {
#pragma HLS LOOP_TRIPCOUNT min=1 max=1 avg=1
                    j--;
                } while(output_length_histogram1[j] == 0);
            }

            // Move leaf with depth i to depth j+1.
            output_length_histogram1[j] -= 1; // The node at level j is no longer a leaf.
            output_length_histogram1[j+1] += 2; // Two new leaf nodes are attached at level j+1.
            output_length_histogram1[i-1] += 1; // The leaf node at level i+1 gets attached here.
            output_length_histogram1[i] -= 2; // Two leaf nodes have been lost from level i.

            // now deepest leaf with codeword length < target length
            // is at level (j+1) unless j+1 == target length
            j++;
        }
    }

    // Copy the output to meet dataflow requirements and check the validity
    unsigned int limit = 1;
 copy_output:
    for(int i = 0; i < TREE_DEPTH; i++) {
        output_length_histogram2[i] = output_length_histogram1[i];
        assert(output_length_histogram1[i] >= 0);
        assert(output_length_histogram1[i] <= limit);
        limit * = 2;
    }
}
```
![圖11.10：重新組織霍夫曼樹的完整代碼，以便任何節點的深度低於參數MAX_CODEWORD_LENGTH指定的目標值。](images/placeholder.png)

重新排序 while循環在每次迭代中移動一個節點。第一個 if語句用於找到最大深度的葉節點。然後，我們將這個結點設為中間結點，並將其添加到比目標的深度大的葉節點作為子節點。這個if從句有一個do/while循環，從目標向下迭代，尋找truncated_length_histogram數組中非零的記錄。它的工作方式與開頭的節點移動for循環相似。當它發現最深的葉節點小於目標深度時就停止，該節點的深度存儲在j中。

現在我們有一個深度為i大於目標的節點，以及一個深度小於存儲在j中的目標的節點。我們將節點從深度i和節點從j移到到深度為j+1的子節點。因此，我們添加兩個符號到truncated_length_histogram[j+1]。我們正在生成一個深度為j的新的中間節點，因此我們從該級別去掉一個符號。我們移動另一個葉結點從深度i到深度i-1。我們從truncated_length_histogram[i]中減去2，因為其中1個節點進入級別j+1，另一級到達級別i-1。這些操作是在truncated_length_histogram數組上的四個語句中完成的。因為我們添加了一個符號到級別j+1，我們更新j，它擁有目標級別以下的最高級別，然後我們重複這個程序直到沒有深度大於目標的附加符號。

該函數是通過創建新的位長度的另一副本來完成的。這是通過存儲數組truncated_length_histogram1的更新位長度到truncated_length_histogram2中完成的。我們將這兩個數組傳遞給huffman_encoding頂層函數的最後的兩個函數，我們需要這兩個數組來確保滿足dataflow指令的約束。

### 11.2.6 Canonize樹
編碼過程的下一步是確定每個符號的位數，我們在圖11.11所示的canoniz_tree函數中執行此操作。該函數按排序順序的符號數組、符號總數（num_symbols）和霍夫曼樹直方圖的長度作為輸入。symbol_bits[]包含每個符號使用的編碼位數。 因此，如果具有值0x0A的符號以4位編碼，則symbol_bits[10] = 4。

canonization過程由兩個循環組成，分別是init_bits 和 process_symbols。 init_bits循環首先執行，初始化symbol_bits[]數組為0。然後process_symbols循環按照從最小頻率到最大頻率的排序順序處理符號。通常，頻率最低的符號被分配最長的碼，而頻率最高的符號被分配最短的碼。 每次通過process_symbols循環時，我們都分配到一個符號的長度。符號的長度由內部do / while循環決定，它遍歷長度直方圖。該循環查找尚未分配碼字的最大位長，並將該長度中的碼字數存儲在count中。 每次通過外循環，count遞減，直到我們用完碼字。 當count變為零時，內部do/while循環再次執行去尋找要分配的代碼字的長度。

```c
#include "huffman.h"
#include "assert.h"
void canonize_tree(
    /* input */ Symbol sorted[INPUT_SYMBOL_SIZE],
    /* input */ ap_uint<SYMBOL_BITS> num_symbols,
    /* input */ ap_uint<SYMBOL_BITS> codeword_length_histogram[TREE_DEPTH],
    /* output */ CodewordLength symbol_bits[INPUT_SYMBOL_SIZE] ) {
    assert(num_symbols <= INPUT_SYMBOL_SIZE);

 init_bits:
    for(int i = 0; i < INPUT_SYMBOL_SIZE; i++) {
        symbol_bits[i] = 0;
    }

    ap_uint<SYMBOL_BITS> length = TREE_DEPTH;
    ap_uint<SYMBOL_BITS> count = 0;

    // Iterate across the symbols from lowest frequency to highest
    // Assign them largest bit length to smallest
 process_symbols:
    for(int k = 0; k < num_symbols; k++) {
        if (count == 0) {
            //find the next non-zero bit length
            do {
#pragma HLS LOOP_TRIPCOUNT min=1 avg=1 max=2
                length--;
                // n is the number of symbols with encoded length i
                count = codeword_length_histogram[length];
            }
            while (count == 0);
        }
        symbol_bits[sorted[k].value] = length; //assign symbol k to have length bits
        count--; //keep assigning i bits until we have counted off n symbols
    }
}
```
![圖11.11：完整的canonizing霍夫曼樹代碼，它定義了每個符號的位數](images/placeholder.png)

```c
int k = 0;
process_symbols:
for(length = TREE_DEPTH; length >= 0; length--) {
    count = codeword_length_histogram[length];
    for(i = 0; i < count; i++) {
#pragma HLS pipeline II=1
        symbol_bits[sorted[k++].value] = length;
    }
 }
 ```
![圖11.12：圖11.11中process_symbols循環的替代循環結構。](images/placeholder.png)

{% hint style='info' %}
process_symbols循環無法進行流水線操作，因為內部do / while循環無法展開。 這有點尷尬，因為內部循環通常只執行一次，步進到直方圖中的下一個長度。 只有在極少數情況下，例如我們碰巧沒有分配任何代碼字，內部循環需要執行多次。 在這種情況下，沒有太多的損失，因為循環中的所有操作都是簡單的操作，除了內存操作之外，這些操作不可能是流水線操作。 還有其他的方式來構建這個循環，但是，它可以是流水線的。 一種可能性是使用外部for循環迭代codeword_length_histogram[]和內部循環來計算每個符號，如圖11.12所示。
{% endhint %}

{% hint style='tip' %}
實現圖11.11中的代碼和圖11.12中的替代代碼結構。 哪一個會實現更高的性能？ 哪種編碼風格對你來説更自然？
{% endhint %}

### 11.2.7 創建碼字
編碼過程中的最後一步是為每個符號創建碼字。 該流程根據Canonical霍夫曼代碼的特性，按順序簡單地分配每個符號。第一個特性是具有相同長度前綴的較長的代碼比短碼有更高的數值。第二個特性是相同長度的代碼隨着符號值的增加而增加。為了在保持代碼簡單的同時實現這些屬性，確定每個長度的第一個碼字是有用的。如果我們知道由碼字長度直方圖給出的每個長度的碼字的數量，則可以使用以下循環來得出結果：
$$
\begin{aligned}
&\text{first\_codeword}(1) = 0 \\
&\forall i>1,\text{first\_codeword}(i) = \\
&(\text{first\_codeword}(i-1) + \text{codeword\_length\_histogram}(i-1))\ll1
\end{aligned}
\qquad(11.3)
$$
實際上，不是一個接一個地分配碼字，而是這個循環首先分配了所有的代碼字。這使我們能夠按照符號值的順序來分配碼字，而不用擔心按長度或頻率對它們進行排序。
除了將碼字分配給符號外，我們還需要對碼字進行格式化，使得它們可以很容易地用於編碼和解碼。 使用霍夫曼編碼的系統通常以位反轉的順序存儲代碼字。 這使得解碼過程更容易，因為從根節點到葉結點，比特流被按照與解碼期間樹遍歷相同的順序存儲。

實現create_codewords函數的代碼如圖11.13所示。Symbol_bits[]包含每個符號的碼字和長度，codeword_length_histogram[]包含每個長度的碼字數量，encoding[]的輸出表示每個符號的編碼。每個元素由實際的碼字和每個碼字的長度打包在一起組成。碼字的最大長度由MAX_CODEWORD_LENGTH參數給定。反過來，這又決定了保持碼字所需的位數，它由CODEWORD_LENGTH BITS給定。 CODEWORD_LENGTH_BITS編碼數組中每個元素的最低有效位包含從輸入數組symbol_bits接收到的相同值。每個編碼元素的高位MAX_CODEWORD_LENGTH包含實際碼字。使用27位MAX_CODEWORD_LENGTH導致CODEWORD_LENGTH_BITS為5是一個特別有用的組合，因為encoding[]中的每個元素滿足一個32位字長度。

該代碼主要由兩個循環組成，分別是first_codewords和assign_codewords。first_codewords循環尋找每個長度的第一個碼字，並執行公式11.3的循環。 assign_codewords循環最終將每個符號與每個碼字相關聯。

碼字是通過每個碼字的長度並索引到first_codeword[]的正確元素中找到的。 該代碼的主要複雜性在於位反轉過程，它是基於bit_reverse32函數。 我們之前在FFT章節中已經討論了這個函數（參見第5.3章），所以我們不會在這裏再討論它。 在反轉碼字中的位之後，下一個語句刪除最不重要的'0'位，只留下位反轉的碼字，然後將位反轉的碼字與低位中的符號的長度一起打包在高位中，並存儲在encoding[]中。 最後，first_codeword[]中的值是遞增的。

{% hint style='tip' %}
在圖11.13的代碼中，輸入實際上包含一些宂餘信息。特別是，我們可以使用直方圖來計算每個碼字symbol_bits[]的長度，計算存儲在codeword_length_histogram[]中每個比特長度的符號數。 相反，在此代碼中，我們選擇複用最初計算的直方圖的truncate_tree()函數。 這樣，我們可以通過重新計算直方圖來節省存儲空間。這是一個很好的考量嗎？ 在這個函數中，需要多少資源來計算直方圖？ 通過流水線傳遞直方圖需要多少資源？
{% endhint %}

{% hint style='tip' %}
估計圖11.13中代碼的延遲。
{% endhint %}

```c
#include "huffman.h"
#include "assert.h"
#include <iostream>
void create_codeword(
  /* input */ CodewordLength symbol_bits[INPUT_SYMBOL_SIZE],
  /* input */ ap_uint<SYMBOL_BITS> codeword_length_histogram[TREE_DEPTH],
  /* output */ PackedCodewordAndLength encoding[INPUT_SYMBOL_SIZE]
) {
    Codeword first_codeword[MAX_CODEWORD_LENGTH];

    // Computes the initial codeword value for a symbol with bit length i
    first_codeword[0] = 0;
 first_codewords:
    for(int i = 1; i < MAX_CODEWORD_LENGTH; i++) {
#pragma HLS PIPELINE II=1
        first_codeword[i] = (first_codeword[i-1] + codeword_length_histogram[i-1]) << 1;
        Codeword c = first_codeword[i];
        //        std::cout << c.to_string(2) << " with length " << i << "\n";
    }

 assign_codewords:
  for (int i = 0; i < INPUT_SYMBOL_SIZE; ++i) {
#pragma HLS PIPELINE II=5
      CodewordLength length = symbol_bits[i];
      //if symbol has 0 bits, it doesn't need to be encoded
  make_codeword:
      if(length != 0) {
          //          std::cout << first_codeword[length].to_string(2) << "\n";
          Codeword out_reversed = first_codeword[length];
          out_reversed.reverse();
          out_reversed = out_reversed >> (MAX_CODEWORD_LENGTH - length);
          // std::cout << out_reversed.to_string(2) << "\n";
          encoding[i] = (out_reversed << CODEWORD_LENGTH_BITS) + length;
          first_codeword[length]++;
      } else {
          encoding[i] = 0;
      }
  }
}
```
![圖11.13:為每個符號生成canonical霍夫曼碼字的完整代碼。可以在知道每個符號的比特數的情況下計算碼字（存儲在輸入數組symbol_bits[]中）。另外，我們有另一個輸入數組codeword_length_histogram[]，其在每個條目中存儲具有該位長度的碼字的符號數，輸出是存儲在encoding[]數組中的每個符號的代碼字。](images/placeholder.png)

現在讓我們來看看我們的運行示例，並説明如何使用它來導出最初的碼字。 在這個例子中，符號A，D和E對於它們的編碼有兩比特; 符號C有三比特; 而符號B和F有四比特。 因此，我們有：

$$
\begin{aligned}
\text{bit\_length}(1) &= 0 \\
\text{bit\_length}(2) &= 3 \\
\text{bit\_length}(3) &= 1 \\
\text{bit\_length}(4) &= 2
\end{aligned}
\qquad(11.4)   
$$

使用公式11.3來計算第一個碼字的值，我們定義：
$$
\begin{aligned}
\text{first\_codeword}(1) &= 0 &= 0b0   \\  
\text{first\_codeword}(2) &= (0 + 0) \ll 1 &= 0b00 \\  
\text{first\_codeword}(3) &= (0 + 3) \ll 1 &= 0b110 \\  
\text{first\_codeword}(4) &= (6 + 1) \ll 1 &= 0b1110
\end{aligned}
\qquad(11.5)
$$
一旦我們確定了這些值，然後從小到大依次考慮每個符號。 對於每個符號，確定其碼字的長度並分配下一個碼字適當的長度。 在運行示例中，考慮符號按字母順序A，B，C，D，E和F排列， 符號A有兩比特用於編碼， 執行查找first_codeword[2]=0。
因此，分配給A的碼字是0b00，我們增加first_codeword[2]的值到1。符號B有四個比特。 由於first_codeword[4]=14=0b1110，所以碼字是0b1110。 符號C有三個比特。 first_codeword[3]=6=0b110，碼字是110。符號D具有兩比特，因此first_codeword[2]=1=0b01；請記住，我們在將碼字分配給符號A後這個值將遞增。符號E有兩位，所以它的碼字是0b01+1=0b10。F有4比特，所以它的碼字是0b1110+1=0b1111。
所有符號的最終碼字是：
$$
\begin{aligned}
\text{A} &\rightarrow \quad00  \\
\text{B}  &\rightarrow \quad1110 \\
\text{C}  &\rightarrow \quad110    \\
\text{D}  &\rightarrow \quad01 \\
\text{E}  &\rightarrow \quad10  \\  
\text{F}  &\rightarrow \quad1111  
\end{aligned}
\qquad(11.6)  
$$

### 11.2.8 測試平台
代碼的最後部分是測試平台，如圖11.14所示。 總體結構是從文件中讀取輸入的頻率值，使用huffman_encoding函數處理它們，並將結果碼字與存儲在文件中的現有黃金參考進行比較。
main()函數首先創建從文件中讀取頻率所需的變量（在這裏，文件是huffman.random256.txt）並將它們放入in[]中，這是在file_to_array函數中完成的，該函數將輸入數據的文件名和數據長度（數組長度）作為輸入，並將該文件中的輸入存儲到array[]變量中。 這個文件包含了每個符號的頻率， 頻率值按照符號順序存儲，因此文件第一個值表示符號'0'的頻率，依此類推。

main()函數通過使用文件中的頻率在in[]中進行初始化，然後它會調用頂層的霍夫曼函數。該函數返回encoding[]中的編碼符號值。由於處理結果應該是一個前綴碼，因此我們檢查前綴碼的屬性是否確實得到滿足，然後將結果與存儲在文件huffman.random256.gold中的黃金參考中的代碼字進行比較。然後將結果寫入名為random256.out的文件中，並使用diff工具執行文件比較操作。如果文件相同，diff工具返回'0'，如果文件不同則返回非零。因此，如果條件不同時，則執行if條件，而當文件相同時，執行else條件。在這兩種情況下，我們都會打印一條消息並將return_val設置為適當的值。Vivado RHLS工具在協同仿真過程中使用該返回值來檢查結果的正確性。如果檢查通過，則返回值為0;如果不通過，則返回值不為零。

```c
#include "huffman.h"
#include <stdio.h>
#include <stdlib.h>

void file_to_array(const char *filename, ap_uint<16> *&array, int array_length) {
    printf("Start reading file [%s]\n", filename);
    FILE * file = fopen(filename, "r");
    if(file == NULL) {
        printf("Cannot find the input file\n");
        exit(1);
    }

    int     file_value = 0;
    int     count = 0;
    array = (ap_uint<16> * ) malloc(array_length*sizeof(ap_uint<16>));

    while(1) {
        int eof_check = fscanf(file, "%x", &file_value);
        if(eof_check == EOF) break;
        else {
            array[count++] = (ap_uint<16>) file_value ;
        }
    }
    fclose(file);

    if(count != array_length) exit(1);
}

int main() {
    printf("Starting canonical Huffman encoding testbench\n");
    FILE * output_file;
    int return_val = 0;
    ap_uint<16> * frequencies = NULL;
    file_to_array("huffman.random256.txt", frequencies, INPUT_SYMBOL_SIZE);

    Symbol in[INPUT_SYMBOL_SIZE];
    for (int i = 0 ; i <  INPUT_SYMBOL_SIZE; i++) {
        in[i].frequency = frequencies[i];
        in[i].value = i;
    }

    int num_nonzero_symbols;
    PackedCodewordAndLength encoding[INPUT_SYMBOL_SIZE];
    huffman_encoding(in, encoding, &num_nonzero_symbols);

    output_file = fopen("huffman.random256.out", "w");
    for(int i = 0; i < INPUT_SYMBOL_SIZE; i++)
        fprintf(output_file, "%d, %x\n", i, (unsigned int) encoding[i]);
    fclose(output_file);

    printf ("\n***************Comparing against output data*************** \n\n");
    if (system("diff huffman.random256.out huffman.random256.golden")) {
        fprintf(stdout, "*******************************************\n");
        fprintf(stdout, "FAIL: Output DOES NOT match the golden output\n");
        fprintf(stdout, "*******************************************\n");
        return_val = 1;
    } else {
        fprintf(stdout, "*******************************************\n");
        fprintf(stdout, " PASS: The output matches the golden output\n");
        fprintf(stdout, "*******************************************\n");
        return_val = 0;
    }

    printf("Ending canonical Huffman encoding testbench\n");
    return return_val;
}
```
![圖11.14：完整的canonical霍夫曼編碼測試平台代碼。該代碼使用輸入文件中的數據初始化in數組，並將它傳遞給頂層的huffman_encoding函數，然後它將結果碼字存儲到一個文件中，並將其與另一個黃金參考文件進行比較，最後打印出比較結果，並返回適當的值](images/placeholder.png)

## 11.3 結論
霍夫曼編碼是許多應用中常用的數據壓縮類型。 雖然使用霍夫曼碼進行編碼和解碼是相對簡單的操作，但生成霍夫曼碼本身可能是計算上具有挑戰性的問題。 在許多系統中，擁有相對較小的數據塊是有利的，這意味着必須經常創建新的霍夫曼碼,使之值得被加速。與我們在本書中研究過的其他算法相比，創建霍夫曼碼包含許多具有完全不同代碼結構的步驟。 有些相對容易實現並行化，而另一些實現起來則更具有挑戰性。 算法的某些部分自然具有較高的O(n)複雜度，這意味着它們必須更加並行化以實現平衡的流水線。但是，使用Vivado HLS中的數據流指令，這些不同的代碼結構可以相對容易地鏈接在一起。
