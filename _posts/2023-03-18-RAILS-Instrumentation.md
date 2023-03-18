---
title: "Rails: Active Support Instrumentation"
excerpt: "研究Rails的黑魔法如何將我們需要的資訊在Console顯示出來"
tags:
  - "Ruby"
  - "Ruby on Rails"
  - "Programming"
---

最近在工作上接到一個任務，要將我們替使用者用送來的request的產生的log message，加上一些讓我們需要的資訊。如果該使用者的request有向Elasticsearch搜尋資料時，要在log message內紀錄Elasticsearch的執行時間。所以便開始去研究Rails究竟是用了什麼黑魔法取的我們需要資訊並且產生log。

Active Support是Rails的核心功能之一，主要有下列的用途：擴充Ruby、提供utility等等。今天要介紹的是Instrumenation API，

在公司的專案中使用Lograge來代替原本ActionController產生的log，這個Gem可以將原本Rails的log有些多餘的資訊刪掉，讓我們利用log來debug時，更容易搜索或閱讀log。而這次的需求是需要取得Elasticsearch的執行時間，在專案中用了Searchkick這個Gem去向Elasticsearch送出request，