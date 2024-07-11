# 第九章 視頻系統

## 9.1 背景

​視頻處理是目前FPGA上常見的應用。其中一個選擇FPGA處理視頻應用的原因是目前的FPGA時鐘處理頻率可以很好地滿足常見視頻數據對處理速率的要求。例如，一般的高清視頻格式，全高清視頻格式（1080p@60HZ）需要的處理速率是`1920（像素/行）x 1080（行/幀）x 60（幀/秒） =  124,416,000（像素/秒）`。

​當對數字視頻流（1080p@60HZ）進行編碼時(1080p視頻幀實際傳輸時像素大小為2200x1125，其中行2200像素 = 行同步+行消隱前肩+有效像素（1920像素）+行消隱後肩，場1125 行=場同步+場消隱前肩+有效行（1080行）+ 場消隱後肩 ），其中消隱像素也會伴隨有效像素以148.5MHz（1080P@60HZ）的頻率在FPGA流水線電路中進行處理。為了達到更高的處理速率，可以通過在每個時鐘週期內處理多個採樣數據的方式實現。數字視頻是如何傳輸的詳細內容介紹將在9.1.2小節中展開。另一個選擇FPGA處理視頻應用需求的原因主要是因為視頻按**掃描線順序**從左上角像素到右下角像素的順序進行處理，如圖9.1所示。這種可預測的視頻數據處理順序允許FPGA在無需消耗大量存儲器件的條件下構建高效的數據存儲架構，從而有效地處理視頻數據。這些體系結構的詳細信息將在第9.2.1小節中介紹。

![圖 9.1: 視頻幀的掃描處理順序](images/videoScanlineOrder.jpg)

​視頻處理應用是HLS一個非常好的目標應用領域。首先，視頻處理通常是可以容忍處理延遲的。儘管一些應用可能會將整體延遲限制到一幀之內，但是許多視頻處理應用中是可以容忍若干幀的處理延遲。因此，在HLS中使用接口和時鐘約束方式實現的高效的流水線處理，可以不用考慮處理時延問題。其次，視頻算法往往非常不統一，算法一般是基於算法專家的個人喜好或擅長的方式被開發的。一般情況下它們是被使用可以快速實現和仿真的高級語言開發的。雖然全高清視頻處理算法在FPGA處理系統上可以以每秒60幀處理速度運行或者是在筆記本電腦上通過使用綜合的C/C++代碼以每秒一幀的處理速度運行是很常見的，但是RTL級仿真時僅能一個小時一幀的速度運行。最後，視頻處理算法通常以適合HLS容易實現的嵌套循環編程風格編寫。這意味着來自算法開發人員編寫的許多視頻算法的C / C ++原型代碼均可以被綜合成相應的FPGA電路。

### 9.1.1 視頻像素表示

​許多視頻輸入和輸出系統都是圍繞人類視覺系統感知光線的方式進行優化的。第一個原因就是眼球中的視錐細胞對紅、綠和藍光敏感。其他顏色都可以視為紅、綠和藍色的融合。因此，攝像機和顯示器模仿人類視覺系統的功能，對紅、綠、藍光采集或者顯示紅、綠和藍像素，並用紅色、綠色和藍色分量的組合來表示顏色空間。最常見的情況是每個像素使用24位數據位表示，其中紅綠藍每個分量各佔8位數據位，其中顏色分量的數據位的位數也存在其他情況，例如高端系統中的每個像素顏色分量可以佔10位位深或甚至12位位深。

​第二原因是人類視覺系統對亮度比對顏色更敏感。因此，在一個視頻處理系統內，通常會將RGB顏色空間轉換到到YUV顏色空間，它將像素描述為亮度（Y）和色度（U和V）的分量組合，其中色度分量 U和V是獨立於亮度分量Y的。例如常見視頻格式YUV422，就是對於4個像素點，採樣4個Y的值，兩個U的值，兩個V的值。這種格式應用於視頻壓縮時的採樣叫**色度子採樣**。另一種常見的視頻格式YUV420表示採樣4個像素值由4個Y值，一個U值和一個V值組成，進一步減少了表示圖像所需像素數據。視頻壓縮通常在YUV色彩空間中進行。

​第三個方面是由於眼睛中的眼杆和感光部分對綠光比對紅光更加敏感，並且大腦主要從綠光中獲取亮度信息。因此，圖像傳感器和顯示器採用馬賽克模式，比如採樣2個綠色像素和1個紅像素與1個藍像素比例的**Bayer**模式。於是在相同數量焦平面採集單元或顯示器顯示單元條件下系統可以採集或顯示更高分辨率的圖像，從而降低圖像採集傳感器或顯示器的製造成本。

{% hint style='info' %}視頻系統多年來一直圍繞人類視覺系統進行設計。最早的黑白相機對藍綠光敏感，以匹配眼睛對該顏色範圍內亮度的敏感度。然而，不幸的是它們對紅光不太敏感，因此紅色（如紅粧）在相機上看起來不正常。當時針對這個問題的解決方案絕對是低技術含量的：拍照時演員穿着華麗的綠色和藍色化的粧。 {% endhint %}

### 9.1.2 數字視頻格式

​除了表示個別的像素之外，數字視頻格式必須以視頻幀的形式對像素進行組織和編碼。在很多情況下，通過同步或**同步**信號的方式來表示連續像素序列的視頻幀開始和結束。在某些視頻接口標準（如數字視頻接口或**DVI**）中，同步信號使用單獨的物理線路引出。在其他標準（如數字電視標準BTIR 601/656）中，同步信號的開始和結束由不會出現在視頻信號中的特殊像素值表示。

​相鄰行像素之間（從左到右順序掃描）由**水平同步信號**分隔開，且相鄰行像素之間的水平同步信號由若干時鐘週期的脈衝信號組成。另外，在水平同步信號和有效像素信號中間還有若干時鐘週期的無效信號，這些信號稱為**行前肩**和**行後肩**。同樣，相鄰幀信號之間（從上到下順序掃描）由**垂直同步脈衝**分隔。每個視頻幀之間有若干行有效的幀同步信號。注意，相鄰幀之間的垂直同步信號僅在視頻幀的開始出現。同樣，幀視頻信號中除了同步信號和視頻有效信號外，還有無效視頻行組成的**場前肩**和**場後肩**信號。另外，大多數數字視頻格式包含表示有效視頻像素起始的數據使能信號。因此，所有無效的視頻像素一起組成**水平消隱間隔**和**垂直消隱間隔**。這些信號如圖9.2所示。

![圖9.2：1080P@60Hz高清視頻信號中的典型的同步信號](images/video_syncs.jpg)

{% hint style='info' %}數字視頻信號均是對原來電視標準的模擬信號（美國的NTSC和許多歐洲國家的PAL）通過抽樣、量化和編碼後形成。由於用於模擬陰極射線管掃描的硬件具有有限的電平轉換速率，所以水平和垂直同步間隔的時間要滿足從硬件恢復掃描到下一行的開始。同步信號由若干個時鐘週期的低電平組成。另外，由於電視無法有效地顯示靠近同步信號的像素，因此引入前肩信號和後肩信號，這樣就可以顯示更多的圖片像素。即使如此，由於許多電視機都是基於**電子束掃描**原理設計的，所以其中的圖像數據有20％的邊框處的像素是不可見。{% endhint %}

​如圖9.2所示，典型1080P（60Hz）視頻幀共包含2200 * 1125個數據採樣像素點。在每秒60幀的情況下，這相當於每秒總共148.5百萬的採樣像素點。這比1080P（60Hz）幀中的有效視頻像素的平均速率（1920 * 1080 * 60 =每秒124.4百萬像素）高很多。現在的大多數FPGA都可以滿足以這個時鐘速率進行高效的處理，並且是在一個時鐘週期進行一次採樣的方式下運行的。對於更高分辨率的處理系統需求，如4K或2K的數字影院，或者每秒120幀甚至240幀的處理需求，就需要一個時鐘週期採樣更多像素點，通過增加數據處理的吞吐量方式來運行。注意，我們通常可以通過在HLS中展開循環的方式來生成此類結構（請參見1.4.2小節）。同樣，當處理低分辨率和低幀率需求時，優選的方案是採用操作共享的方式，多個時鐘週期去處理一次採樣。這樣的處理結構是通過指定循環的處理間隔方式實現。

```c
#include ”video_common.h”
unsigned char rescale(unsigned char val, unsigned char offset, unsigned char scale) {
    return ((val − offset) ∗ scale) >> 4;
}

rgb_pixel rescale_pixel(rgb_pixel p, unsigned char offset, unsigned char scale) {
    #pragma HLS pipeline
    p.R = rescale(p.R, offset, scale);
    p.G = rescale(p.G, offset, scale);
    p.B = rescale(p.B, offset, scale);
    return p;
}

void video_filter_rescale(rgb_pixel pixel_in[MAX_HEIGHT][MAX_WIDTH],
rgb_pixel pixel_out[MAX_HEIGHT][MAX_WIDTH],
unsigned char min, unsigned char max) {
    #pragma HLS interface ap_hs port = pixel_out
    #pragma HLS interface ap_hs port = pixel_in
    row_loop:
    for (int row = 0; row < MAX_WIDTH; row++) {
        col_loop:
        for (int col = 0; col < MAX_HEIGHT; col++) {
            #pragma HLS pipeline
            rgb_pixel p = pixel_in[row][col];
            p = rescale_pixel(p,min,max);
            pixel_out[row][col] = p;
		}
	}
}
```

![圖9.3 簡單視頻濾波器的實現代碼](images/placeholder.png)

​例如，在圖9.3中代碼展示了一個簡單的視頻處理應用程序，程序中執行每次循環的II = 1(Initiation interval),也就是一個時鐘週期處理一次採樣數據。代碼中採用嵌套循環的方式是按照圖9.1中所示的掃描線順序處理圖像中的像素。II = 3的設計方式可以通過共享計算組件的方式來減少資源使用量。通過設置內層循環展開因子為2和輸入輸出數組拆分因子為2就可以滿足一個時鐘處理兩個像素。這種例子相對簡單，因為每個組件和每個像素的處理是相對獨立的。更復雜的功能可能不會因為資源共享而處理更方便，也可能無法同時處理多個像素。

![圖9.4  將視頻處理設計與BRAM接口集成](images/video_BRAM_interface.jpg)

```c++
void video_filter(rgb_pixel pixel_in[MAX_HEIGHT][MAX_WIDTH],
rgb_pixel pixel_out[MAX_HEIGHT][MAX_WIDTH]) {
	#pragma HLS interface ap_memory port = pixel_out // The default
	#pragma HLS interface ap_memory port = pixel_in // The default
}
```

{% hint style='info' %}高速計算機視覺應用是需要滿足每秒處理10000幀的200*180像素的視頻幀的速度要求。這樣的應用通常是使用一個高速圖像傳感器直接和FPGA直接連接，並且中間不需要有同步信號。在這種情況下，你會每個時鐘週期內進行多少次採樣處理呢？FPGA可以很好的完成處理麼？答案是你可以使用HLS編寫嵌套循環的結構來完成這樣的設計。{% endhint %}

### 9.1.3 視頻處理系統架構

​到目前為止，我們專注於構建視頻處理應用部分的程序而不關心如何將其整合到一個大的系統程序中去。在很多情況下，如圖9.1.2中示例代碼，大部分像素處理髮生在循環內，並且當循環運行時每個時鐘週期僅處理一個像素。在本節中，我們將討論講視頻處理部分程序集成到大的系統程序中的情況。

​默認情況下，Vivado@HLS軟件會將代碼中的函數數組接口綜合成簡單的存儲器接口。其中，存儲器寫數據接口由地址總線、數據總線和寫使能信號線組成。存儲器每次讀取和寫入數據均有相應地址變化，並且數據讀取和寫入時間固定。如圖9.4所示，將這種接口與片上存儲資源Block RAM資源集成在一起使用很簡單方便。但是即使在大容量的芯片，片上Block RAM資源也是很稀缺的，因此如果存儲大容量的視頻資源會很快消耗完片上Block RAM資源。

{% hint style='info' %}針對每像素24位的1920x1080視頻幀，芯片需要消耗多少BlockRAM資源才能完成存儲存一幀視頻幀？片上Block RAM資源最多又能存儲多少幀呢？{% endhint %}

![圖9.5 將視頻處理設計與外部存儲接口集成](images/video_DDR_interface.jpg)

```c
void video_filter(pixel_t pixel_in[MAX_HEIGHT][MAX_WIDTH],
pixel_t pixel_out[MAX_HEIGHT][MAX_WIDTH]) {
   		#pragma HLS interface m_axi port = pixel_out
   		#pragma HLS interface m_axi port = pixel_in
}
```

​通常，大部分視頻系統更好的選擇是將視頻幀存儲在外部存儲器中，比方説DDR。在圖9.5中展示了系統與外部存儲器典型系統集成模式。在FPGA內部，**外部存儲器的控制器**（MIG）集成了ARM AXI4標準化從接口與FPGA內的其他模塊進行通信。FPGA內的其它模塊使用AXI4主接口或者是通過專門的**AXI4內部互聯模塊**與外部存儲器的AXI4從接口相連。AXI內部互聯模塊允許多個主設備模塊訪問多個從設備模塊。該架構抽象了外部存儲器的操作細節，允許在無需修改FPGA設計中其它模塊的情況下，允許不同外部存儲器模塊和標準互換使用。

​儘管大多數處理器都使用高速緩存，從而高性能處理數據。但通常情況下，如圖9.5所示，基於FPGA實現的視頻處理系統，無需片上高速緩存。在處理器系統中，高速緩存可以提供對先前訪問的數據更低時延的訪問，並通過始終讀取或寫入完整的高速緩存行來提高對外部存儲器訪問的帶寬。一些處理器甚至還使用更復雜的數據處理機制，如預取和推測讀取，以減少對外部存儲器的訪問時延和增加對外部存儲器的訪問帶寬。對於大多數基於FPGA的視頻處理系統，因為大多數視頻算法訪問外部存儲器是可預測的，所以使用線性緩衝區和窗口緩衝區可以一次從外部緩衝區中讀取多個數據。另外,當以猝發模式對外部存儲器訪問時，Vivado@HLS能夠提前調度地址變換，避免讀取數據時拖延計算。

![圖9.6:通過DMA將視頻處理設計模塊與外部存儲器接口集成](images/video_DDR_DMA_interface.jpg)

```c
void video_filter(pixel_t pixel_in[MAX_HEIGHT][MAX_WIDTH],
pixel_t pixel_out[MAX_HEIGHT][MAX_WIDTH]) {
  #pragma HLS interface s_axi port = pixel_out
  #pragma HLS interface s_axi port = pixel_in
```

```c
void video_filter(pixel_t pixel_in[MAX_HEIGHT][MAX_WIDTH], pixel_t pixel_out[MAX_HEIGHT][MAX_WIDTH]) {
	#pragma HLS interface ap_hs port = pixel_out
	#pragma HLS interface ap_hs port = pixel_in

  video_filter(hls::stream<pixel_t> &pixel_in, hls::stream<pixel_t> &pixel_out)
}
```

![圖9.7: HLS中流接口的代碼實現方式](images/placeholder.png)

如圖9.6所示為另一種存儲器體系架構。在這種架構中，算法處理模塊通過外部直接存儲器（DMA）與外部存儲器連接，DMA完成與外部存儲控制器（MIG）之間地址轉換的細節操作。算法處理模塊可以以AXI總線數據流（AXIS）的形式將數據傳輸給DMA控制器。DMA假設數據是算法處理模塊生成的，並將數據寫入外部存儲器。在Vivado@HLS中，有多種編碼方式可以將代碼中的數組接口綜合成AXIS流接口，如圖9.7所示是其中一種。在這種情況下，C代碼與前面看到的代碼相同，但接口約束指令不同。另外一種代碼實現方法是使用`hls::stream<>`方式顯式建立總線流接口。無論那種情況都必須注意，DMA中生成數據的順序要與C代碼中訪問數據的順序相同。

​AXIS數據流接口的一個優點是它允許在設計系統中，採用多個算法處理模塊級聯的方式，並且不需要在外部存儲器中存儲算法處理計算中間結果。在一些應用例子中，如圖9.8所示，FGPA系統在沒有外部存儲器的情況下，將從輸入接口（如HDMI）上接收到的像素數據處理後直接將它們發送到輸出接口。這樣的設計通常要求算法處理模塊的吞吐量必須達到需求，以滿足外部接口嚴格實時約束的要求。當一些複雜的算法構建時，系統難以保證其數據吞吐量需求，因此係統如果至少能提供一幀緩衝區，就可以為算法構建提供更大的靈活性。當輸入和輸出像素速率不同或者可能不相關時（例如接收任意輸入視頻格式並輸出不同任意視頻格式的系統），幀緩衝可以起到簡化構建系統的作用。

![圖9.8：將視頻處理設計與流接口集成](images/video_streaming_interface.jpg)

```c
void video_filter(pixel_t pixel_in[MAX_HEIGHT][MAX_WIDTH],
 pixel_t pixel_out[MAX_HEIGH][MAX_HEIGHT]) {
		#pragma HLS interface ap_hs port = pixel_out
		#pragma HLS interface ap_hs port = pixel_in
}
```

## 9.2 實現

​當一個系統實際處理視頻數據時，一般從視頻處理算法實現角度來進行系統分解集成。在本章的剩餘部分，我們將假設輸入數據是按照流的方式送入系統，並且按照掃描線的順序進行處理的。採用HLS開發，我們不關注代碼RTL級具體實現方式，只關注HLS設計能滿足所需要的性能指標要求即可。

### 9.2.1 行緩衝和幀緩衝

​在視頻處理算法中，通常計算一個輸出像素的值時需要輸入像素及其周圍的像素的值做參考，我們把儲存計算所需要的輸入像素值的區域叫**窗口**。從概念上講，按照窗口大小去Z型掃描圖片，然後根據掃描到的像素值計算出輸出結果就可以。如圖9.9所示示例代碼中，示例代碼是針對視頻幀進行二維濾波。示例代碼中在計算每一個輸出像素結果前需要從輸入視頻幀中讀取相應的窗口像素數據（存儲在數組中）。

{% hint style='info' %} 在圖9.9中，代碼中包含的`int wi =row + i – 1;int wj = col +j -1;`, 解釋這些表達式為什麼包含”-1”這一項。提示：如果濾波器核換成7×7，而不是3×3，”-1”這個數字項會改變麼？ {% endhint %}

​注意，在此代碼中，每計算一個輸出像素值，必須多次讀入像素並填充到窗口緩衝區中。如果每個時鐘週期只能執行一次讀操作，示例代碼的執行性能會受限於讀入像素速率大小。因此示例代碼基本上是圖1.8中所示一維濾波器的二維版本。另外，因為輸入像素不是以正常掃描線順序讀取的，所以接口約束也只有有限的幾個可選。（本主題將在9.1.3小節中更詳細討論）。

​仔細觀察相鄰的窗口緩衝區中緩衝的數據，你會發現緩衝的數據高度重疊，這意味着相鄰窗口緩衝區之間數據更高的依存性。這也意味來自輸入圖像的像素可以被本地緩存或者高速緩存存儲，以備數據被多次訪問使用。通過重構代碼來每次只讀取輸入像素一次並存儲在本地緩衝區中，這樣可以使系統性能得到更好的表現。在視頻系統中，由於本地緩衝區存儲窗口周圍多行視頻像素，所以本地緩衝區也稱為**線性緩衝區**。線性緩衝區通常使用Block RAM(BRAM)資源實現，而窗口緩衝區則使用觸發器（FF）資源實現。圖9.10所示是使用線性緩衝區重構的代碼。注意，對於N×N圖像濾波器，只需要N-1行存儲在線性緩衝區中即可。

```c
rgb_pixel filter(rgb_pixel window[3][3]) {
  const char h[3][3] = {{1, 2, 1}, {2, 4, 2}, {1, 2, 1}};
	int r = 0, b = 0, g = 0;
i_loop: for (int i = 0; i < 3; i++) {
 j_loop: for (int j = 0; j < 3; j++) {
      r += window[i][j].R ∗ h[i][j];
      g += window[i][j].G ∗ h[i][j];
      b += window[i][j].B ∗ h[i][j];
	}
}
  rgb_pixel output;
  output.R = r / 16;
  output.G = g / 16;
  output.B = b / 16;
  return output;
}
void video_2dfilter(rgb_pixel pixel_in[MAX_HEIGHT][MAX_WIDTH],
		rgb_pixel pixel_out[MAX_HEIGHT][MAX_WIDTH]) {
    rgb_pixel window[3][3];
row_loop: for (int row = 0; row < MAX_HEIGHT; row++) {
 col_loop: for (int col = 0; col < MAX_WIDTH; col++) {
    #pragma HLS pipeline
    for (int i = 0; i < 3; i++) {
      for (int j = 0; j < 3; j++) {
          int wi = row + i − 1;
          int wj = col + j − 1;
          if (wi < 0 || wi >= MAX_HEIGHT || wj < 0 || wj >= MAX_WIDTH) {
          window[i][j].R = 0;
          window[i][j].G = 0;
          window[i][j].B = 0;
      } else
      		window[i][j] = pixel_in[wi][wj];
      }
    }
    if (row == 0 || col == 0 || row == (MAX_HEIGHT − 1) || col == (MAX_WIDTH − 1)) {
          pixel_out[row][col].R = 0;
          pixel_out[row][col].G = 0;
          pixel_out[row][col].B = 0;
    } else
		pixel_out[row][col] = filter(window);
		}
	}
}
```
![圖9.9: 沒有使用線性緩衝區的2D濾波代碼](images/placeholder.png)


​如圖9.10所示代碼，是使用線性緩衝區和窗口緩衝區方式實現的，其中代碼實現過程如圖9.11所示。代碼每次執行一次循環時，窗口會移動一次，使用來自於1個輸入像素和兩個來自於行緩衝區緩存的像素來填充窗口緩衝區。另外，輸入的新像素被移入線性緩衝區，準備被下一行的窗口運算過程所使用。請注意，由於為了每個時鐘週期處理一個像素並輸出結果，所以系統必須在每個時鐘週期內完成對窗口緩衝區數據的讀取和寫入。另外，當展開”i”循環後，對於窗口緩衝區數組而言，每個數組索引都是一個常量。在這種情況下，Vivado@HLS將轉換數組中的每個元素成一個標量變量（一個成為**標量化**的過程）。窗口數組中的元素隨後使用觸發器(FF)編譯實現。同樣，線性緩衝區的每一行數據都會被訪問兩次（被讀取一次和寫入一次）。示例代碼中明確約束行緩衝區中每一行數組元素被分割成到一塊單獨存儲區中。根據MAX WIDTH的可能取值情況，最後決定使用一個或者多個Block RAM實現。注意，每個Block RAM可在每個時鐘週期支持兩次獨立訪問。

{% hint style='info' %}線性緩衝區是應用於模塊式計算的**重用緩衝區**的一種特殊例子。如圖9.9中所示，重用緩衝區和線性緩衝區的高級綜合是目前一個熱點研究領域。參考[[8](./BIBLIOGRAPHY.md#8), [31](./BIBLIOGRAPHY.md#31)]。{% endhint %}

{% hint style='info' %}Vivado@HLS包含`hls::linebuffer<>`和`hls::window buffer<>`類，它們可以簡化窗口緩衝區和行緩衝區的管理。{% endhint %}

{% hint style='info' %}對於3×3的圖像濾波器核，存儲每個像素佔用4字節的1920×1080圖像的一行數據需要使用多少個FPGA片上Block RAM?{% endhint %}

### 9.2.2 因果濾波器

​圖9.10中實現的濾波器是每個時鐘週期讀取一個輸入像素，計算出一個輸出像素。該濾波器實現原理與圖9.9中所示不完全相同。輸出結果是從先前讀取像素的窗口緩衝區中數據計算出來的。窗口緩衝區中取數順序是“向上和向左”。因此，輸出圖像相對於輸入圖像是“向下和向右”移動的。這種情況類似於信號處理中的因果濾波器和非因果濾波器的原理。大多數信號處理理論的重點是因果濾波器。因為只有因果濾波器對於時間採樣信號（例如，其中$$x[n] = x(n*T)$$和​$$y[n] = y(n*T)$$）才有實用價值。

{% hint style='info' %}在**因果**濾波器$$h[n]$$中，$$\forall k < 0,h[k] = 0$$。雖然，有限濾波器$$h[n]$$不是因果濾波器，但是可以通過延時方法使得而被$$\wedge h[n] = h[n - D]$$轉換為因果濾波器$$h[n]$$。新濾波器$$\wedge y = x \otimes  \wedge h$$的輸出與舊濾波器$$y = x \otimes h$$的延時輸出相同。具體而言，$$ \wedge y = y[n - D]$$。{% endhint %}

```c
void video_2dfilter_linebuffer(rgb_pixel pixel_in[MAX_HEIGHT][MAX_WIDTH],
                               rgb_pixel pixel_out[MAX_HEIGHT][MAX_WIDTH]) {
#pragma HLS interface ap_hs port=pixel_out
#pragma HLS interface ap_hs port=pixel_in
rgb_pixel window[3][3];
rgb_pixel line_buffer[2][MAX_WIDTH];
#pragma HLS array_partition variable=line_buffer complete dim=1
row_loop: for (int row = 0; row < MAX_HEIGHT; row++) {
	col_loop: for (int col = 0; col < MAX_WIDTH; col++) {
	#pragma HLS pipeline
    for(int i = 0; i < 3; i++) {
        window[i][0] = window[i][1];
        window[i][1] = window[i][2];
}
      window[0][2] = (line_buffer[0][col]);
      window[1][2] = (line_buffer[0][col] = line_buffer[1][col]);
      window[2][2] = (line_buffer[1][col] = pixel_in[row][col]);
      if (row == 0 || col == 0 ||
      row == (MAX_HEIGHT − 1) ||
      col == (MAX_WIDTH − 1)) {
      pixel_out[row][col].R = 0;
      pixel_out[row][col].G = 0;
      pixel_out[row][col].B = 0;
} else {
	  pixel_out[row][col] = filter(window);
			}
		}
	}
}
```

![圖 9.10: 使用線性緩衝區實現2D濾波器的代碼](images/placeholder.png)

![圖9.11:圖中所示為圖9.10中的代碼實現的存儲器。 存儲器在特定的循環週期時將從線性緩衝區中讀取出數據存儲在窗口緩衝區的最右端區域。黑色的像素存儲在行緩衝區中，紅色的像素存儲在窗口緩衝區中。](images/video_buffers.jpg)


{% hint style='info' %}通過卷積的定義證明前面原理：
$$
y = x \otimes h:y[n] = \sum\limits_{k =  - \infty }^\infty  {x[k] * h[n - k]}
$$
{% endhint %}

​就本書而言，大多數變量不是時間採樣信號，且單個輸入單個輸出的產生時間可能是在綜合過程中決定的。對於設計時間採樣信號的系統，在使用HLS開發過程中可以進行時序約束處理。只要系統能達到所需的任務延遲，那麼就認為系統設計是正確的。

​在大多數視頻處理算法中，在上文代碼中所引入的空間位移不是預期的，是需要被消除的。雖然有很多種修改代碼的方法來解決這個問題，但是一種常見的方式是**擴展迭代域**。這種技術是通過增加少量的循環邊界，以便第一次循環迭代就讀取第一個輸入像素，但是第一個輸出像素直到等到後面的迭代空間才寫出。修改後的版本濾波器代碼如圖9.12所示。代碼原理如圖9.14所示，和圖9.10中原始緩衝區中代碼相同。使用HLS編譯後，可以看出數據依賴性以完全相同的方式得到滿足，並且待綜合的代碼是硬件可實現的。

### 9.2.3 邊界條件

​在大多數情況下，計算緩衝區窗口只是輸入圖像的一個區域。然而，在輸入圖像的邊界區域，濾波器進行計算的範圍將超過輸入圖像的邊界。根據不同應用的要求，會有很多不同的方法來解決圖像邊界處的計算問題。也許最簡便的方法來解決邊界問題就是忽略邊界，這樣輸出的圖像就會比輸入的圖像長寬各少濾波器核大小。但是在輸出圖像大小固定的應用中，如數字電視，這種方法就行不通了。

```c
void video_2dfilter_linebuffer_extended(
  	rgb_pixel pixel_in[MAX_HEIGHT][MAX_WIDTH],
	rgb_pixel pixel_out[MAX_HEIGHT][MAX_WIDTH]) {
    #pragma HLS interface ap_hs port=pixel_out
    #pragma HLS interface ap_hs port=pixel_in
    rgb_pixel window[3][3];
    rgb_pixel line_buffer[2][MAX_WIDTH];
    #pragma HLS array_partition variable=line_buffer complete dim=1
    row_loop: for(int row = 0; row < MAX_HEIGHT+1; row++) {
     col_loop: for(int col = 0; col < MAX_WIDTH+1; col++) {
      #pragma HLS pipeline II=1
      rgb_pixel pixel;
      if(row < MAX_HEIGHT && col < MAX_WIDTH) {
      	pixel = pixel_in[row][col];
        }
      for(int i = 0; i < 3; i++) {
        window[i][0] = window[i][1];
        window[i][1] = window[i][2];
      }
      if(col < MAX_WIDTH) {
        window[0][2] = (line_buffer[0][col]);
        window[1][2] = (line_buffer[0][col] = line_buffer[1][col]);
        window[2][2] = (line_buffer[1][col] = pixel);
      }
      if(row >= 1 && col >= 1) {
          int outrow = row−1;
          int outcol = col−1;
          if(outrow == 0 || outcol == 0 ||
              outrow == (MAX_HEIGHT−1) || outcol == (MAX_WIDTH−1)) {
                  pixel_out[outrow][outcol].R = 0;
                  pixel_out[outrow][outcol].G = 0;
                  pixel_out[outrow][outcol].B = 0;
              } else {
             	 pixel_out[outrow][outcol] = filter(window);
              }
          }
      	}
      }
}
```

![圖 9.12: 使用線性緩衝區並且循環迭代範圍擴展為1時實現2D的濾波器代碼](images/placeholder.png)

![圖9.13:不同濾波器濾波後的效果圖。圖b是圖9.9中代碼生成效果圖，圖c是由圖9.10中代碼生成效果圖，圖d是由圖9.12中代碼生成效果圖與圖像b相同](images/filter2d_results_withshifting.jpg)

![圖9.14：使用線性緩衝區方法代碼執行時序。上方執行順序為圖9.10中代碼執行時序。下方執行時序為圖9.12中代碼執行時序。紅色框標記的像素在兩種實現中以相同的週期輸出，在上方執行時序中表示第二行第二個像素，在下方時序中表示第一行第一個像素](images/video_timelines.jpg)

![圖9.15：不同邊界條件處理方法效果圖](images/filter2d_results_boundary_conditions.jpg)

另外，當處理大量大小略微不同的圖片時，需要一系列大小的濾波器核來進行處理，這種情況就複雜了。圖9.10中的代碼展示，輸出圖像通過在邊界處填充已知值的像素（一般填充黑色）來達到與輸入圖像大小相同。或者通過其他方式來合成缺失的值。

* 缺少的輸入值可以用常量來填充；
* 可以從輸入圖像的邊界像素處填充缺少的輸入值；
* 缺失的輸入值可以在輸入圖片內部像素中進行重構。

當然，也存在更復雜和更計算密集的方案。圖9.16中展示了一種處理邊界條件的方案。此方案是通過計算每個像素在窗口緩衝區中的偏移地址方法。這種方案的一個明顯的缺點就是每次從窗口緩衝區中讀取的地址都是變化的地址。因此，在計算濾波之前，這個變量會導致多路複用。對於N×N的抽頭濾波器，針對N個輸入將會消耗大約N×N個多路複用器。對於濾波器而言，多路複用器佔濾波器資源消耗的大部分。另一種處理數據寫入窗口緩衝區時的邊界條件是以規則模式移動窗口緩衝區。在這種情況下，只消耗N個多路複用器，而不是N*N個，從而是資源消耗量降低很多。

{% hint style='info' %}修改圖9.16中的代碼，從窗口緩衝區中讀取數據使用常量地址。你節省了多少硬件資源？ {% endhint %}

## 9.3 結論

​視頻處理是一種常見的非常適合使用HLS實現的FPGA應用。大多數視頻處理算法有一個共同特點：數據局部性，即可以在使用少量的外部存儲器訪問的情況下，能夠實現流式傳輸和本地緩存的應用。

```c
void video_2dfilter_linebuffer_extended_constant(
  rgb_pixel pixel_in[MAX_HEIGHT][MAX_WIDTH], rgb_pixel pixel_out[MAX_HEIGHT][MAX_WIDTH]) {
  #pragma HLS interface ap_hs port=pixel_out
  #pragma HLS interface ap_hs port=pixel_in
  rgb_pixel window[3][3];
  rgb_pixel line_buffer[2][MAX_WIDTH];
  #pragma HLS array_partition variable=line_buffer complete dim=1
  row_loop: for(int row = 0; row < MAX_HEIGHT+1; row++) {
	col_loop: for(int col = 0; col < MAX_WIDTH+1; col++) {
	#pragma HLS pipeline II=1
	rgb_pixel pixel;
	if(row < MAX_HEIGHT && col < MAX_WIDTH) {
		pixel = pixel_in[row][col];
	}
	for(int i = 0; i < 3; i++) {
		window[i][0] = window[i][1];
		window[i][1] = window[i][2];
	}
	if(col < MAX_WIDTH) {
		window[0][2] = (line_buffer[0][col]);
		window[1][2] = (line_buffer[0][col] = line_buffer[1][col]);
		window[2][2] = (line_buffer[1][col] = pixel);
	}
	if(row >= 1 && col >= 1) {
		int outrow = row−1;
		int outcol = col−1;
		rgb_pixel window2[3][3];
		for (int i = 0; i < 3; i++) {
			for (int j = 0; j < 3; j++) {
				int wi, wj;
				if (i < 1 − outrow) wi = 1 − outrow;
				else if (i >= MAX_HEIGHT − outrow + 1) wi = MAX_HEIGHT − outrow;
				else wi = i;
				if (j < 1 − outcol) wj = 1 − outcol;
				else if (j >= MAX_WIDTH − outcol + 1) wj = MAX_WIDTH − outcol;
				else wj = j;
				window2[i][j] = window[wi][wj];
	}
}
		pixel_out[outrow][outcol] = filter(window2);
			}
		}
  	}
}
```

![圖9.16: 使用線性緩衝區核常量邊界來實現處理邊界條件的代碼，若用這種方法處理消耗硬件資源成本很大](images/placeholder.png)
