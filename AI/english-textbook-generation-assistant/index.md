+++
title = "從一小時到一分鐘：重塑父親的 AI 英文教材產製工作流 —— 「AI 英文教材產生助手」"
description = "見證 AI 工作流如何將一小時的英文教材製作時間縮短為一鍵生成！本文完整揭露筆者如何在 Coze 平台上實作自動化教材產製工作流，並分享實際應用成效與技術實現細節。"
date = "2025-01-27T23:14:42.442Z"
updated = "2026-01-20T14:22:50.775Z"

[taxonomies]
licenses = [ "GFDL 1.3" ]
tags = [ "AI", "Coze" ]

[extra]
banner = "preview.png"
withAI = "本篇文章透過「[筆韻智匠 Quill Sage🖋️✨](https://www.coze.com/s/Zs8k6GASu/)」和 Claude 3.5 Sonnet 聯合創作"
iscn = "iscn://likecoin-chain/rz7P_N3CtgK0Cj1FzHqIcl1nwhxE4cabkLAArLgFplc/1"
archive = "由於 [Coze 下架 Asynchronous operation 功能](https://www.coze.com/open/docs/guides/announcement_decommissioning)，本助手已無法正常運作。"

  [extra.preview]
  withAI = true
  url = "https://civitai.com/images/54269254"

  [extra.comments]
  id = "agzbfqs4x37i00ty"
+++
## ChatGPT 🤖 幫我產生一份英文教材

在探索 AI 應用的過程中，前段時間我發現了一個值得優化的日常工作流程。

> 老爸的英文教材
>
> ---
> {{ youtube(id="RveajhHL7Q4") }}

我爸每天會使用 **ChatGPT** 為我兩個就讀建中的弟弟製作英文教材。他每天都會發一篇在我們家的 Messenger 群組裡，他已經做了一整個系列。

深入了解後，我發現我爸每天會花費{{ cr(body="約 50 分鐘")}}製作這些教材，大部分時間用於與 ChatGPT 互動，以產生所需的文章內容並確保格式一致。

50 分鐘！  
這是一個相當可觀的時間，尤其是在他每天都重複進行的情況之下 😮

作為 AI 應用開發者，我認為這是一個很適合通過 AI workflow 來自動化的需求案例。因此，我開發了「英文教材產生助手 (English Textbook Generation Assistant)」 Coze bot ，{{cg(body="目標是將我爸的需求轉成工作流，實現高度自動化。")}}

在接下來的章節中，我會介紹這個 AI 助手的核心功能、Workflow 設計和技術選擇，以及實際應用效果。這個案例展示了日常任務的自動化，也為 AI 在教育領域的應用提供了一個範例。我希望能展示 AI 在提升日常工作效率方面的潛力，並激發更多人思考如何在自己的領域中應用 AI 技術。

<!-- more -->

## 英文教材產生助手 English textbook generation assistant

> {{ image(url="coze.png", alt="Coze store screenshot") }}
> 英文教材產生助手 - Coze AI Agent
>
> ---
> <https://www.coze.com/s/Zs8U1oerD/>

<aside>

{{ image(url="logo.png", alt="Logo of English textbook generation assistant") }}
</aside>

**它能夠根據給定的單字和概念自動產生英文課文。這篇課文包含了目標單字，並能巧妙地融入指定的主題或概念。** 課文設計以對話的形式呈現，確保內容生動有趣，易於理解和記憶。**另一個主要功能是中文翻譯。** AI 助手會為產生的英文課文提供逐句對照的中文翻譯。這種對照形式有助於學習者理解英文句子的結構和意義。

這個工具的一大特點是其靈活性。**使用者可以根據自己的需求自定義單字列表和要傳達的概念。** 如果使用者沒有特定要求，AI 助手會自動從高中 7000 單隨機選取 20 個單字，並確保單字的難度分佈均衡。教師可以根據需求產生不同主題的課文，以滿足不同的教學需求。

