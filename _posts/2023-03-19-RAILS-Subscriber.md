---
title: "Rails: Active Support Subscriber"
excerpt: "研究Rails的黑魔法如何將我們需要的資訊在Console顯示出來(Episode 2 of 2)"
tags:
  - "Ruby"
  - "Ruby on Rails" 
  - "Programming"
---

本篇接續上一篇介紹Rails Instrumentation API的文章，如果不清楚或忘記Active Support Instrumentation是做什麼的歡迎回顧[上一篇](https://benwanghsien.github.io/RAILS-Instrumentation/)文章。

上篇文末介紹了Rails有提供了許多的事件可以使用，主要是用來產生log(或有其他用途但我沒深入研究)，例如我們常常在Rails Console中下某個query後，會在console出現下面的訊息：

![sql.active_record事件](/assets/images/article_about/rails_sql_log.png){: .align-center}

這個訊息就是來自Rails內建的`sql.active_record`事件。當我們透過ActiveRecor向資料庫送出請求時，會送出一個`sql.active_record`事件，而在某處有個Subscriber會處理該事件會產生log。今天就是要來把這個Subscriber找出來他到底藏在Rails的哪個角落！

## Active Support Subscriber

Rails Guide中特別的頁面在沒有特別介紹他在哪裡設定了Subscriber，因此我到了Rails Guide的API去找答案，找到了Class`ActiveSupport::Subscriber`在負責訂閱事件。根據官方文件描述這個Class的用途：

*引用自Rails Guide API*

> `ActiveSupport::Subscriber` is an object set to consume `ActiveSupport::Notifications`. The subscriber dispatches notifications to a registered object based on its given namespace.
>
> An example would be an Active Record subscriber responsible for collecting statistics about queries:

{% highlight ruby linenos %}
module ActiveRecord
  class StatsSubscriber < ActiveSupport::Subscriber
    attach_to :active_record

    def sql(event)
      Statsd.timing("sql.#{event.payload[:name]}", event.duration)
    end
  end
end
{% endhighlight %}

> After configured, whenever a “sql.active_record” notification is published, it will properly dispatch the event (`ActiveSupport::Notifications::Event`) to the sql method.
>
> We can detach a subscriber as well:

{% highlight ruby linenos %}
ActiveRecord::StatsSubscriber.detach_from(:active_record)
{% endhighlight %}

簡單翻譯如下：

> `ActiveSupport::Subscriber`是個用來處理`ActiveSupport::Notifications`的物件。Subscriber會根據namespace將事件的通知傳遞到註冊事件的物件。
>
> 範例：Active Record的subscriber中負責收集ActivceRecord執行query時的資訊：
>
> ***example code showed before***

> 當subscriber設置好後，每當`sql.active_support`事件發出通知時，會自動將事件(`ActiveSupport::Notifications::Event`)傳遞到`sql`方法。
>
> 也可以將subscriber與已經對接的事件分離。
>
> ***example code showed before***

由此可知，原本是透過`ActiveSupport::Notification.subscribe`來訂閱事件，現在可以利用`ActiveSupport::Subscriber`來訂閱事件。所謂的namespace則是依照原本事件的naming來進行設置。`ActiveSupport::Subscriber`中定義了兩個的class method：
1. `attach_to`: 負責將library發出的事件對接到subscriber，參數的type為symbol，其value為事件的naming convention中library的部分。
2. `detach_from`： 負責將已經對接的subscriber分離，參數的設置與`attach_from`一樣。

naming convention中event的部分，則需要我們自己定義instance method，其名稱與event的名稱相同，這麼一來當事件發出通知時，subscriber會將通知分配到該instance method。因此原本定義在`ActiveSupport::Notification.subscribe`的block內的程式碼就變成寫在instance method內。

有個設置subscriber的base class(`ActiveSupport::Subscriber`)後，我們可以透過繼承這個class來設定subscriber，如果剛剛有注意到前面範例的程式碼的話便可以發現，實際上Rails也是這麼做的！Rails有個專門用於產生log的Subscriber－`ActiveSupport::LogSubscriber`，範例裡面Class`ActiveRecord::StateSubscriber`繼承自`ActiveSupport::LogSubscriber`，而`ActiveSupport::LogSubscriber`繼承自`ActiveSupport::Subscriber`。

## Lograge

接下來來介紹一個Gem－Lograge，這個Gem可以將原本Rails中`ActionController`以及`ActionCable`的log有些多餘的資訊刪掉，讓我們利用log來debug時，更容易搜索或閱讀log。今天就來翻翻source code來看看它是怎麼辦到的！

{% highlight ruby linenos %}
# /lib/lograge.rb
module Lograge
  def remove_existing_log_subscriptions
    ActiveSupport::LogSubscriber.log_subscribers.each do |subscriber|
      case subscriber
      when ActionView::LogSubscriber
        unsubscribe(:action_view, subscriber)
      when ActionController::LogSubscriber
        unsubscribe(:action_controller, subscriber)
      end
    end
  end

  def unsubscribe(component, subscriber)
    events = subscriber.public_methods(false).reject { |method| method.to_s == 'call' }
    events.each do |event|
      ActiveSupport::Notifications.notifier.listeners_for("#{event}.#{component}").each do |listener|
        ActiveSupport::Notifications.unsubscribe listener if listener.instance_variable_get('@delegate') == subscriber
      end
    end
  end

  def attach_to_action_controller
    Lograge::LogSubscribers::ActionController.attach_to :action_controller
  end
end
{% endhighlight %}

在`/lib/lograge.rb`這個檔案內，可以看到Lograge定義了`remove_existing_log_subscriptions`方法，用`ActiveSupport::LogSubscriber.log_subscribers`找出所有的subscriber，並且取消了將這兩個subscriber訂閱的事件。也定義了`attach_to_action_controller`方法，將`Lograge::LogSubscribers::ActionController`對接到事件名稱library的部分是`action_controller`內的事件。

Lograge自己建立了一個Class`Lograge::LogSubscribers::ActionController`來代替原本的`ActiveSupport::LogSubscriber`，所以接下來來看看這檔案`/lib/lograge/log_subscribers/action_controller.rb`定義了哪些東西。

{% highlight ruby linenos %}
module Lograge
  module LogSubscribers
    class ActionController < Base
    def process_action(event)
      process_main_event(event)
    end

    def redirect_to(event)
      RequestStore.store[:lograge_location] = event.payload[:location]
    end

    def unpermitted_parameters(event)
      RequestStore.store[:lograge_unpermitted_params] ||= []
      RequestStore.store[:lograge_unpermitted_params].concat(event.payload[:keys])
    end
  end
end
{% endhighlight %}

在這個class內，定義了三個instance method，`process_action`、`redirect_to`、`unpermitted_parameters`，而這三個方法的目的就是要處理事件名稱是`process_action.action_controller`、`redirect_to.action_controller`、`unpermitted_parameters.action_controller`的事件！而這三個事件也就是上一篇文末提的，Rails原本就提供有關於`ActionController`的事件。

----------
Rails Instrumentation API的發布事件還有訂閱事件的介紹就到這邊。這次工作上接到的任務讓我又學到了Rails的其中一種黑魔法，而且也追了兩個Gem的source code來看看他們是怎麼透過Rails Instrumentation API以及Active Support Subscriber來實作產生log message。

<br>

此篇文主要目的在記錄學習程式語言時的筆記，如果有錯誤之處請不吝於留言給予指教，我會很感謝您給的回饋！
以上內容參考自下列網站/文章：
1. [Rails Guide API](https://api.rubyonrails.org/classes/ActiveSupport/Subscriber.html)
2. [Lograge](https://github.com/roidrage/lograge)