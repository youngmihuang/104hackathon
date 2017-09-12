# 104hackathon

- [參賽心得](https://medium.com/@cyeninesky3/104-hackathon-%E5%88%9D%E6%88%B0%E5%BF%83%E5%BE%97-a810cfe09192)
![image alt](https:// "title")

---

# 104 Hackathon 初戰心得
104 Hackathon 0729一整天比賽，實際前置作業大約1週。

## 心態：第一次參加黑客松
有踏出比賽這一步比什麼都不做來得好，比賽過程著重實戰hands-on，真的有學到不少東西，不管結果如何，都要保有與自己比的健康心態。
Gain More than nothing to do.
## 這次主要做的事情
104 Hackathon這次主要分成三個主題，我們參加的是Match Recommendation，104給予一段特定區間內去識別化的行為記錄（user Log），以及千大企業刊登的工作內容做為 Training Data，從中找尋出使用者瀏覽應徵模式，以預測求職者是否有應徵該工作。計分標準為推算結果之準確率(Precision, Recall, F-Measure)。

1. 整理資料：這次104給的Training資料最主要的 “user_log.csv”就高達7.97GB，內含2016一整年顧客的瀏覽工作、瀏覽公司、儲存工作、應徵工作的行為資料檔（104萬不重複uid、約6萬不重複jobNo）

其餘的資料為該Job更詳細的資料面向（e.g.職務名稱、學歷要求、工作型態(正職/兼職)、薪資希望待遇、相關經驗、相關科系…etc)；以及該公司的型態：是否為上市櫃公司、是否為資本額500大公司
詳細的data schema：https://github.com/youngmihuang/2017-104Hackathon-Recommendation/tree/master/data-schema


---

2. 因子創建：
和組員討論因子與建模方向，之前看過許多文章都說因子的創建往往決定模型的好壞，我自己心中也是傾向要創出有用的因子 >> 一樣的因子試不同的模型；但就如同每個模型有其優劣點，像是NN這種需要多層的就適合因子很多的時候使用，當因子選的好，試不同的模型才能發揮它的功效。
(1)時間對應徵者應徵工作的影響：在未分產業前，整體工作求職者點選apply的時間分佈如下，週間以日～四投遞為主，週五六偏少；應徵者apply的熱門高峰在過年後2月、4月、7–9月。
應徵人數與時間的分佈(2) 整體user：每小時瀏覽量
每小時瀏覽量(3) 整體user：每個月不重複的uid分佈：2–3月年後轉職潮、7–8月暑期工作(maybe短期工讀大量出現 ->這部分尚未細究)
每月不重複uid分佈(4) 每月應徵機率分佈：應徵人數/(應徵人數+沒有應徵人數)
每月應徵機率若把維度切成只有Apply的分佈長的類似，因此時間是個大環境因子，並非決定應徵者是否Apply的顯著因子。
(4) job資訊(自然語言處理)：隊友有把job_stuctured_info所有描述轉成word2vec的形式，從一開始拆成300欄-> 100欄 ->50欄，切成幾欄位效果最好其實不一定，比賽當天因為我們底層AWS運算環境server效能，所以我們一路從300欄一路縮減成50欄丟進去當X因子，以避開Memory Error的問題。
(5) viewJob、saveJob、viewCust(看公司的紀錄、jobNo為空值)：當action是viewCust的時候需要特別處理，一開始的想法是先將行為資料依照uid序時排序後，可以找出viewCust的前一筆看的工作帶入該工作的jobNo (代表他看了該工作之後對該公司有興趣)，但卻也發現該user第一筆就為viewCust的情形，於是果斷先去除viewCust的資料。（因最後predict是以一筆uid jobNo 為input）
(6) 這次來不及創建出的因子：job-prefered or company-prefered
當一個user進入求職網站時，有分兩種型態：
一種是以job種類為主的user，舉例來說：過去都在PM領域的，就會想找專案經理的職位、那這樣行為的user一進來的時候我就可以分流它使用job-based的模型；若是另一種就使用company-prefered的模型。
另一種是以產業為主的user，舉例來說：一個只看產業&特定公司投履歷的user，他在意的是進到半導體業就OK，他不會以job title為優先 &應徵非半導體業的job（比方說他就不會去應徵彩妝業的工程師)；再舉一個company-prefered的例子，一個想應徵無印良品、MUJI的user，她因為想進這兩間公司，於是只要是這間公司的job都符合她apply的需求。
其實有想過很多可能性，還有已經串好但尚未用上的因子：公司的500大註記、是否上市櫃、時間註記、job info的挑選等


