---
title: 404 Not Found
date: 1970-01-01 00:00:00
permalink: /404.html
comments: false
---

<!-- markdownlint-disable MD039 MD033 -->

很抱歉，你目前存取的頁面並不存在。

預計將在約 <span id="timeout">500</span> 秒後返回首頁。

如果你很急著想看文章，你可以 **[點這裡](/)** 返回首頁。

<script>
let countTime = 500;

function count() {

  document.getElementById('timeout').textContent = countTime;
  countTime -= 1;
  if(countTime === 0){
    location.href = '/';
  }
  setTimeout(() => {
    count();
  }, 1000);
}

count();
</script>
