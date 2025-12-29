+++
title = "打造 VTuber 須多夜花官方網站：Zola SSG 與 AI 協作開發實錄"
description = "分享使用 Zola 靜態網站生成器打造 VTuber 須多夜花官方網站的開發心得。採用 SDD 規格驅動開發，讓 AI 自動實作 Tera 模版遷移、Sass 樣式整合，並透過 GitHub CI/CD 部署至 Cloudflare Pages。"
date = "2025-12-29T09:27:20.808Z"
updated = "2025-12-29T17:12:44.963Z"

[taxonomies]
tags = [ "AI", "Cloudflare", "Livestream", "VTuber", "SudaYoruka" ]
licenses = [ "All Rights Reserved" ]

[extra]
withAI = "本文使用 GitHub Copilot Claude Opus 4.5 協助撰寫"
banner = "preview.png"
+++

> {{ color(body="須多夜花", color="#e34565ff") }}官方網站  
> <https://SudaYoruka.com>

這篇文章記錄了我為 VTuber [{{ color(body="須多夜花", color="#e34565ff") }}](https://www.youtube.com/@sudayoruka) 打造官方網站的開發歷程。我是琳，是須多夜花的 IP 營運負責人，同時也是這個官方網站的獨立開發者。

<!-- more -->

## 為什麼要做官方網站

製作官網其實是我個人的想法。在安排夜花的復出計劃時，原本並不包含這個專案。身為{{ ch(body="前") }}網頁工程師，我本來就很喜歡做網站。擁有個人網站的 VTuber 並不多，{{cr(body="因為開發成本高")}}，除非像我這樣自己動手，才有可能實現。

我的專業是網頁後端及系統整合，{{cr(body="這種以前端和美術為主的網站並非我的強項")}}。原本計劃委託網頁公司製作，但詢價後因預算不足，最終決定自己來。也因此，我們得到了一個「還過得去」的成品，它更像是我的個人專案。

## 技術選型：為什麼選擇 Zola

我採用了一個尋常網頁公司絕對不會使用的框架：[**Zola**](https://www.getzola.org/)。Zola 使用相對小眾的 [Tera](https://keats.github.io/tera/docs/) 模版語言搭配 [Sass](https://sass-lang.com/) 建構靜態網頁。它是 SSG （Static Site Generator） 靜態網頁生成系統。Zola 的設計適合製作一篇一篇內容的網頁，這正好符合我們官網的需求。比起「官方網站」，我的需求更接近「官方部落格」的型態。

還有就是，我喜歡 Zola 😆

<figure>
{{ image(url="9.png", alt="Markdown 文章", full=true) }}
<figcaption>Markdown 文章</figcaption>
</figure>

更重要的是，{{cg(body="Zola 允許使用 Markdown 撰寫內容")}}。

這意味著未來的編輯者 —— 大家最愛的{{ color(body="夜花花", color="#e34565ff") }} —— 不需要懂 HTML 和 CSS 就能發佈文章。

{% chat(speaker="jim") %}
妳什麼時候要來學怎麼發文
{% end %}

{% chat(speaker="yoruka") %}
好麻煩喔，我發 twitter 你幫我發一發就行了吧
{% end %}

當然好，全都聽妳的。

## 版型遷移：從 Panini 到 Tera

版型我在 themeforest 上挑了很久，最後選定 [FriendKit](https://cssninja.io/product/friendkit) 這個社群網站版型，打造出像社群平台的官方網站。這個版型其實很豐富，一個標準社群網站該有的頁面它全都有設計，但我們只用到了其中幾頁。

原始的 HTML 版型使用 [Panini](https://github.com/foundation/panini) 模版系統，說實話這是我第一次知道它。這並不是我要的，在實作時我把它改成了 Tera 模版。至於 Sass 基本上是直接遷移過來，然後砍掉多餘的部份。

這是很大的工作量，不過沒關係，反正不是我要做 😉

<figure>
{{ image(url="1.png", alt="網站首頁截圖", full=true) }}
<figcaption>網站首頁截圖</figcaption>
</figure>

## 讓 AI 來寫程式碼

這個網站是使用我流的 **SDD（Specification-Driven Development）規格驅動開發**，{{cg(body="整個開發流程外包給 AI 來執行")}}。

我在本地準備好了開發需要的所有知識脈絡，包含 Tera 和 Zola 的撰寫指引。我準備了 `AGENTS.md` 作為開發憲章，然後使用 GitHub 的 Issue 和 PR 系統做任務管理。

具體流程如下：我讓 VSCode 的 GitHub Copilot（Claude 模型）規劃完整的工作，並使用 MCP 在 GitHub 開 Issue。每個 Issue 的內容都是一個詳盡的開發計劃，我告訴它要寫得詳細到連菜逼八也能照著實作。接著使用 GitHub Coding Agent 依序實作每一個 Issue。我負責審閱 PR。

<figure>
{{ image(url="7.png", alt="AGENTS.md 截圖", full=true) }}
{{ image(url="8.png", alt="AGENTS.md 截圖", full=true) }}
<figcaption>AGENTS.md 截圖</figcaption>
</figure>

<figure>
{{ image(url="2.png", alt="Issue 清單截圖", full=true) }}
{{ image(url="3.png", alt="Issue 清單截圖", full=true) }}
<figcaption>Issue 清單截圖</figcaption>
</figure>

<figure>
{{ image(url="4.png", alt="其中一個詳盡的 Issue 截圖", full=true) }}
<figcaption>其中一個詳盡的 Issue 截圖</figcaption>
</figure>

<figure>
{{ image(url="6.png", alt="PR 成果截圖", full=true) }}
<figcaption>PR 成果截圖</figcaption>
</figure>

最後的成品中，{{cg(body="由我手寫的程式碼其實非常少")}}。

{{cr(body="Zola 和 Tera 並不是常見的技術堆疊，一般通用 AI 不太能成功撰寫相關程式碼。")}}這個專案能夠成功，仰賴的是我對這些工具的熟悉程度，我在開發過程中提供了足夠的知識脈絡，才使得 AI 有辦法完成任務。我並不推薦其他人貿然走這條路。

當你問我說：

{% chat(speaker="user") %}
哇，這個網站是你自己做的嗎？
{% end %}

{% chat(speaker="jim") %}
其實我是用 AI 做的啦www
{% end %}

的時候，我指的是像上面描述的那樣。

事實上，別說外行人，我想很多工程師都不一定能駕馭這種開發流程。SDD 和 Context Engineering 脈絡工程可算是 2025 年最前沿的開發方法論。用這個方法，我只用了三週的業餘時間就完成了九成的程式碼開發。

若你對 SDD 有興趣，可以參考我的另一篇文章，這裡有更詳細的講解： [規格驅動開發 (Spec-Driven Development) 與 AI 協作全流程實戰](https://聆.tw/AI/sdd-ai-copilot-codex-devops-workflow/)

## 讓 AI 替 VTuber 寫文章

在網站差不多完成後，我寫了一個「用 {{ color(body="須多夜花", color="#e34565ff") }} 人格來寫作」的 GitHub Copilot Instruction 指示詞。這個 prompt 主要著重在產出正確的 Markdown 格式，再加上一點點的角色扮演元素。你可以想像是[設計角色扮演 LLM Prompt 的實務心得](@/AI/design-roleplay-llm-prompts/index.md)與部落格寫作指引的結合體，但不包含夜花本人的個性參考素材。

像是官網上的這篇[「我回來了！這次，我不會再消失」](https://sudayoruka.com/posts/2025-12-21-intro/)介紹文，初始文案是夜花自己寫的，我再用上述的指示詞擴寫為最終文字。

偷偷爆料，最後那一句{{ color(body="「請多指教，我所愛的人們」", color="#e34565ff") }}是夜花寫給我的，去調侃她她會害羞喔 🤫

## 部署流程：GitHub Actions 與 Cloudflare Pages

這個專案使用 GitHub Workflows 進行 CI/CD，部署至 Cloudflare Pages。我們的部署流程比一般的 Zola 網站複雜一些。

除了標準的 Zola 建構之外，還有兩個額外的 Python 指令碼需要執行：一個負責取得 YouTube 影片清單，另一個負責取得[聲音按鈕](https://sound-buttons.click/yoruka)清單。這些指令碼會處理即時資料之後產生我需要的 JSON 檔案。

由於這是靜態網頁，為了讓影片清單保持更新，{{cg(body="CD Pipeline 每天都會定期執行一次")}}。

## SEO 與 AI 時代的網站價值

這個網站特別注重 SEO，我把畢生所學的網站調校技能都用上了。

有人認為，在這個時代大家都透過 AI 總結直接獲取資訊，因此 SEO 已經不重要了。但事實上，{{cg(body="只有在搜尋結果中成功出現，才有機會讓 AI 決定閱讀內文")}}，而後我們提供的「知識」才能流入 AI 的脈絡，呈現在使用者面前。

我希望當使用者問 ChatGPT「{{ color(body="須多夜花", color="#e34565ff") }}是誰？」的時候，他們能得到正確資訊——或者說，是我想要他們看到的資訊。這就是特別為她製作網頁的價值所在。

{% chat(speaker="user") %}
須多夜花是誰？
{% end %}

{% chat(speaker="chatgpt", align="right") %}
須多夜花（Suda Yoruka）是來自「盟龍世界」的清楚偶像台灣 VTuber，官網介紹她以歌回、遊戲實況與聊天直播為主，且於 2025 年 12 月 21 日在官網公告正式恢復 VTuber 活動並轉為個人勢體制。
{% end %}

## 開源與版權

我特地建了一個 [GitHub 組織](https://github.com/Suda-Yoruka)用來存放夜花的專案。

我自稱開源狂熱者，我真的很想將網站原始碼開源給各位參考。{{cr(body="可惜因為版型的授權限制，我無法這麼做。")}}相對的，我在這篇文章中放了不少截圖供讀者參考。

<details><summary>網站資料夾結構</summary>

```txt
.
├── AGENTS.md
├── README.md
├── config.toml              # Zola configuration (base_url, taxonomies, Sass, etc.)
├── frontmatter.json         # VS Code Front Matter workspace metadata
├── wrangler.jsonc           # Cloudflare Pages deployment config
├── content/
│   ├── _index.md            # Homepage (timeline style profile)
│   ├── contact.md           # Contact page stub
│   ├── derivative-works.md  # Fan creation guidelines page
│   ├── privacy-policy.md    # Markdown version of the privacy policy
│   ├── privacy-policy-en.md # English version of privacy policy
│   ├── videos.md            # Video showcase page
│   └── posts/
│       ├── _index.md                              # News section listing
│       ├── 2025-12-21-intro.md                    # Intro post
│       ├── 2025-12-21-return-announcement.md      # Return announcement post
│       ├── 2025-12-22-original-song-first-with-you/  # Original song post (with assets)
│       ├── 2025-12-22-weekly-schedule/            # Weekly schedule post (with assets)
│       └── 0000-01-01-demo-*/                     # Demo posts (gallery, video, link types)
├── templates/
│   ├── 404.html             # 404 Error page
│   ├── atom.xml             # Atom feed template
│   ├── base.html            # Shared shell with sidebar layout
│   ├── index.html           # Homepage & profile layout
│   ├── macros.html          # Reusable Tera macros
│   ├── posts-section.html   # News listing section
│   ├── post.html            # Generic long-form page layout
│   ├── videos.html          # Video showcase layout
│   ├── taxonomy_list.html   # Taxonomy listing template
│   ├── taxonomy_single.html # Single taxonomy page template
│   ├── layouts/
│   │   └── sidebar-v1.html  # Base sidebar structure
│   ├── partials/
│   │   ├── footer/          # Footer components (fat-footer.html)
│   │   ├── head/            # Head metadata (head.html, scripts.html, stylesheets.html, etc.)
│   │   ├── lightbox/        # Gallery overlays (custom-lightbox.html)
│   │   ├── modals/          # Modal components (logout-info-modal.html)
│   │   ├── navigation/      # Sidebar + toolbar components
│   │   ├── pageloader/      # Loading overlay (pageloader.html)
│   │   ├── pages/           # Page-specific fragments (index/, posts/, profile/)
│   │   ├── widgets/         # Sidebar widgets (soundbuttons, star-friends, videos, etc.)
│   │   └── strings.html     # Localization strings
│   └── shortcodes/
│       ├── picture.html     # Picture shortcode
│       ├── video.html       # Video embed shortcode
│       └── youtube.html     # YouTube embed shortcode
├── sass/
│   ├── main.scss            # Master Sass entrypoint
│   ├── abstracts/           # Variables & mixins (_fonts, _mixins, _variables)
│   ├── components/          # Buttons, cards, forms, modals, gallery, soundbuttons, etc.
│   ├── layout/              # Shell + panel styles (_footer, _layout)
│   ├── navigation/          # Sidebar / navbar styling (_dropdowns, _sidebar-v1, _toolbar)
│   └── pages/               # Page-level styles (_feed, _profile, _stories, _tags, _videos)
├── static/
│   ├── _headers             # Cloudflare Pages headers config
│   ├── robots.txt           # Robots exclusion file
│   ├── site.webmanifest     # PWA manifest
│   ├── favicon.svg / favicon.ico / apple-touch-icon.png / favicon-96x96.png
│   ├── web-app-manifest-192x192.png / web-app-manifest-512x512.png
│   ├── assets/
│   │   ├── img/             # Images (avatars, covers, icons, tachie, logo, etc.)
│   │   │   ├── avatars/     # User avatars (profile.avif/webp/jpg, guest avatars)
│   │   │   ├── covers/      # Cover images (profile-cover, 404)
│   │   │   ├── icons/       # UI icons (activities, chat, emoji, feed, etc.)
│   │   │   ├── illustrations/ # Illustration assets
│   │   │   ├── logo/        # Brand logos (yoruka_logo.svg, sound-buttons.svg)
│   │   │   ├── shapes/      # Decorative shapes (card-shape-1.svg, etc.)
│   │   │   ├── tachie/      # Standing character images (1.0, 2.0, surprise)
│   │   │   └── vector/      # Vector icons
│   │   └── js/              # JavaScript files
│   │       ├── fuse.js               # Fuse.js search library
│   │       ├── global.js             # Global functions
│   │       ├── lightbox-controller.js # Lightbox component
│   │       ├── main.js               # Main script
│   │       ├── search-fuse.js        # Search implementation
│   │       ├── tracking.js           # Analytics tracking
│   │       ├── components/
│   │       │   ├── gallery-layout.js # Gallery layout controller
│   │       │   └── soundbuttons.js   # Sound Buttons widget
│   │       └── navigation/
│   │           └── sidebar-v1.js     # Sidebar navigation
│   ├── data/
│   │   ├── youtube-videos.json  # Cached YouTube feed (CI/CD generated)
│   │   └── soundbuttons.json    # Cached Sound Buttons data (CI/CD generated)
│   └── .well-known/
│       └── gpc.json             # Global Privacy Control JSON
├── scripts/
│   ├── README.md                # Scripts documentation
│   ├── fetch_youtube_videos.py  # Pulls the latest YouTube metadata for caching
│   └── fetch_soundbuttons.py    # Fetches Sound Buttons data for widget
├── docs/
│   ├── brand-colors.md          # Brand color guidelines
│   ├── visual-style-guide.md    # Visual style and component guidelines
│   ├── modal-system.md          # Modal component usage documentation
│   ├── 須多夜花-設定.md          # Character settings and world view
│   └── content-writing-guides/  # Step-by-step guides for content writers
│       ├── README.md            # Overview and navigation
│       ├── 01-git-basics.md ~ 13-zola-intro.md  # Sequential workflow guides
│       └── 99-faq.md            # Frequently asked questions
├── .devcontainer/           # Development container definition
├── .friendkit/              # Original FriendKit reference (do not touch)
├── .frontmatter/            # Front Matter VS Code extension data
├── .github/                 # Workflows + instructions
├── .vscode/                 # Editor settings
└── themes/                  # Reserved for Zola theme overrides if needed
```

</details>

## 來看看成品吧

網站已經上線運作中，歡迎造訪 [{{ color(body="須多夜花官方網站", color="#e34565ff") }}](https://SudaYoruka.com)看看成品！網站上有她的自我介紹、最新活動公告、影片展示，還有來自其他 VTuber 朋友們的祝福留言牆。

如果你對這個專案有任何問題，或者單純想聊聊天，歡迎加入[夜花的 Discord 社群](https://discord.gg/bVJq88QeVa)來找我。我是裡面的管理員，看到訊息會回覆的。

最後如果你還不認識夜花，不妨去她的 [YouTube 頻道](https://www.youtube.com/@sudayoruka/)聽聽她唱歌吧。