---

3. 模型使用與預測能力
(1) 在訓練模型的時候是以整理完成的Datamart（歸至uid、jobNo），以7:3的比例去train模型，整體的資料約有3000多萬筆。
對於預測是否apply的前兩名FeatureviewJob次數、saveJob次數、ratio_viewJob、ratio_saveJob
第一次上傳的版本是僅使用4個因子、使用sklearn的LogisticRegression，因資料會有imbalanced的問題，故設定LogisticRegression參數時需設定class_weight=’balanced’。模型在train出來的時候，在Recall的部分表現較差，等於在召回有apply的能力較弱。
F-measure: 0.4682 (#1)
總共只能上傳10次，分數為F-measure(2) 後續有將上述4個因子使用NN上傳了一版(#2)，表現不是很好哈哈，因子數過少不適合用NN；接下來用job info那50個欄位黏起來再丟進去用Logistic(#4)、使用NN(#5)，兩個的F-measure仍差一些。
(3) 儘管中間做了那麼多事情，可以發現我們分數最高的一次(#7)，也不是做了非常了不起的因子，而是使用了Rule-based: viewJob次數>2次，前面這樣做下來viewJob這個因子仍是最強的。


---

4. Python能力的運用
整理資料真的是一件超級大大大大工程，主要感謝我的隊友與估狗大神，尤其在描述我想要做到事情(e.g. 用SQL表示我想要做到的事情: proc sql; select uid, jobNo, count(action) …..(省略) group by uid, jobNo)
pandas幫助我了許多大忙，但似乎效能就會不好，有一好必有一不好。pd.head() -> 大概榮登我使用最多次的一個指令哈哈
其他技能像是：filter, merge, drop, duplicated, sort_values, rename, dtypes……….etc


---

可改善的方向&下次要練習的技能
運算效能真的很重要

最痛的點就是不斷遇到 memory error，這次比賽有AWS的贊助，我們組員五個人使用同一台遠端server，根據資料工程隊友說明，我們的底層運算效果應該要選擇memory最優化才不會有不能算的情形；同時這次大賞得AWS獎的是趨勢科技的team，據說他們所設計的運算架構就很優。這次比賽之後，可以來瞭解一下底層架構的東西。
2. 受限於前置時間（有其他事物排擠），無法把整好的因子通通試一遍，這次在比賽當天才把第一版模型做出來，把資料接上，丟出第一版資料。
3. 可改善研究因子對目標output的特性：
(1) 時間(求職旺季)、職稱主管職/非主管職註記、company & job的分類表(類目表整理)
example： company分類表：1001001001 -> [100, 100, 1001] = [‘電子資訊/軟體/半導體相關業’, ’軟體及網路相關業’, ‘電腦系統整合服務業’]
Company分類表(2) job描述特徵萃取的最優化：300個如何挑選出最有效的? 或是50個、30個其實就有效果?
(3) Class分類器(company-preferred or job-preferrd這件事)
4. 視覺化的task下次要多練習（這次被隊友carry了)：e.g. matplotlib、seaborn。
Matplotlib documentation：https://matplotlib.org/examples/index.html
Ploty：https://plot.ly/python/
C3.js：http://c3js.org/examples.html

5. 讀取優化效能的檔案：e.g.暫存成Pickle、no more csv
6. 有其他組的hacker反應此次主辦單位應可多給一些user基本的Profile資料提供建模，這些因子應能幫助提升預測能力。


---

接下來要做的事情
就是把這次練習的過程，整理code到github，這一步其實是相當重要的，累積可用程式碼& Sharing，若後續整理完成後會再補上這裡:)
心得
以上就是這次黑客松的心得，雖然是第一次參加黑客松(還遇到颱風天)，預測結果是Match Recommendation分組第4，實戰真的學得比較快、多；過程中遇到的Python技巧問題一一擊破，再次感謝這次組隊的隊友們和估狗大神，能學到東西才是最重要的。
