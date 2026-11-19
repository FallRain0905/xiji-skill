---
name: xiji-homework
description: 希冀平台（course.educg.net）Python 课程作业的查看、编写、提交与判错排查全流程。当用户提到"希冀平台、 educg、课程作业、看看作业、完成作业、提交作业、作业判错/输出错误/格式错误"或发来平台题面让你做题时使用。覆盖登录（含验证码）、三类题型（CodeMirror 编程题 / 程序片段填空 / 单选题 / 文件上传）、提交按钮定位、结果核对与详细评判排查。
---

# 希冀平台作业流程

一套经过实战验证的 course.educg.net 作业自动化流程。核心原则：**每个动作后用最便宜的方式观察结果；判错时永远以「详细评判」的期望输出为准，不信题面样例的排版。**

## 0. 前置

- 用 `browser-use:control-browser` skill 的浏览器流程操作，`getForUrl("https://course.educg.net/")` 选择后端。
- 凭据从用户记忆/用户提供处获取，不要硬编码在本 skill 里。

## 1. 登录（每次会话大概率要重做）

1. 打开任意平台页（如 `assignment/index.jsp`）；若正文为空且 URL 是 `indexcs/simple.jsp?loginErr=0`，会话已过期。
2. 回 `https://course.educg.net/`，三个输入框 id 固定：`#username`、`#password`、`#captchaCode`。
3. 验证码是图片 `img[src="/cgjiaoyan"]`，位置约在视口 `{x:890,y:115,width:240,height:200}` 的 clip 内。`fill` 账号密码后截图读图填验证码。**验证码大小写敏感**。
4. 点登录按钮不要用 getByRole（会超时），用：
   ```js
   document.getElementById('cgstuloginbtn').click()
   ```
5. 成功标志：跳转到 `main.jsp`。失败常见原因是验证码读错——重截重试即可。

## 2. 列出作业

- 登录后 `assignment/index.jsp` 左侧栏「当前作业」「历史作业」列出全部作业。页内提取：
  ```js
  [...document.querySelectorAll('a')]
    .map(a => a.getAttribute('href') || '')
    .filter(h => /assignID=\d+/.test(h) && !/programList|programFillGap|judgeDetails|singleOption|problemContent/.test(h))
  ```
- 主页（main.jsp）能看到各作业的截止时间与倒计时。
- 某作业总览页：`assignment/index.jsp?courseID=<C>&assignID=<A>`，一页含全部题目、分值、得分、每条测试数据的评判结果。**核对得分永远回这一页**，不要信题页的结果面板。

## 3. 四种题型与提交方式

### 3.1 编程题（programList.jsp）
- 答题区是 CodeMirror。写入并提交：
  ```js
  document.querySelector('.CodeMirror').CodeMirror.setValue(src);
  document.getElementById('cgSubmitBtn').click();
  ```
- 提交后等 4~6 秒，回总览页核对得分。
- **源文件上传变体**：无 CodeMirror，只有 `<input type="file" id="CGFILE">`。浏览器不支持系统文件框，绕过：
  ```js
  const dt = new DataTransfer();
  dt.items.add(new File([src], "main.py", { type: "text/x-python" }));
  input.files = dt.files;
  input.dispatchEvent(new Event('change', { bubbles: true }));
  document.getElementById('cgSubmitBtn').click();
  ```

### 3.2 程序片段填空（programFillGapList.jsp）
- 填空是 `<textarea name="answerInput1..N">`，直接 `.fill()` 后派发 input/change 事件，再点 `cgSubmitBtn`。隐藏字段 `answerN` 为空是正常的，平台读 textarea。
- **缩进规则（最容易翻车）**：
  - 多数空位平台会按模板基础缩进把内容**整体再缩进一次**——内容只写相对缩进、首行顶格；
  - 但也有空位基准缩进为 0（模板静态代码行尾无空格），内容必须**自带 4/8 空格缩进**。
  - 判断方法：看该空位前面 `code.cgcode` 元素的 `textContent` 行尾——`"\n    "` 结尾表示平台会加 4 空格（内容顶格写）；`"\n"` 结尾表示不会（内容自己带缩进）。
  - 判错信息若含 `IndentationError`，就是这里错了。

### 3.3 单选题（singleOptionList.jsp）
- 选项是隐藏样式的 RadioButton，Playwright `.check()` 会超时。页面内 JS：
  ```js
  const t = document.querySelector('input[type=radio][value="B"]');
  t.checked = true;
  t.dispatchEvent(new Event('change', { bubbles: true }));
  const f = [...document.forms].find(x => /stuAnswerHandler/.test(x.action || ''));
  f.submit();
  ```
