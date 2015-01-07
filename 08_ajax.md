---
layout: default
title: 评论提交 ajax 化
---

最近两个月我跟着老师学习画素描，有时候把基本的形状和明暗关系找到之后就不想往下画了，老师说不行，你不深入刻划一下
局部，真正巧妙的东西还是领悟不到。咱们还是来看这个评论框，现在如果提交评论，那页面会整个
刷新，于是会跳到页顶，这个用户体验不是很好。所以作为真正热爱写代码的你，当然是希望再进一步刻划一下，
所以这集来把评论功能 [ajax 化](http://guides.rubyonrails.org/working_with_javascript_in_rails.html)。

### 更改前端 html

前端最关键的一步就是添加 `remote: 'true'` 。到 _comment_box.html.erb 中

{% highlight erb %}
- <%= form_for(Comment.new(:issue_id => issue.id, :user_id => current_user.id)) do |f| %>
+ <%= form_for(Comment.new(:issue_id => issue.id, :user_id => current_user.id), remote: 'true') do |f| %>
{% endhighlight %}

这样转换成的 html 其实变化不大，就是多了 `data-remote="true"` 这几个字，但是注意在 application.js 文件中
有 `require jquery_ujs`，真正的魔术都在 jquery_ujs 这个 js 文件中呢，里面的代码会把这里的表单提交自动变成一次
ajax 提交。具体细节不用管，真正要关心的就是后台 log 中的变化。如果出现
 `Processing by IssuesController#show as JS` (非 ajax 请求是请求 html) 证明前端要做的修改就弄好了。
下面最到后端的 controller 里面来提供 JS 格式的响应。

### 修改后端

{% highlight ruby %}
def create
_ c = Comment.new(comment_params)
- c.save
- redirect_to c.issue
+ @comment = Comment.new(comment_params)
+ @comment.save
+ respond_to do |format|
+   format.js
+ end
end
{% endhighlight %}

然后创建 app/views/comments/create.js 注意这样位置不能敲错，不然 rails 就找不着这个文件了。

先在里面写个 `alert("hello");` 这样在前端提交一下评论，就可以看到后端给发送过来 create.js 的内容，并在
浏览器里执行了。好，这就是基本流程，挺简单吧。

所以现在就要在 create.js 添加适当的内容，来把新发布的评论添加到页面中来。


我想要放到 create.js 中的是大概这些内容：
{% highlight erb %}
$(".replies").append("<%= j render(partial: 'shared/comment', locals: {c: @comment}) %>");
{% endhighlight %}

注意： `append()` 括号的内容，外面用双引号，里面有单引号。不要同时都用单引号或是双引号。
另外，这里的 `j` 是 `escape_javascript` 的简写形式。目的是把 partial 中的引号和回车进行转义，否则 render 出来的
内容直接放到 create.js 中就会造成 js 的代码错误了。


需要一个 shared/_comment.html.erb 但是现在没有，所以可以来重构一下 _comment_list.html.erb 首先要把它重命名为
_comment.html.erb 。然后里面的内容做出下面的调整。

{% highlight erb %}
- <% comments.each do |c| %>
- <% end %>
{% endhighlight %}

这样 issues/show.html.erb 中的代码也要调整。把原有的使用 shared/_comment_list.html.erb 的代码改为

{% highlight erb %}
<% @comments.each do |c| %>
  <%= render partial: 'shared/comment', locals: {comment: c} %>
<% end %>
{% endhighlight %}

然后再把

{% highlight erb %}
  <%= render partial: 'shared/comment_box', locals: {issue: @issue} %>
{% endhighlight %}

往下移动一行，也就是把它从 class 名为 `replies` 的 div 中拿出来，配合 create.js 中的 `$(".replies").append()`。

发一条评论，ajax 生效了，哈哈。但是评论框没有被清空，所以到 create.js 里再来添加一行

{% highlight js %}
$(".reply textarea").val('');
{% endhighlight %}

### devtools 中的调试技巧

http://happycasts.net/episodes/107