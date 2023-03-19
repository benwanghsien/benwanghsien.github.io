---
title: "Rails: Active Support Instrumentation"
excerpt: "研究Rails的黑魔法如何將我們需要的資訊在Console顯示出來(Episode 1 of 2)"
tags:
  - "Ruby"
  - "Ruby on Rails"
  - "Programming"
---

最近在工作上接到一個任務，要將我們替使用者用送來的request產生的log message，加上一些讓我們需要的資訊。如果該使用者的request有向Elasticsearch搜尋資料時，要在log message內紀錄Elasticsearch的執行時間。所以便開始去研究Rails究竟是用了什麼黑魔法取得我們需要資訊並且產生log。

Active Support是Rails的核心功能之一，主要有下列的用途：擴充Ruby、提供utility等等。今天要介紹的是Instrumenation API，主要用於測量在application中某些行為的執行時間、執行過程的資訊。這個API不一定要搭配Rails才能使用，也可以在其他的framework或是單純的Ruby script使用它。Insturmentation API也有另一個用途，在Application中用來實作Observation Pattern，或稱作Publish-Subscribe Pattern。簡單來說就是在A物件執行特定的行為時發送事件，其他有訂閱該事件的物件便可以取得A物件在事件中夾帶的一些資訊。目前只是大概知道這個Pattern在幹嘛，更詳細的Observation Pattern(Publish-Subscribe Pattern)等到之後開始學習Design Pattern在說深入研究！

Instrumentation API使用的方法：
- 建立Instrumenter：首先先準備發布事件的物件，透過`ActiveSupport::Nofitication`的`instrument`方法發布某個事件，第一個參數的type為string，代表的是事件的名稱。在Rails中事件的名稱有個convention－`event.library`，`event`指什麼事件，`libray`指這個事件與物件、函式庫有關。第二個參數type為hash，代表的是payload(要傳送的資訊)，可以隨意的自定義payload的內容。後面接著block將要執行的行為放在block內，當block內的程式碼完成後，事件就會發布通知。`ActiveSupport`會自動將payload加上某些特定資訊：事件**開始時間**、**結束時間**、**Instrumenter unique ID(該事件的ID)**。
{% highlight ruby linenos %}
ActiveSupport.Notification.instrument("event.library", {custom_data: "this is a custom event."}) do
  # Actions
end
{% endhighlight %}


- 建立Subscriber：有了事件的來源後，其他物件則訂閱該事件去取得發布事件的物件要取得的訊息，透過`ActiveSupport::Notification`的`subscribe`方法去註冊某個事件，第一個參數的type可以是string或是regexp，如果是string是指的是特定的事件名稱，如果是regexp則指的是符合該表達式的所有事件。第二個參數為optional, 放入一個可以回應`call`方法的callback物件，這個`call`方法用來執行收到事件後要做的行為，可以取得五個參數，分別是：

| parameter | description |
| :-------: | ----------- |
| `name` | 接收到事件的事件名稱 |
| `started` | 事件的開始時間，從Instrumenter執行block時開始計時 |
| `finished` | 事件的結束時間，當Instrumenter執行完block時結束計時 |
| `id` | 事件的ID |
| `payload` | 事件的payload |

