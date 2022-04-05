---
title: "JavaScript中常用來操作Array的三個方法－map、filter、reduce"
excerpt: "一定要徹底了解JavaScript中，常用來操作Array的三個方法，map()"
tags:
  - "JavaScript"
  - "Programming"
---

![array is similiar with a pill box.](/assets/images/article_about/JS%20array-map_filter_reduce.jpg){: .width-three .align-center}

<p style="text-align: center; color: #777">Photo by Laurynas Mereckas on Unsplash</p>

清明四天的連假不知不覺又要最後一天了！趁著連假的最後一天趕快把這週應該要完成的文章寫完，預計從這週開始在課堂上教學 Ruby 後要花許多時間在 Ruby 以及 Rails 上！儘管如此還是要不斷的提醒自己要努力完成每週至少一篇文章的目標。這次決定來講一下在 JavaScript 中常常用來操作 array 的三個方法－`map`、`filter`、`reduce`，並且額外記錄一下在剛接觸時常常會用錯的`forEach`。

## forEach：一個一個看過 array 內的每個元素

剛學習 JavaScript 時想要取出 array 中每個元素時大都使用`for`迴圈來完成，例如：

{% highlight javascript linenos %}
const fruitBasket = ["banana", "guava", "apple"];

for (let i = 0; i < fruitBasket.length; i++) {
  console.log(fruitBasket[i]);
}
{% endhighlight %}

但是在 ES6 之後提供了`forEach`方法來讓我們完成一樣的事情，`forEach`的第一個參數內要放入一個 callback function，callback 可以放入三個參數：

1. 必填的參數，代表逐次從 array 中取出來的元素
2. 選用的參數，代表逐次取出來元素的 index 值
3. 選用的參數，代表呼叫`forEach`的 array（幾乎沒用到）

我個人認為這種寫法除了可以撰寫的比較快，閱讀上也比較快一點，來看看`forEach`的例子：

{% highlight javascript linenos %}
const fruitBasket = ["banana", "guava", "apple"];

fruitBasket.forEach((fruit, index) => {
  console.log(`從籃子內取出的第${index + 1}個水果是${fruit}`);
});
{% endhighlight %}

既然`forEach`這麼好用，那幹嘛還要學使用`for`迴圈來取得 array 的元素呢？想當然爾一定是有一些情況會碰到不能夠使用`forEach`，常碰到的應該是以下幾種情況：

- 想要在碰到某個條件後停止繼續取出 array 的元素

{% highlight javascript linenos %}
const order = ["Ben", "Tgo", "Sarah", "Jason", "Molly", "Chirs", "Kendi"];

for (let i = 0; i < order.length; i++) {
  if (i === 3) {
    console.log("名額已滿");
    break;
  }

  console.log(`${order[i]}報名成功`);
}
{% endhighlight %}

- 碰到某些 array-like 的物件不包含`forEach`方法時，例如：HTMLCollection

{% highlight javascript linenos %}
const divs = document.getElementsByTagName("div");

for (let i = 0; i < divs.length; i++) {
  console.log(divs[i]);
}
{% endhighlight %}

當然不是所有的 array-like 都這樣，例如 NodeList 就有包含`forEach`可以使用

## map：對 array 的元素改造後並且組成新的 array

當碰到需要將 array 的每個元素的出來並且進行計算時，就會需要用到`map`方法，`map`的第一個參數要放入 callback function，callback 的程式碼就是用來改造元素的部分，而 callback 可以放入三個的參數：

1. 必填的參數，代表逐次從 array 中取出來的元素
2. 選用的參數，代表逐次取出來元素的 index 值
3. 選用的參數，代表呼叫`map`的 array（幾乎沒用到）

來看看使用`map`來操作 array 的例子：

{% highlight javascript linenos %}
const scores = [{ name: "Tgo", score: 24 }, { name: "Ben", score: 40 }, { name: "Sarah", score: 58 }];

const newScores = scores.map((student) => {
  return {
    name: student.name,
    score: Math.round(Math.sqrt(student.score) * 10),
  };
});

console.log(newScores);
// [{ name: 'Tgo', score: 49 },{ name: 'Ben', score: 63 },{ name: 'Sarah', score: 76 }]
{% endhighlight %}

為了要將這次的分數好看一點，但又想要保留原始分數的紀錄。可以使用`map`方法來達成我們的目的，在 callback 的第一個參數是每次取出來包含學生姓名和分數的物件，利用`Math`物件提供的`sqrt`開平方和`round`四捨五入的方法幫學生的成績加分，並且在最後要注意的是要<span style="color: #f33;">將運算後的結果`return`出來，不然`map`會接收不到運算後的結果</span>

## filter：從 array 中篩選出符合條件的元素

`filter`的第一個參數要放入 callback function，callback 的程式碼就是篩選的條件，而 callback 可以放入三個的參數：

1. 必填的參數，代表逐次從 array 中取出來的元素
2. 選用的參數，代表逐次取出來元素的 index 值
3. 選用的參數，代表呼叫`filter`的 array（幾乎沒用到）

這次接續著上個例子來看看`filter`的使用方法：
{% highlight javascript linenos %}
const failedStudents = newScores.filter(({ score }) => {
  return score < 60;
});

console.log(failedStudents); // [ { name: 'Tgo', score: 49 } ]
{% endhighlight %}

