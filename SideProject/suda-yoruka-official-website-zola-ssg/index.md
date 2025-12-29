+++
title = "打造 VTuber 須多夜花官方網站：Zola SSG 與 AI 協作開發實錄"
description = "分享使用 Zola 靜態網站生成器打造 VTuber 須多夜花官方網站的開發心得。採用 SDD 規格驅動開發，讓 AI 自動實作 Tera 模版遷移、Sass 樣式整合，並透過 GitHub CI/CD 部署至 Cloudflare Pages。"
date = "2025-12-29T09:27:20.808Z"
updated = "2025-12-29T09:27:20.144Z"
draft = true

[taxonomies]
tags = [ "Cloudflare", "Zola", "AI", "VTuber", "Livestream" ]
licenses = [ "All Rights Reserved" ]

[extra]
withAI = "本文使用 GitHub Copilot Claude Opus 4.5 協助撰寫"
+++

> {{ color(body="須多夜花", color="#e34565ff") }}官方網站  
> <https://SudaYoruka.com>

這篇文章記錄了我為 VTuber {{ color(body="須多夜花", color="#e34565ff") }} 打造官方網站的開發歷程。我是琳，是須多夜花的 IP 持有者及營運負責人，同時也是這個官方網站的獨立開發者。

<!-- more -->

## 為什麼要做官方網站

製作官網其實是我個人的想法。在安排夜花的復出計劃時，原本並不包含這個專案。身為前網頁工程師，我本來就很喜歡做網站。擁有個人網站的 VTuber 並不多，{{cr(body="因為開發成本高")}}，除非像我這樣自己動手，才有可能實現。

我的專業是網頁後端及系統整合，{{cr(body="這種以前端和美術為主的網站並非我的強項")}}。原本計劃委託網頁公司製作，但詢價後因預算不足，最終決定自己來。也因此，我們得到了一個「還過得去」的成品，它更像是我的個人專案。

## 技術選型：為什麼選擇 Zola

我採用了一個尋常網頁公司絕對不會使用的框架：**Zola SSG**（Static Site Generator）。比起「官方網站」，這個成品更接近「官方部落格」的型態。

Zola 使用相對小眾的 `Tera` 模版語言搭配 `Sass` 建構靜態網頁。它本身是靜態部落格生成系統，你正在閱讀的這個部落格也是使用 Zola 建置。Zola 的設計適合製作一篇一篇內容的網頁，這正好符合我們官網的需求。

更重要的是，{{cg(body="Zola 允許使用 Markdown 撰寫內容")}}。這意味著未來的編輯者——大家最愛的夜花——不需要懂 HTML 或 CSS 就能發佈文章。不過她很懶，總是叫我幫她寫。好吧，畢竟網站是我想做的。

## 版型遷移：從 Panini 到 Tera