<br>
  除了接收第二個參數以外，`subscribe`也可以放入block來執行收到事件後要做的行為，使用方式與放入第二個參數很像，兩者擇一使用就可以。
  {% highlight ruby linenos %}
  # 加上第二個參數
  class MyObject
    def call(name, started, finished, id, payload)
      # Actions
    end
  end

  ActiveSupport.Notification.subscribe("event.library", MyObject.new)

  # 加上block
  ActiveSupport.Notification.subscribe("event.library") do |name, started, finished, id, payload|
    # The action
  end
  {% endhighlight %}

  不論是放入block或是第二個參數，他們的parameter都落落長，可以將這些參數送到`ActiveSupport::Notification::Event`建立一個event物件。
  {% highlight ruby linenos %}
  # 加上第二個參數
  class MyObject
    def call(*args)
      event = ActiveSupport::Notification::Event.new(*args)
      # Actions
    end
  end

  ActiveSupport.Notification.subscribe("event.library", MyObject.new)

  # 加上block
  ActiveSupport.Notification.subscribe("event.library") do |*args|
    event = ActiveSupport::Notification::Event.new(*args)
    # The action
  end
  {% endhighlight %}

  再懶一點，如果參數只設定一個的話，那個參數會直接是event物件！
  {% highlight ruby linenos %}
  # 加上第二個參數
  class MyObject
    def call(event)
      # event就是ActiveSupport::Notification::Event
      # Actions
    end
  end

  ActiveSupport.Notification.subscribe("event.library", MyObject.new)

  # 加上block
  ActiveSupport.Notification.subscribe("event.library") do |event|
    # event就是ActiveSupport::Notification::Event
    # The action
  end
  {% endhighlight %}

- 取消訂閱某事件Unsbuscribe: 一個訂閱完整的訂閱機制除了可以訂閱，當然也可以取消訂閱！取消訂閱有兩種方式，一種是取消某個subscription，另一種是取消某個事件所有的subscriptions，同樣都使用`ActiveSupport"::Notification`的`unsubscribe`方法。`unsubscribe`的參數可以放入兩種不同type的參數，如果是subscriber物件的話，將那個訂閱取消掉，如果是事件名稱的話，則會取消所有訂閱該事件的subscriptions。
  {% highlight ruby linenos %}
  # 取消某個subscription
  # ActiveSupport.Notification.subscribe會回傳subscriber物件
  subscriber = ActiveSupport.Notification.subscribe("event.library") do |*args|
  event = ActiveSupport::Notification::Event.new(*args)
  # The action
  end

  ActiveSupport.Notification.unsubscribe(subscriber)

  # 取消所有訂閱event.library事件的subscriptions
  ActiveSupport.Notification.unsubscribe("event.library")
  {% endhighlight %}

- 處理Exception：在Publisher的block內會放入我們執行的行為，在block內發生例外時，程式並不會被中斷，Rails會捕捉例外情況，並且會在payload加上兩個key來存放例外的資訊，`payload[:exception]`是個兩個element的array，第一個element是例外的class名稱，第二個element是例外的訊息。`payload[:exception_object]`是該次例外的物件。
  {% highlight ruby linenos %}
  subscriber = ActiveSupport.Notification.subscribe("event.library") do |*args|
  event = ActiveSupport::Notification::Event.new(*args)
  payload = event.payload

  payload[:exception] # ["ArgumentError", "Invalid value"]
  payload[:exception_object] # #<ArgumentError: Invalid value>
  end
  {% endhighlight %}


整個Instrumentation API的使用方法就是這樣子啦！如果是Rails專案，Rails已經提供許多event可以使用，例如：`process_action.action_controller`、`sql.active_record`等等，這些都是Rails許多library裡面建立的事件，在[Rails Guide](https://guides.rubyonrails.org/active_support_instrumentation.html#rails-framework-hooks)中有列出有哪些事件可以使用。


在理解了Instrumentation API怎麼用之後，Rails還有其他的黑魔法將這些物件分配到其他物件去印出log message。剩餘的部分就在下一篇文章再來介紹！

<br>

此篇文主要目的在記錄學習程式語言時的筆記，如果有錯誤之處請不吝於留言給予指教，我會很感謝您給的回饋！
以上內容參考自下列網站/文章：

1. [Rails Guide - Active Support Instrumentation](https://guides.rubyonrails.org/active_support_instrumentation.html)
2. [Rails source code](https://github.com/rails/rails/tree/main/activesupport/lib/active_support)
3. [Akshay's Blog \| Understanding the Instrumentation API in Rails](calendar.google.com/calendar/u/0/r?tab=wc&pli=1)
