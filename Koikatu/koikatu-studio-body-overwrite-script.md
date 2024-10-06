+++
title = "[Koikatu][BepInEx v5.X] Studio 角色覆寫腳本 (Studio Body Overwrite Script)"
description = ""
date = 2020-05-16T05:48:00.013Z
updated = 2021-10-31T11:12:17.939Z
draft = false
aliases = ["/2020/05/koikatu-studio-body-overwrite-script.html"]

[taxonomies]
tags = ["Koikatu", "C#"]
+++
![](https://img.maki0419.com/blog/preview/demo15.jpg) 
  
  
這個不是Plugin而是Script，**請用[ScriptLoader](https://github.com/denikson/BepInEx.ScriptLoader)載入執行** 

請手動編輯.cs檔，將內容修改為你要覆寫的數值

* 數值等於Maker中的數字除以100 (即遊戲內數值89 = 0.89f)
* Color為Color(r, g, b, a)，請參閱[UnityDoc](https://docs.unity3d.com/ScriptReference/Color-ctor.html)
* 不需覆寫的項目請在開頭加上「//」做行註解

**Studio中以O鍵觸發** 
  
**衝突: Koikatu PushUp** 因其內部自己存了一份胸部數據，並會在儲存和切換時回寫

#### 需求依賴

* コイカツ！ ダークネス (Koikatu! Darkness)  
這不相容於Steam Koikatsu Party
* **BepInEx v5.0.1**
* BepisPlugins r13.1.1

#### 安裝方式

1. 安裝[ScriptLoader](https://github.com/denikson/BepInEx.ScriptLoader)
2. 將\*.cs置於「Koikatu/scripts」下

#### 下載位置

<https://cloud.maki0419.com/s/sxp596jEzfapnzL>