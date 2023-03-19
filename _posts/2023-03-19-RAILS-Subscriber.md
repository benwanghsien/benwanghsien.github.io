---
title: "Rails: Active Support Subscriber"
excerpt: "研究Rails的黑魔法如何將我們需要的資訊在Console顯示出來(Episode 2 of 2)"
tags:
  - "Ruby"
  - "Ruby on Rails" 
  - "Programming"
---

本篇接續上一篇介紹Rails Instrumentation API的文章，如果不清楚或忘記Active Support Instrumentation是做什麼的歡迎回顧[上一篇](https://benwanghsien.github.io/RAILS-Instrumentation/)文章。

上篇文末介紹了Rails有提供了許多的事件可以使用，主要是用來產生log（或有其他用途但我沒深入研究），例如我們常常在Rails Console中下某個query後，會在console出現下面的訊息：

![sql.active_record事件](/assets/images/article_about/rails_sql_log.png){: .align-center}

這個訊息就是來自Rails內建的`sql.active_record`事件。當我們透過ActiveRecor向資料庫送出請求時，會送出一個`sql.active_record`事件，而在某處有個Subscriber會處理該事件會產生log。今天就是要來把這個Subscriber找出來他到底藏在Rails的哪個角落！

## Active Support Subscriber
Rails Guide中特別的頁面在沒有特別介紹他在哪裡設定了Subscriber，因此我到了Rails Guide的API去找答案，找到了Class`ActiveSupport::Subscriber`在負責訂閱事件。根據官方文件描述這個Class的用途：

*引用自Rails Guide API*

> `ActiveSupport::Subscriber` is an object set to consume `ActiveSupport::Notifications`. The subscriber dispatches notifications to a registered object based on its given namespace.
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
> We can detach a subscriber as well:

{% highlight ruby linenos %}
ActiveRecord::StatsSubscriber.detach_from(:active_record)
{% endhighlight %}

簡單翻譯如下：

> `ActiveSupport::Subscriber`是個用來處理`ActiveSupport::Notifications`的物件。Subscriber會根據namespace將事件的通知傳遞到註冊事件的物件。

在

由此可知，原本是透過`ActiveSupport::Notification.subscribe`來訂閱事件，現在可以利用`ActiveSupport::Subscriber`來訂閱事件。

在公司的專案中使用Lograge來代替原本ActionController產生的log，這個Gem可以將原本Rails的log有些多餘的資訊刪掉，讓我們利用log來debug時，更容易搜索或閱讀log。而這次的需求是需要取得Elasticsearch的執行時間，在專案中用了Searchkick這個Gem去向Elasticsearch送出request，