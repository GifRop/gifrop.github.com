title: 一些hexo博客搬家的问题
date: 2014-06-26 21:45:50
tags: [Hexo]
---
这个博客在win上搭建的，最近准备把博客从工作机器上搬运到自己的本子，遇到了一些问题。找到了解决方案，这里记录一下。 <!-- more -->
第一个:  
{% codeblock lang:js %}
<%- partial('_partial/head') %>
<%- partial('_partial/header') %>
<%- body %>
<%- partial('_partial/footer') %>
<%- partial('_partial/after-footer') %>
{% endcodeblock %}
这个问题很奇怪，在新机器上一直有这问题。找了很久没找到答案，自己猜测应该是某些东西没安装。最后找到了ejs没安装，解决方案：  
{% codeblock lang:js %}
$ npm install hexo-renderer-ejs --save
{% endcodeblock %}
解决了上面的发现还有这个:  
{% codeblock lang:js %}
<%- partial('_partial/head') %>
<%- partial('_partial/header') %>
<%- body %>
<% if (theme.sidebar && theme.sidebar !== 'bottom'){ %> <%- partial('_partial/sidebar') %> <% } %>
<%- partial('_partial/footer') %>
<%- partial('_partial/mobile-nav') %> <%- partial('_partial/after-footer') %>
{% endcodeblock %}
解决方案:  
{% codeblock lang:js %}
$ npm install hexo-renderer-stylus --save
$ npm install hexo-renderer-marked --save
{% endcodeblock %}
  
博主做为一个业余圈外人士，还出现了另外的错误:  
{% codeblock lang:js %}
[wron]no layout
{% endcodeblock %}
查看目录的时候发现，主题文件夹里没有内容。把主题目录拷贝过来就好了。或者把config.yml文件里的主题更改成默认主题。  
继续hexo d发现奇怪错误:
{% codeblock lang:js %}
[info] Start deploying: github
[info] Clearing .deploy folder...
[info] Copying files from public folder...
Assertion failed: (item->nowildcard_len <= item->len && item->prefix <= item->len), function prefix_pathspec, file     pathspec.c, line 308.
fatal: Unable to create './.git/index.lock': File exists.
If no other git process is currently running, this probably means a
git process crashed in this repository earlier. Make sure no other git
process is running and remove the file manually to continue.
fatal: 'github' does not appear to be a git repository
fatal: Could not read from remote repository. 
Please make sure you have the correct access rights
and the repository exists.
{% endcodeblock %}
开始一直按照网络上的删除index.lock，发现一直不行。折腾很久。  
最后终于搞定了
{% codeblock lang:js %}  
删除.deploy  
hexo g  
hexo d  
{% endcodeblock %}

The end.