- 底部「提交并继续下一道题」是个 `<a>`，**只跳转不提交**，必须如上直接 submit 表单。
- 成功标志：题面出现「已提交」。总览页单选行只显示「已提交」，不显示得分。

### 3.4 文件上传题（problemContent.jsp）
- 过期后页面**没有任何上传控件**（只剩说明文字），无法补交——截止前务必提交。

## 4. 判错排查

1. 总览页每题的「详细」链接指向 `judgeDetailsRedirect.jsp?assignID=<A>&problemID=<P>`（跟随中转到 judgeDetails.jsp）。
2. 展开差异对比：
   ```js
   [...document.querySelectorAll('img[src="/images/expandall.gif"]')].forEach(i => i.click());
   ```
3. 逐条对照「错误输出 vs 期望输出」。常见判错原因见 references/pitfalls.md。

## 5. 做题质量流程（重要）

1. **先本地验证再提交**：把完整代码存临时文件，用样例输入跑一遍，输出与样例逐字符比对（用 `repr()` 看行尾空格/换行差异）。平台判题对格式零容忍。
2. **提示符抠字符**：带 `input("提示")` 的题，提示串的每个空格都要和期望输出一致（如 `How many numbers are there?` 的 `?` 后无空格）。
3. **行尾空格**：期望输出「行尾有空格且无换行」时用 `print(x, end=" ")`；要求行尾无空格时用 join 后统一 print。
4. **文件不存在回退**：读文件题若不确定评测是否提供文件，用 try/except FileNotFoundError 回退读 stdin，两种情况都能过。
5. **CSV 输出**：csv.writer 会给含逗号的字段自动加双引号包裹。判题期望无引号文本时，改用 `f.write()` 手写。
6. **标点串含弯引号**：题面给的标点替换串可能含 U+2018/2019/201C/201D，复制时别丢——丢了会导致 `China’s` 这类词切不开。
7. **有序输出的同次数排序**：词频等题同次数元素的顺序可能是「首次出现顺序」而非字母序，以期望输出为准（Python 的 sorted 是稳定排序，`key=lambda kv: -kv[1]` 可保留插入序）。
8. **样例输出是图片**：innerText 抽不出来时，JS 取 img 的 `getBoundingClientRect()` 后按 clip 截图读图。

## 6. 作业文档归档（每份作业做完后必做）

一个作业全部提交并核对满分后，**新建该作业的归档文档**：`<归档目录>/<assignID>-<作业名>.md`。目录用作业归档专用的文件夹（向用户确认或用既有约定，如 `python作业归档/`）。

文档为 Markdown，固定五个部分（参考已归档的 `51600-CSV文件读写-课堂练习.md`）：

1. **头部信息**：作业名、assignID、做题时间、最终得分、截止时间、得分总表（题号/题目/类型/分值/得分）。
2. **原题**：每题引用题面（问题描述、输入输出形式、样例）。填空题要把模板骨架贴出来并用 `____(N)____` 标出空位；样例输出是图片的要转写成文本。
3. **答案**：最终提交且通过全部测试点的代码（填空题按空位给答案）。只放最后通过的版本，不放中途错版；错版内容放进解析里的踩坑小节。
4. **解析**：逐空/逐段讲为什么这样写、涉及的语言特性。**踩坑记录单独列出**：每次被判错的经过（第几次提交、报什么错、根因、怎么修的）——这是文档最有价值的部分。
5. **知识点总结**：从本作业提炼的通用知识点（语言特性 + 平台判题规则），给可运行的代码示例而非纯文字。

写作要求：
- 原题用引用块（`>`）与解析区隔；
- 代码块标注语言（```python）；
- 踩坑记录写「现象 → 根因 → 修法」，供以后同类题直接对照；
- 文档独立成篇，不依赖对话上下文也能看懂。

## 7. 收尾

- 全部题目提交后回总览页核对：无「还未提交」、无「输出错误/格式错误」项（0 分选做题的「部分错误」若题面已声明可忽略则不算遗留）。
- 完成第 6 步的作业文档归档后，才算这份作业结束。
- 把本次新学的坑（格式细节、平台行为）追加到用户记忆，而不是只留在对话里。

## references/pitfalls.md

排查「输出错误/格式错误」时按此清单对照，读完再改代码重提：
- 提示符多了/少了空格（最常见的格式错误来源）
- 行尾空格、结尾换行差异（用 repr() 对比）
- 填空缩进被平台二次缩进或缺失
- 整数结果误用 `.2f`（输出 36.00 判错）
- eval 遇到空格分隔输入（应改 split）
- CSV 引号包裹、编码（utf-8 vs utf-8-sig）
- 集合输出顺序不稳定（题面声明可忽略的除外）