版型我在 themeforest 上挑了很久，最後選定 [FriendKit](https://cssninja.io/product/friendkit) 這個社群網站版型，打造出像社群平台的官方網站。這個版型其實很豐富，一個標準社群網站該有的頁面它全都有設計，但我們只用到了其中幾頁。

原始的 HTML 版型使用 Panini 模版系統，這並不是我要的。在實作時我把它改成了 Tera 模版。Sass 基本上是直接遷移過來，然後砍掉多餘的部份。

這是很大的工作量，不過沒關係，反正不是我要做 😉

<!-- pic: 網站首頁截圖 -->

## 讓 AI 來寫程式碼

這個網站是使用我流的 **SDD**（Specification-Driven Development）規格驅動開發，{{cg(body="整個開發流程外包給 AI 來執行")}}。

我在本地準備好了開發需要的所有知識脈絡，包含 Tera 和 Zola 的撰寫指引。我準備了 `AGENTS.md` 作為開發憲章，然後使用 GitHub 的 Issue 和 PR 系統做任務管理。

具體流程如下：我讓 VSCode 的 GitHub Copilot（Claude 模型）規劃完整的工作，並使用 MCP 在 GitHub 開 Issue。每個 Issue 的內容都是一個詳盡的開發計劃，我告訴它要寫得詳細到連菜逼八也能照著實作。接著使用 GitHub Coding Agent 依序實作每一個 Issue。我負責審閱 PR。

<!-- pic: Issue 清單截圖 -->

<!-- pic: 其中一個詳盡的 Issue 截圖 -->

<!-- pic: PR 成果截圖 -->

<!-- pic: AGENTS.md 截圖 -->

最後的成品中，{{cg(body="由我手寫的程式碼其實非常少")}}。

{{cr(body="Zola 和 Tera 並不是常見的技術堆疊，一般通用 AI 不太能成功撰寫相關程式碼。")}}這個專案能夠成功，仰賴的是我對這些工具的熟悉程度。我在開發過程中提供了足夠的知識脈絡，才使得 AI 有辦法完成任務。我並不推薦其他人貿然走這條路。

當你問我說：

{% chat(speaker="user") %}
哇，這個網站是你自己做的嗎？
{% end %}

{% chat(speaker="jim") %}
其實我是用 AI 做的啦www
{% end %}

的時候，我指的是像上面描述的那樣。事實上，別說外行人，我想很多工程師都不一定能駕馭這種開發流程。SDD 和 Context Engineering 是 2025 年最前沿的開發方法論。

如果你對 SDD 有興趣，可以參考我的另一篇文章：[規格驅動開發 (Spec-Driven Development) 與 AI 協作全流程實戰](https://聆.tw/AI/sdd-ai-copilot-codex-devops-workflow/)

## 讓 AI 替 VTuber 寫文章

在網站差不多完成後，我寫了一個「用 {{ color(body="須多夜花", color="#e34565ff") }} 人格來寫作」的 GitHub Copilot Instruction 指示詞。這個 prompt 主要著重在產出正確的 Markdown 格式，再加上一點點的角色扮演元素。你可以想像是[設計角色扮演 LLM Prompt 的實務心得](@/AI/design-roleplay-llm-prompts/index.md)與部落格寫作指引的結合體，但不包含夜花本人的個性參考素材。

像是官網上的這篇[「我回來了！這次，我不會再消失」](https://sudayoruka.com/posts/2025-12-21-intro/)介紹文，初始文案是夜花自己寫的，我再用上述的指示詞擴寫為最終文字。

偷偷說，最後那句「請多指教，我所愛的人們」是她寫給我的，去跟她說她會害羞喔 🤫

## 部署流程：GitHub Actions 與 Cloudflare Pages

這個專案使用 GitHub Workflows 進行 CI/CD，部署至 Cloudflare Pages。我們的部署流程比一般的 Zola 網站複雜一些。

除了標準的 Zola 建構之外，還有兩個額外的 Python 指令碼需要執行：一個負責取得 YouTube 影片清單，另一個負責取得[聲音按鈕](https://sound-buttons.click/yoruka)清單。這些指令碼會處理 YouTube RSS 後產生我需要的 JSON 資料檔案。

由於這是靜態網頁，為了讓影片清單保持更新，{{cg(body="CD Pipeline 每天都會定期執行一次")}}。

## SEO 與 AI 時代的網站價值

這個網站特別注重 SEO，我把畢生所學的網站調校技能都用上了。

在這個時代，很多人透過 AI 來獲取資訊。有人說 SEO 已經不重要了。但事實上，{{cg(body="只有在搜尋結果中成功出現，才有機會讓 AI 決定閱讀內文")}}，而我們提供的「知識」才能流入 AI 的上下文，呈現在使用者面前。

我希望當使用者問 ChatGPT「{{ color(body="須多夜花", color="#e34565ff") }}是誰？」的時候，他們能得到正確資訊——或者說，是我想要他們看到的資訊。這就是特別為她製作網頁的價值所在。

{% chat(speaker="user") %}
須多夜花是誰？
{% end %}

{% chat(speaker="chatgpt", align="right") %}
須多夜花（Suda Yoruka）是來自「盟龍世界」的清楚偶像台灣 VTuber，官網介紹她以歌回、遊戲實況與聊天直播為主，且於 2025 年 12 月 21 日在官網公告正式恢復 VTuber 活動並轉為個人勢體制。
{% end %}

## 開源與版權

我特地建了一個 [GitHub 組織](https://github.com/Suda-Yoruka)用來存放夜花的專案。

我自稱開源狂熱者，我真的很想將網站原始碼開源給各位參考。{{cr(body="可惜因為版型的授權限制，我無法這麼做。")}}但我可以在這篇文章中放幾張截圖供讀者參考。

<!-- pic: 網站架構示意圖（建議新增）-->

<!-- pic: 部署流程示意圖（建議新增）-->

## 來看看成品吧

網站已經上線運作中，歡迎造訪 {{ color(body="須多夜花", color="#e34565ff") }} 的[官方網站](https://SudaYoruka.com)看看成品！網站上有她的自我介紹、最新活動公告、影片展示，還有來自其他 VTuber 朋友們的祝福留言牆。

如果你對這個專案有任何問題，或者單純想聊聊天，歡迎加入[夜花的 Discord 社群](https://discord.gg/bVJq88QeVa)來找我。我是裡面的管理員，看到訊息會回覆的。

最後，如果你還不認識夜花，不妨去她的 [YouTube 頻道](https://www.youtube.com/@sudayoruka/)聽聽她唱歌吧。