這次想要找出加分後成績不及格的學生，`filter`可以幫我們找到所有符合條件的學生，不過這次的例子裡剛好不及格的就只有一個，得到的 array 只有一個學生的分數。這裡可以發現 return 的地方是一個布林值，因為`filter`是<span style="color: #f33">依照布林值來判斷要不要將取得的元素放到新的 array 當中</span>，如果是`return true`則會將該次取出來的元素放進新的 array 當中。

## reduce：將 array 的元素進行改造後產生一個值

看完敘述不要以爲這個與`map`的用法一樣，先來看看`reduce`的語法。`reduce`的第一個參數放入一個 callback function，callback 的程式碼是用來改造元素的部分，callback 可以放入以下參數：

1. 代表前一次取出元素並且進行運算後的結果，如果是取 array 元素的第一圈時，則看有沒有設定初始值來決定是誰
2. 代表當前取出 array 的元素，如果是取 array 元素的第一圈時，則看有沒有設定初始值來決定是誰
3. 代表當然取出 array 元素的 index，如果是取 array 元素的第一圈時，有設定初始值的話從`0`開始，沒有設定則從`1`開始
4. 選用的參數，代表呼叫`reduce`的 array(應該不常用到)

`reduce`的第二個參數則是設定初始值，有沒有設定初始值會影響第一個參數 callback 從 array 取出元素的順序，來看個簡單的例子觀察整個`reduce`從 array 當中取出元素的順序：
{% highlight javascript linenos %}
const arr = [0, 1, 2, 3, 4];
let counter = 1;

function reduceWithoutInitial(arr, counter) {
  arr.reduce((prev, current, index) => {
    console.log(
      `第${counter}圈：prev值是${prev}, current值是${current}, 當前的index值是${index}`
    );
    counter++;
    return current;
  });
}

reduceWithoutInitial(arr, counter);

function reduceWithInitial(arr, counter) {
  arr.reduce((prev, current, index) => {
    console.log(
      `第${counter}圈：prev值是${prev}, current值是${current}, 當前的index值是${index}`
    );
    counter++;
    return current;
  }, 0);
}

reduceWithInitial(arr, counter);
{% endhighlight %}

因為`console.log`印出的結果比較長建議可以複製到編輯器內去執行看看結果。當我們沒有設定`reduce`的第二個參數（沒有初始值）的情況下跑第一圈時，callback 的第一個參數的值就會是 array 的第一個元素，第二個參數值會是第二個元素，第三個參數值自然指的是第二個參數的 index 也就是 1。因為 array 的第一個元素直接在第一圈就被取出來了，所以整個 array 就少跑了一圈。

當我們設定了`reduce`的第二個參數，也就是有初始值的情況下，跑第一圈時 callback 的第一個參數的值就會是初始值，第二個參數值會是第一個元素，第三個參數值自然指的是第一個參數的 index 也就是 0。這時候的整個 array 跑的圈數跟 array 的長度是一樣的。

看完了漏漏長的語法，來看看有什麼地方會用到`reduce`，根據 google 的結果`reduce`的用途非常多，但這篇文章先介紹兩種用途：

- 將每個元素進行數學計算法後得到計算後的結果，這邊一樣可以延續學生的例子來討論
{% highlight javascript linenos %}
const totalScore = newScores.reduce((prev, current) => {
  return prev + current.score;
}, 0);

console.log(totalScore); // 188
{% endhighlight %}

`reduce`可以幫我們達成取出學生的分數後使用加法取得所有學生分數的總和。比起使用其他的方法，使用`reduce`可以很快的達成我們將的目的。

- 將二維 array 變成一維 array
  {% highlight javascript linenos %}
const arr = [
  [1, 2, 3],
  [4, 5, 6],
];

const newArr = arr.reduce((prev, current) => {
  return prev.concat(current);
}, []);

console.log(newArr); // [ 1, 2, 3, 4, 5, 6 ]
{% endhighlight %}

有時候碰到要將二維的陣列降維之後再操作裡面的元素，用`reduce`的寫法也相當的簡短閱讀上也不難懂。

總結一下 reduce 使用時要注意的地方：

1. 初始值會影響迴圈跑的次數以及 callback 在第一圈時取的值
2. 每跑一次迴圈 callback 第一個參數的值，會是前一次迴圈 callback 中 return 的值，因此使用時要注意不然容易發生錯誤
3. `reduce`是操作 array 後回傳一個值，那麼這個值可以是不同的東西，例如：string、number、array、object、map、set...等，所以可以利用`reduce`衍生出許多種操作資料用法


## 總結

這次介紹的`map`、`filter`、`reduce`算是常常會用到的方法，使用時都要特別注意在 callback function 內的`return`值是什麼，不然容易產生預期以外的結果。此外這三種方法在介紹 Higher Order Function 時常被拿來當作範例，至於 Higher Order Function 是什麼就再挖個坑給之後的自己去填起來吧！文章寫到這也實在是夠長的了...

<br>

此篇文主要目的在記錄學習程式語言時的筆記，如果有錯誤之處請不吝於私訊或來信指教，我會很感謝您給的回饋！
以上內容參考自下列網站/文章：
1. [Array - JavaScript - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array)
2. [卡斯伯的 Blog \| JavaScript 陣列處理方法](https://www.casper.tw/javascript/2017/06/29/es6-native-array/#Array-prototype-reduce)
3. [Medium 學習Blog \| 【JavaScript】Array Reduce的用法](https://tzulinchang.medium.com/javascript-array-reduce%E7%9A%84%E7%94%A8%E6%B3%95-c435611a2935)