除了課文，**AI 助手還會產生單字解釋和例句**，幫助學習者更好地理解每個單字的意思。

最終成品的形式也值得一提。**除了文本形式的教材外，AI 助手還能產生 TTS (文字轉語音) 音檔。** 這個音檔能夠幫助學習者提高聽力技能，文本和音檔的結合為學習者提供了全方位的學習體驗。

AI 英文教材產生助手通過自動化和個性化，提高了英語教材的製作效率。它節省了教育工作者的時間，也為學習者提供了高品質、多樣化的學習資源。在下一章中，我將深入探討這個 AI 助手的技術實現過程，讓大家了解它是如何運作的。

## 揭秘 AI 教材生成的六步曲

> 所有的 LLM 提示詞都已經公開在我的 [GitHub](https://github.com/jim60105/prompt/tree/master/english-textbook-generation-assistant)，歡迎大家參考。

在開發 AI 英文教材產生助手時，我設計了一個六步驟的工作流程，以確保能自動化的產生高品質的教材。

### 輸入處理

{{ image(url="1.png", alt="輸入處理") }}

首先，系統接收使用者的輸入。使用者可以提供特定的單字列表和文章主題。如果使用者沒有提供，系統會自動從高中 7000 單字庫中按照難易度平均選取 20 個單字，同時從數十個優良主題中隨機抽選一個。

值得一提的是我專門開了一個 Google Sheet 做為資料庫讓我爸維護這些優良主題，並以 Google Apps Script 寫了一個 GET api 讓 workflow 呼叫。Google Sheet 的優勢在於協作的簡易性，使得非技術人員也能輕鬆編輯。

> 延伸閱讀
>
> ---
> [以 Google 試算表作為簡易資料庫 (上) —— 資料庫的建立及寫入](@/Database/google-spreadsheets-as-database/index.md)

### 單字解釋

{{ image(url="2.png", alt="單字解釋") }}

在這一步驟中，我選擇使用 GPT-4o mini 模型。GPT-4o mini 在處理直觀的任務上表現出色，能夠快速產生正確的單字解釋和例句。

### 英文課文

{{ image(url="3.png", alt="英文課文") }}

這是整個流程中最關鍵的步驟，我選擇了 Claude 3.5 Sonnet 模型。Claude 在文章產生跟脈絡理解方面表現優異，能夠根據給定的單字和主題創作出連貫、有趣的課文。它的創意寫作能力使得產生的內容更加生動有趣。

由於 LLM 的 JSON 回應並不太穩定，我做了三次 retry 機制以提升成功機率。

### 中文翻譯

{{ image(url="4.png", alt="中文翻譯") }}

課文產生後，我再次使用 GPT-4o mini 模型進行翻譯。在每個不同的任務使用不同的 AI 模型，充分發揮各個模型的優勢，以獲得最佳的性價比。

### 內容整合

{{ image(url="5.png", alt="內容整合") }}

最後，將前面產生的所有內容 (單字解釋、英文課文、中文翻譯) 整合到一個固定格式的文字檔中，讓每次產生的教材格式統一。

### TTS 音檔

{{ image(url="6.png", alt="TTS 音檔") }}

{{ image(url="7.png", alt="TTS 輸出") }}

這是一個附加功能，我使用 TTS (文字轉語音) 技術將英文課文轉換為音檔。我利用了微軟免費提供的 Edge TTS 服務來達成這個任務。

在實作這個流程時遇到了一個小小的地雷，那就是微軟的 Edge TTS 服務有時會發生 Timeout。{{cr(body="Coze 的文件明寫了 Plugin Timeout 時間是 3 分鐘，但我實際運行時發現在 60 秒就會 Timeout。")}}而這個問題我透過拆分流程來解決。

- 將 TTS 部份獨立出來為另一個指令，這樣使用者可以單獨運行，而不怕在最後發生 Timeout 時會使整個流程失敗。執行 TTS 指令並不會耗用 Coze Credits，因此使用者可以放心的重覆嘗試。
- 再來將整個輸出拆成三段，降低每個音檔的長度以避免 Timeout，同時以平行處理的方式提高效率。
- 當它還是失敗的時候... 提供使用者一個[外部連結](https://huggingface.co/spaces/innoai/Edge-TTS-Text-to-Speech)，讓他們自行產生音檔 😅

{{ image(url="edge_tts.png", alt="Edge TTS") }}

## 成果展示

以下是 AI 英文教材產生助手生成的一份教材：

<audio controls preload="auto" style="width: 100%">
  <source src="audio.mp3" type="audio/mpeg">
  Your browser does not support the audio element.
</audio>

<details>
<summary>點擊展開教材全文</summary>

以下是簡短的 AI 生成的英文教材，介紹 20 個高中 7000 單中的單字的用法

---

Sarah, the newly appointed director of the Civil Service Reform Commission, stood at the podium, her voice resonating through the dome-shaped auditorium. 'We aim for zero tolerance on corruption,' she declared, her tone tinged with controlled anger. 'It's time we demonstrate our commitment to a clean government.' The audience, a mix of seasoned bureaucrats and fresh recruits, listened intently. Sarah continued, 'Our productivity has spiraled downwards, and we must address this with significant changes.' She emphasized, 'The path to a transparent civil service isn't just about rules. It's about creating a system where gaining a position is challenging and fair, ensuring basic economic security for officials, and maintaining high standards of moral education for all citizens.'

After the speech, Sarah retreated to her office, its walls bare save for a single framed quotation: 'Integrity is the crown of public service.' She sighed, pouring over estimate reports. A knock at the door interrupted her concentration. 'Come in,' she called. Tom, her assistant, entered, carrying a stack of files. 'Ma'am, the foreign consultants have arrived. They're waiting in the conference room with their reservations about the new system.' Sarah nodded, 'I'll be right there. Let's hope they don't howl too loudly at our proposals.' As she stood, she noticed a small potted plant on her desk, a gift from the botany department, reminding her of the growth she hoped to nurture in the civil service.

In the conference room, tensions were high. Dr. Lena, a botany professor turned government efficiency expert, was gesticulating wildly. 'You can't just uproot the entire system!' she exclaimed. 'It's reckless!' Sarah remained calm, 'Dr. Lena, we deem these changes necessary. The old system is a mere skeleton of what it should be.' She paused, then added with a smile, 'Perhaps we need to approach this like you would tend to a delicate plant - with care, but also with the courage to prune where needed.' Sarah continued, 'We're implementing stricter qualification exams for civil servants. This will ensure that positions are earned through merit, not connections. Additionally, we're reviewing salary structures to provide economic security, reducing the temptation for corruption.'

As the meeting concluded, Sarah felt a headache brewing. She longed for a cold beer to take the edge off, but knew better than to indulge during work hours. Instead, she focused on the task ahead: to decorate the civil service with integrity and efficiency. It would be a challenging journey, but Sarah was determined to see it through. She glanced at the spiral staircase leading to the upper floors of the building, a metaphor for the upward trajectory she envisioned for the department. 'We'll get there,' she murmured to herself, 'step by step, reform by reform.' As she left the room, she made a mental note to coordinate with the Education Ministry on enhancing moral education in schools. 'After all,' she thought, 'a truly incorruptible civil service begins with the values we instill in our citizens from a young age.'

---

中文解釋

---

Sarah, the newly appointed director of the Civil Service Reform Commission, stood at the podium, her voice resonating through the dome-shaped auditorium.  
新任公務員改革委員會主任莎拉站在講臺上，她的聲音在圓頂形的禮堂中迴盪。

'We aim for zero tolerance on corruption,' she declared, her tone tinged with controlled anger.  
「我們對貪腐的忍受度為零，」她宣佈，語氣中流露出克制的憤怒。

'It's time we demonstrate our commitment to a clean government.'  
「是時候表現出我們對清廉政府的承諾了。」

The audience, a mix of seasoned bureaucrats and fresh recruits, listened intently.  
觀眾由資深官僚和新進人員組成，大家聚精會神地傾聽。

Sarah continued, 'Our productivity has spiraled downwards, and we must address this with significant changes.'  
莎拉繼續說：「我們的生產力已經急劇下滑，我們必須以重大變革來解決這個問題。」

She emphasized, 'The path to a transparent civil service isn't just about rules.  
她強調：「通往透明公務的道路不僅僅是制定規則。

It's about creating a system where gaining a position is challenging and fair, ensuring basic economic security for officials, and maintaining high standards of moral education for all citizens.'  
而是創造一個能讓獲得職位變得艱難而公平的系統，確保官員的基本經濟安全，並為所有公民保持高標準的道德教育。」

After the speech, Sarah retreated to her office, its walls bare save for a single framed quotation: 'Integrity is the crown of public service.'  
演講結束後，莎拉退回到她的辦公室，牆上空曠，只掛著一句框架裱裱的名言：「誠信是公共服務的皇冠。」

She sighed, pouring over estimate reports.  
她嘆了口氣，仔細查看預算報告。

A knock at the door interrupted her concentration.  
門口一陣敲門聲打斷了她的專注。

'Come in,' she called.  
「請進，」她叫了一聲。

Tom, her assistant, entered, carrying a stack of files.  
她的助手湯姆進來，手裡拿著一疊文件。

'Ma'am, the foreign consultants have arrived.  
「女士，外國顧問已經到了。

They're waiting in the conference room with their reservations about the new system.'  
他們在會議室等待，對新系統有些顧慮。」

Sarah nodded, 'I'll be right there.  
莎拉點點頭說：「我馬上就來。

Let's hope they don't howl too loudly at our proposals.'  
希望他們對我們的提議不要大喊大叫。」

As she stood, she noticed a small potted plant on her desk, a gift from the botany department, reminding her of the growth she hoped to nurture in the civil service.  
當她起身時，注意到桌上有一盆小植物，是植物學系的贈禮，提醒她希望在公務員體系中培養的成長。

In the conference room, tensions were high.  
在會議室裡，緊張氣氛濃厚。

Dr. Lena, a botany professor turned government efficiency expert, was gesticulating wildly.  
莉娜博士，一位從植物學教授轉行為政府效率專家的專家，正在激烈地比劃。

'You can't just uproot the entire system!' she exclaimed.  
「你不能僅僅把整個系統連根拔起！」她驚叫道。

'It's reckless!'  
「這太魯莽了！」

Sarah remained calm, 'Dr. Lena, we deem these changes necessary.  
莎拉保持冷靜，「莉娜博士，我們認為這些變革是必要的。

The old system is a mere skeleton of what it should be.'  
舊系統只是一個應有體系的空殼而已。」

She paused, then added with a smile, 'Perhaps we need to approach this like you would tend to a delicate plant - with care, but also with the courage to prune where needed.'  
她停頓了一下，然後微笑著補充道：「也許我們需要像照顧脆弱的植物那樣對待這個問題——小心翼翼，但也必須勇於修剪。」

Sarah continued, 'We're implementing stricter qualification exams for civil servants.  
莎拉接著說：「我們將對公務員實施更嚴格的考試資格。

This will ensure that positions are earned through merit, not connections.  
這將確保職位是通過優秀的表現獲得的，而非靠關係。

Additionally, we're reviewing salary structures to provide economic security, reducing the temptation for corruption.'  
此外，我們正在審查薪資結構，以提供經濟安全，減少貪腐的誘惑。」

As the meeting concluded, Sarah felt a headache brewing.  
會議結束時，莎拉感到一陣頭痛。

She longed for a cold beer to take the edge off, but knew better than to indulge during work hours.  
她渴望一杯冰啤酒來放鬆心情，但知道在工作時間不宜如此縱容自己。

Instead, she focused on the task ahead: to decorate the civil service with integrity and efficiency.  
相反地，她專注於接下來的任務：用誠信和效率來裝飾公務體系。

It would be a challenging journey, but Sarah was determined to see it through.  
這將是一段艱難的旅程，但莎拉下定決心要將其完成。

She glanced at the spiral staircase leading to the upper floors of the building, a metaphor for the upward trajectory she envisioned for the department.  
她瞥了一眼通往大樓上層的螺旋樓梯，這象徵著她對部門未來上升軌道的想像。

'We'll get there,' she murmured to herself, 'step by step, reform by reform.'  
「我們會做到的，」她自言自語，「一步一步，改革一項。」

As she left the room, she made a mental note to coordinate with the Education Ministry on enhancing moral education in schools.  
當她離開會議室時，腦中記下要與教育部協調加強學校的道德教育。

'After all,' she thought, 'a truly incorruptible civil service begins with the values we instill in our citizens from a young age.'  
「畢竟，」她想，「一個真正廉潔的公務員隊伍始於我們從小傳授給公民的價值觀。」

---

單字用法

---

[Anger]  
A strong feeling of displeasure or rage.

[Anger]  
憤怒：一種強烈的不悅或憤怒的感覺。

[例句]  
His anger was evident when he received the unfair news.  
當他收到不公平的消息時，他的憤怒顯而易見。

[Foreign]  
Relating to or originating in a country or language other than one's own.

[Foreign]  
外國的：指與自己所在國家或語言不同的國家或語言。

[例句]  
She enjoys learning about foreign cultures and traditions.  
她喜歡了解外國的文化和傳統。

[Zero]  
The numerical symbol representing no quantity or null value.

[Zero]  
零：代表無數量或空值的數字符號。

[例句]  
The temperature dropped to zero degrees Celsius last night.  
昨晚的氣溫降到了零度攝氏。

[Beer]  
An alcoholic beverage made from fermented grains, typically flavored with hops.

[Beer]  
啤酒：由發酵穀物製成的含酒精飲料，通常用啤酒花調味。

[例句]  
He enjoys having a cold beer after work.  
他喜歡在下班後喝一杯冰啤酒。

[Director]  
A person responsible for the overall management and operation of a film, play, or organization.

[Director]  
導演：負責電影、劇本或組織的整體管理和運作的人。

[例句]  
The director of the film won an award for his outstanding work.  
該部電影的導演因其出色的工作而獲獎。

[Decorate]  
To make something look more attractive by adding items or features.

[Decorate]  
裝飾：通過增添物品或特徵使某物看起來更具吸引力。

[例句]  
They plan to decorate their home for the holidays.  
他們計劃為假期裝飾他們的家。

[Crown]  
A circular ornamental headdress worn by a monarch as a symbol of authority.

[Crown]  
皇冠：由統治者佩戴的圓形裝飾性頭飾，象徵著權威。

[例句]  
The queen wore a beautiful crown during the ceremony.  
女王在儀式上佩戴著一頂漂亮的皇冠。

[Significant]  
Having meaning or importance, especially in a particular context.

[Significant]  
重要的：在特定上下文中具有意義或重要性。

[例句]  
The discovery of the new species was a significant contribution to science.  
發現新物種是對科學的重要貢獻。

[Bare]  
Without covering or clothing; exposed.

[Bare]  
光禿的；裸體的：沒有覆蓋或衣物；暴露的。

[例句]  
The tree stood bare after losing all its leaves in autumn.  
這棵樹在秋天落光所有葉子後光禿禿的屹立著。

[Demonstrate]  
To show or prove something clearly through evidence or examples.

[Demonstrate]  
演示：通過證據或例子清楚地展示或證明某事。

[例句]  
The teacher used experiments to demonstrate the principles of science.  
老師用實驗來演示科學原理。

[Estimate]  
An approximate calculation or judgement of the value, number, quantity, or extent of something.

[Estimate]  
估計：對某事物的價值、數量、數目或程度的近似計算或判斷。

[例句]  
We need to estimate the costs before starting the project.  
在開始項目之前，我們需要估算成本。

[Reservation]  
An arrangement made in advance to secure accommodations, a seat, or a service.

[Reservation]  
預訂：事先安排以確保住宿、座位或服務。

[例句]  
I made a reservation at the restaurant for dinner tonight.  
我在餐廳預訂了今晚的晚餐。

[Botany]  
The scientific study of plants, including their structure, properties, and biochemical processes.

[Botany]  
植物學：對植物的科學研究，包括它們的結構、屬性和生化過程。

[例句]  
She is studying botany to understand more about plant life.  
她正在學習植物學，以更多地了解植物生活。

[Howl]  
To make a long, loud, mournful sound, often associated with wolves or the wind.

[Howl]  
嚎叫：發出長的、響亮的、悲傷的聲音，通常與狼或風有關。

[例句]  
The wolves began to howl as the moon rose in the sky.  
當月亮升起時，狼群開始嚎叫。

[Skeleton]  
The framework of bones in a body; also refers to a model or representation of such a framework.

[Skeleton]  
骨架：身體中的骨框架；也指這種框架的模型或表示。

[例句]  
The scientist studied the skeleton to learn about human anatomy.  
科學家研究這具骨架以了解人體解剖。

[Reckless]  
Showing a lack of care or concern for dangerous or risky situations; careless.

[Reckless]  
魯莽的：對危險或風險情況表現出缺乏關心或照顧；粗心的。

[例句]  
Driving at high speeds in the rain is a reckless behavior.  
在雨中高速駕駛是一種魯莽的行為。

[Deem]  
To regard or consider in a specified way.

[Deem]  
認為：以特定的方式看待或考慮。

[例句]  
She deemed it necessary to apologize for her mistake.  
她認為道歉是有必要的，以便彌補她的錯誤。

[Dome]  
A rounded vault forming the roof of a building or structure.

[Dome]  
圓頂：構成建築物或結構屋頂的圓形拱頂。

[例句]  
The church features a beautiful dome above the altar.  
教堂的祭壇上方有一個美麗的圓頂。

[Productivity]  
The state or quality of producing something, especially in large amounts.

[Productivity]  
生產力：生產某物的狀態或品質，特別是大量生產。

[例句]  
Improving workplace productivity can lead to greater profits.  
提高工作場所的生產力可以帶來更高的利潤。

[Spiral]  
A curve that winds around a fixed center point while moving away from or toward it.

[Spiral]  
螺旋：一種圍繞固定中心點並向外或向內移動的曲線。

[例句]  
The staircase had a beautiful spiral design that was very elegant.  
這座樓梯有著優雅的美麗螺旋設計。
</details>

## AI 助手的實戰成效

在實際應用中，AI 英文教材生成助手展現了令人滿意的效果。我爸使用這個工具後，教材製作的時間從原本的 50 分鐘大幅縮短。現在他能將更多的精力放在教材設計跟內容創作上，而不必花費大量時間與 ChatGPT 互動。

想想看，日常生活中還有哪些方面可以透過 AI 來自動化呢？**關鍵在於識別那些重複性高、有規律性，且適合 AI 參與其中的任務**，這些通常是以 AI 自動化的好選擇。

如果你也有學習英文的需求，不妨親自試試這個工具。簡單的按幾下按鈕，幾分鐘後，你就能獲得一份包含單字解釋、英文課文、中文翻譯和音檔的完整教材。這個工具非常適合教師、家長、學生使用。

技術創新確實能顯著提升我們的生活品質。AI 英文教材生成助手只是眾多可能性中的一個例子。它展示了我如何利用 AI 工作流來解決實際問題，提高效率並創造價值。鼓勵大家以創新的視角看待周圍的事物，思考如何將 AI 技術應用到自己的生活中。
