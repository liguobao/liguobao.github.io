---
title: How To Delete All Your Zhihu Answers and Ideas At Once?
slug: how-to-delete-all-answers-on-zhihu
date: 2024-03-01 08:00:00+0000
categories:
  - Zhihu
tags:
  - 2024
---

Source: https://www.zhihu.com/question/265203478/answer/3415024932

Backup. For when you need it.

JavaScript snippet to batch‑delete short answers on a page:

```javascript
// Delete a single answer
function deleteAnswer(answerId) {
  var xhr = new XMLHttpRequest();
  xhr.withCredentials = true;
  xhr.addEventListener("readystatechange", function () {
    if (this.readyState === 4) {
      console.log(this.responseText);
    }
  });
  xhr.open("DELETE", "https://www.zhihu.com/api/v4/answers/" + answerId);
  xhr.setRequestHeader("authority", "www.zhihu.com");
  xhr.setRequestHeader("accept", "*/*");
  xhr.setRequestHeader("accept-language", "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7");
  xhr.send();
  console.log('delete answer: ' + answerId + ' success');
}

// Clean all answers on the current page (shorter than N characters)
function clearAllAnswers(longer_than = 100) {
  let answerItemList = document.querySelectorAll('.AnswerItem');
  for (let answerItem of answerItemList) {
    let answerContentDev = answerItem.querySelector('.RichContent-inner');
    if (!answerContentDev) continue;
    let ans_id = answerItem.getAttribute('name');

    let answerContent = answerContentDev.innerText;
    if (answerContent.length >= longer_than) {
      console.log(`answer:${ans_id} is too long, skip`);
      continue
    }
    console.log(`delete answerContent: ${answerContent}`);
    deleteAnswer(ans_id);
  }
}
```

Steps

1) Go to your answers list page, open the browser DevTools console (F12).

2) Paste the code above into the Console and press Enter.

3) Run `clearAllAnswers(50)` and press Enter.

Note: 50 means answers longer than 50 characters are skipped.

4) Keep paging through your answers and repeat:

`clearAllAnswers(50)`

Done.

