+++
title = "[Koikatu / Koikatsu Sunshine] 開門查水表！ (FBI Open Up!)"
description = "此插件可依照原始角色，將她們轉變為小蘿莉。"
date = 2020-01-05T10:22:00.015Z
updated = 2022-12-16T13:01:59.379Z
draft = false
aliases = [
  "/2020/01/koikatu-fbi-open-up.html",
  "/2020/01/koikatubepinex-v5x-fbi-open-up.html"
]

[taxonomies]
tags = [ "Koikatu", "Koikatsu Sunshine" ]
licenses = [ "GFDL 1.3" ]

[extra]
banner = "../preview/demo9.png"
iscn = "iscn://likecoin-chain/QEHgpUPrvtcteiTiKfw0x5XpaECGGnHYzCjbgwsy9uw/2"
hot = true
+++
{{ video(url="/Koikatu/preview/demo9.mp4", alt="FBI Open Up video", controls=true, loop=true, autoplay=true, muted=true) }}

* 此插件可依照原始角色，將她們轉變為小蘿莉
* **支援替換模板角色**，例如:
  * 若將模板自訂為巨乳姊姊，就可以轉變功能為替換成大姊姊
  * 將模板訂為三頭身 (Chibi) 並開啟 ABMX 設定，這就能成為三頭身變化功能
* 可在 Main Game、Studio、Maker 和 Free H 內執行
* 我置入了幾張過場圖片和動畫，作為娛樂效果

<!-- more -->

{{ youtube(id="0-286KH6ZQA") }}

## 使用說明

[![](/Koikatu/preview/demo9-1.png)](/Koikatu/preview/demo9-1.png)

1. 功能觸發圖標為一{{cr(body="紅色書包")}}，位置紀錄如下
   * Studio: 位在「Add」→「女角色」
   * Maker: 位在正上方之「衣裝切換欄」的右側
   * Free H: 位在左上角的「服裝」子選單之中
   * MainGame 主遊戲: 位在滑鼠中鍵暫停時的右排按鍵最下方
2. **短按**一次**啟動 \ 關閉**功能，並**替換 \ 倒回**場景內的**所有角色**  
   (Studio 內**長按**可啟動 \ 關閉功能但**不變更現有角色**)
3. 若功能開啟，Studio 和 Maker 載入人物時人物會自動被替換  
   這包含 Studio 的 Scene 存檔載入也會套用
4. 計算邏輯為: **新數據 = 原始數據 + ((模板數據 - 原始數據) \* Change Rate)**  
   此運算會套用至身體和臉部的所有數值  
   (大致上等於 Maker 中身體 \ 臉部頁籤最下面的所有陳列數值)
5. ABMX 功能{{cr(body="沒有計算")}}，只能全部覆蓋，功能需要於設定中開啟。  
   **此功能設計用來三頭身化**

## Configure 設定說明

* **Change rate: 原始人物向模板人物改變的比例**  
  數值為 0 (不改變)\~1 (全改變)。
* Enable: 是否啟用插件。  
  這同時反映在遊戲中的紅色書包圖標之明暗狀態。  
  Studio 和 Maker 如果在啟用狀態載入新人物，新人物將會直接被替換。
* **Effect on ABMX: 啟用 ABMX 覆寫功能**  
  若啟用會把模板的 ABMX 全覆蓋至對象，且會禁用回退功能。
* Sample chara: 模板人物路徑。  
  留空白即可使用預設人物。  
  可傳入絕對路徑或相對路徑，如「UserData/chara/female/\*.png」。
* Video path: FBI.mp4 影片的路徑  
  預設路徑為「UserData/audio/FBI.mp4」
* Video volume: 影片音量  
  預設為 0.04，請視喜好自行調整。

## ※注意事項※

1. 雖然目前有作主遊戲之功能，但並未完整測試，且沒有計畫再完善它  
   **主遊戲功能請單純作為附加功能視之**
2. Free H 內沒有過場插圖；主遊戲沒有人物加亮動畫
3. 模板角色可在 Configuration Manager 設定內更改，製作要點請見後述
4. {{cr(body="如果不想要 FBI 影片，請移除 mp4 檔案即可")}}

## 模板角色製作指南

* 請製作出一個**你心目中 100% 的角色存檔**，例如: 100% 的蘿莉、100% 的御姊
* 對於 ABMX 數據，開啟功能後模板的 ABMX 是**完全覆蓋**對象人物，不受 Change Rate 影響  
  ABMX 請只用在特殊身形，**例如三頭身化**  
  {{cr(body="製做普通的模板時請不要使用 ABMX")}}
* 建議扒光她所有的衣服和飾品，以降低存檔體積和降低電腦負擔

## 需求依賴

### Koikatu

* コイカツ！ ダークネス (Koikatu! Darkness)  
  這不相容於 Steam Koikatsu Party
* **BepInEx v5.1 (不支援 v5.0.X)**
* BepisPlugins r15
* Koikatu ABMX v4.3

### Koikatsu Sunshine

* **BepInEx v5.4.15**
* BepisPlugins r16.8.1
* Koikatu ABMX v4.4.5

## 安裝方式

* 將 \*.dll 放至「BepInEx/plugins/jim60105」資料夾之下
* KK: 將 \*.mp4 影片放至「UserData/audio」資料夾之下
* KKS: 將 \*.mp4 影片放至「UserData/Custom」資料夾之下

## 下載位置

<https://cloud.maki0419.com/s/Y8RGD4k9HnaCRnE>
