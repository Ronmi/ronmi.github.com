+++
date = "2015-09-17T09:12:04+08:00"
title = "root Z Ultra 5.1 (368)"
categories = ["sony"]
tags = ["sony", "root"]
+++

之前升 5.0 的時候在 XDA 找到一鍵 root，用下去看到殘體字介面就覺得完了完了完了。開機後果然發現偷裝 apk，刪掉還會自動重裝；手機反應速度也慢到誇張，開個 Fx beta 要 5～6 秒；Free memory 也只剩約 700 MB。當時就有清空手機重刷的想法，只是一直找不到時間備份資料。前幾天 5.1 更新出來，更新上去發現 root 雖然不見了，可是反應時間沒有改善，決定長痛不如短痛。可惜 XDA 沒有直接教 root 368 的教學，只好自己手動處理。

整個清掉重刷之後果然手機反應速度回復正常，free memory 也回到 1.2 GB，不禁讓人感嘆這後門程式的粗製濫造。同志，走後門也是得裝個樣子你說是不？

<!--more-->

## Tools

* [舊版台灣ftf](https://mega.co.nz/#!nEAzHb6A!5aTtcHZtqgA3W7rRADI7To8DwaOxBZtEnOkezQoiITE)
* [Flashtool](http://www.flashtool.net/index.php)
* [PRFCreator](http://forum.xda-developers.com/crossdevice-dev/sony/tool-prfcreator-easily-create-pre-t2859904)
* [XZlockeddualrecovery](http://nut.xperia-files.com/) installer 跟 flashable 兩個版本都要下載
* [SuperSU](https://download.chainfire.eu/696/SuperSU)

## Synopsis

根據 XZlockeddualrecovery 的警告，我推測這個刷法對 Unlocked Bootloader 可能會有無法預期的後果，要用這個方法之前建議還是先 Relock

下載 FTF 可能會佔去數 GB 的空間。

1. 用 flashtool 裡的 iFirm 下載最新版 ftf
2. 用 PRFCreator 產生 flashable zip 並放進手機 SD 卡
3. 用 flashtool 刷舊版 ftf
4. 用 installer 安裝 recovery
5. flash zip、清 delvik 快取、poweroff
6. 用 flashtool 刷新版 ftf (需避開 system, partition 跟 TA)

## Tips

iFirm 用法：點左邊選手機型號，點中間選韌體的區域，最右邊會有可下載的韌體版本，有點小字不是很容易注意到。

PRFCreator 產生的是 flashable zip 不是 ftf。它使用的 recovery zip 要 flashable 版。

Flashtool 附的驅動記得要裝。

flash zip 清快取之後直接從 recovery 的主選單關機。

刷新版 ftf 的時候可以順便把 USER DATA 清掉。
