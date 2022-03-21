---
tags: Blog
---

# Ruby 中如何將 Array 的元素轉換成 String

###### tags: `Ruby`, `Array`

## 透過`each`迭代出 array 的元素

最容易理解的方法，透過迭代的方式，將 array 內的元素逐一取出來後，將取出來的字串使用`+`方法串接起來

{% highlight ruby linenos %}
greeting_arr = ["This", "is", "note", "about", "how", "to", "manipulate", "string", "on", "Ruby"]

result_str = ""

greeting_arr.each.with_index do |str, idx|
str += " "
result_str += str
end

puts result_str
{% endhighlight%}

最後一行程式碼中得到的輸出結果為

```bash=
This is note about how to manipulate string on Ruby
```

使用以上的方式雖然在 terminal 上看不出來最後面有個空格，不過實際上在最後一個單字 Ruby 的後面存在一個空格，改用`p`方法來印出字串就可以發現

```ruby=
greeting_arr = ["This", "is", "note", "about", "how", "to", "manipulate", "string", "on", "Ruby"]

result_str = ""


greeting_arr.each do |str|
  str += " "
  result_str += str
end

p result_str # 這裡改成使用p印出結果
```

```bash=
"This is note about how to manipulate string on Ruby "
```

如果想要消除最後一個空格，就需要加上`if`來判斷是不是從 array 中取得最後一個字串，如果是最後一個的話就改成加上`.`

```ruby=
greeting_arr.each do |str|
  if str == greeting_arr.last # 當取得的是最後一個字串，改成加上句號
    str += "."
    result_str += str
  end

  str += " "
  result_str += str
end

p result_str
```

## 使用 array 的 join 方法
