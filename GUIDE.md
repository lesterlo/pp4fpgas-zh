# 翻譯之前

Parallel Programming for FPGAs這本書的原作採用的是`latex`進行內容的編寫和排版。為了提高翻譯寫作的速度和協作的效率，本次翻譯任務選擇了在`GitHub`這個平台上進行協作，採用了`Markdown`使得譯者可以專注文字內容而不是排版樣式，安心寫作。

這也給參與翻譯任務的諸位帶來了一點小挑戰，需要諸位事先熟悉一下`GitHub`平台的使用、`git`的使用以及`Markdown`語言的規範，下面是相關的參考鏈接給諸位快速上手。

## 排版約定

- [排版約定](https://xupsh.github.io/pp4fpgas-cn/RULES.html)

## 編輯器

一個界面美觀、交互UI設計良好的編輯器可以幫我們節省很多力氣，這裏我們比較推薦使用以下幾款編輯器來進行翻譯工作

- [Atom](https://atom.io/)
- [VS Code](https://code.visualstudio.com/)

## `Markdown`語言

> 事實上這篇README就是用Markdown寫成的 :)

- [認識與入門Markdown](https://sspai.com/post/25137)
- [Markdown 語法説明 (簡體中文版)](http://wowubuntu.com/markdown/basic.html)

## `git`

git可以説是現在最為流行的版本管理工具了。

- [廖雪峯的git教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)
- [猴子都能懂的GIT入門](https://backlog.com/git-tutorial/cn/)

其實最常用的命令無非下面幾條

### 下載git庫到本地

```console
git clone https://github.com/xupsh/pp4fpgas-cn.git
```

### 保存本地的修改並上傳到雲端服務器(GitHub)

```console
git add -A
git commit -m "最近的修改裏都做了什麼"
git pull
git push
```

## `GitHub`的Pull Request操作

在`GitHub`上進行協作，通常採用的方式是先各自fork一份到自己的個人帳户，經過一段時間的工作之後，通過pull request的方式，將自己的工作內容提交到公共項目帳户中，而pull request之後往往還需要進行review才能正式進入公共項目。

[github的pull request官方文檔](https://help.github.com/articles/about-pull-requests/)

### Pull Request 的流程

- 第一步，你需要把別人的代碼，克隆到你自己的倉庫，Github 的術語叫做 fork。

- 第二步，在你倉庫的修改後的分支上，按下"New pull request"按鈕。

![new pr](http://www.ruanyifeng.com/blogimg/asset/2017/bg2017071802.png)

- 這時，會進入一個新頁面，有Base 和 Head 兩個選項。Base 是你希望提交變更的目標，Head 是目前包含你的變更的那個分支或倉庫。

![compare changes](http://www.ruanyifeng.com/blogimg/asset/2017/bg2017071806.png)

- 第三步，填寫説明，幫助別人理解你的提交，然後按下"create pull request"按鈕即可。

- PR 創建後，管理者就要決定是否接受該 PR。對於非代碼變更（比如文檔），單單使用 Web 界面就足夠了。但是，對於代碼變更，Web 界面可能不夠用，需要命令行驗證是否可以運行。
