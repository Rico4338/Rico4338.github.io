---
title: "關於C# List Add效能的那點事之一"
date: 2023-02-20T22:57:13+08:00
draft: false
---

# 陣列長度是固定的，但是List\<T\>好像沒有特別的限制耶?

> 以前在學校學到，當今天宣告一個array的時候是需要給他`長度`的。  
> 所以當arry的長度不夠用的時候，需要宣告一個新的array來把東西搬過來放。

但是我們用C#的時候很多時候都直接用`List<T>`做處理，像是下面
```csharp
var userNo = new List<int>();
for(var i = 0 ; i < 5 ; i++)
{
    userNo.Add(i);
}
```
  
我們就是單純的把數字填入`List<T>`中，那...在這段程式怎麼在記憶體中分配呢?分配的容量又是多少呢??  
  
首先，我們先來看一下，`List<T>`中有這個屬性  

{{< image src="../img/list-capacity.png" caption="範例1 (`list capacity`)" src_l="../img/list-capacity.png" >}}
以及
{{< image src="../img/new-list-capacity.png" caption="範例2 (`list capacity`)" src_l="../img/new-list-capacity.png" >}}  

沒錯，就是`Capacity`，我們來看看[MSDN上面的說明](https://learn.microsoft.com/zh-tw/dotnet/api/system.collections.generic.list-1.capacity?view=net-7.0)  
也寫點程式來驗證一下..
```c#
var a = new List<int>();
Console.WriteLine(a.Capacity);
for (var i = 0; i < 3; i++)
{
    a.Add(i);
    Console.WriteLine(a.Capacity);
}
//output
// 0 --最一開始是零沒問題
// 4 --疑?我加一個東西就變成4了....
// 4
// 4
// 4
// 8 --為啥我加到第五個就變成8了呢?
```
這個時候最快的方式就是進去Add看一下`List<T>.Add()`做了啥事

{{< image src="../img/list-add.png" caption="反編譯後的Add method" src_l="../img/list-add.png" >}}  
然後我們來看看裡面的`AddWithResize(item)`

{{< image src="../img/add-with-resize.png" caption="反編譯後的AddWithResize" src_l="../img/add-with-resize.png" >}}  

再追深一點，`Grow`
{{< image src="../img/grow.png" caption="反編譯後的Grow" src_l="../img/grow.png" >}}  

從這邊可以看到，當我們今天容量不夠的時候，我們的容量是直接`乘二`(DefaultCapacity是4)。  

從這邊我們看到一個很重要的資訊，也就是當我今天沒有給`Capacity`的時候，我們的空間會是每滿一次就擴容乘二一次，所以當我們今天有個List\<T\>，然後有129個變數要放進去，他就會從0->4->8->16->32->64->128->256，改變了七次的大小。  
如果今天只是單純改變大小可能還沒事，因為List在記憶體中是不連續的位置，是靠指標來指向下一個記憶體位置，如下圖  

{{< image src="../img/array-ram-address.png" caption="很醜的記憶體不連續示範" src_l="../img/array-ram-address.png" >}}  

但是，就是這個討厭的但是，我們來看產生List\<T\>的地方，觀察一下`new List<T>`裡面的行為
{{< image src="../img/list-constructor.png" caption="list的constructor" src_l="../img/list-constructor.png" >}}  

咦? `T[0]`? 塞了一個空的array耶? 那剛剛看了Add method那段裡面用來放入資料的不就是單純的Array嗎? 只是他會幫我自動變長(伸縮自...不對他不會縮)  
那我們再來看一下`Capacity`，我們上面有提到容量不夠會自動乘二，`Capacity`的數值改變的時候會發生啥事情

{{< image src="../img/capacity-get-set.png" caption="反編譯後的Add method" src_l="../img/capacity-get-set.png" >}}  

原來這邊他做的是`Array.Copy`....只是一個有自動擴大的Array而已。  
看到這邊有經驗的大大應該就會知道後面又會是一個`深淺複製`與`reference type`、`value type`的問題了。  
所以我的結論是要特別小心`value type`類型的`List`的Add次數問題。  
或是直接給他一個預設大小。  
{{< image src="../img/new-list-with-capacity.png" caption="反編譯後的Add method" src_l="../img/new-list-with-capacity.png" >}}  

{{< typeit >}}
不然 [GC](https://learn.microsoft.com/zh-tw/dotnet/api/system.gc?view=net-7.0) 的 **壓力** 會影響到 *整體效能*的...
{{< /typeit >}}




{{< admonition tip "外部參考" >}}
* https://stackoverflow.com/questions/2760931/initial-capacity-of-collection-types-e-g-dictionary-list
* https://blog.darkthread.net/blog/oom-with-adequate-memory/
{{< /admonition >}}




