VoIP建置
===
* 目標的架構圖
![](https://i.imgur.com/jOy8PwG.png)

* 由SIP protocol stack來看，SIP屬於應用層的協定，類似於HTTP，會有交握的過程，那他是如何透過VoIP gateway來達成跟PTSN溝通?
![](https://i.imgur.com/vSAr82t.jpg)

* SIP與VoIP溝通的過程，如下圖可以清楚看到，整個訊息交換的過程。
![](https://i.imgur.com/qA8PaNk.png)

IAM = Initial Address Message
ACM = Address Complete Message
ANM = Answer Message
REL = Release
RLC = Release Complete 

 
## 以下是幾項必須完成的項目:
* [x] 架設VoIP server-Asterisk
* [x] 設定Asterisk
* [x] 測試Asterisk-經由softphone[zoiper]
* [x] 設定VoIP gateway
* [x] 結合Softphone撥打PTSN
* [x] 遭遇的問題

## 架設VoIP server-Asterisk_11.16
* 開始安裝，選擇11.16版較為穩定
![](https://i.imgur.com/AuTJxFK.png)
* 設定IP，可以設定動態或靜態的取得IP
![](https://i.imgur.com/20b5J1d.png)
* 選擇時區
![](https://i.imgur.com/kOInKNZ.png)
* 設定密碼
![](https://i.imgur.com/QBlMSPV.png)
* 開始安裝
![](https://i.imgur.com/awnR8mS.png)
* 安裝完成
![](https://i.imgur.com/NGaEQXH.png)

## 設定Asterisk
* 由/etc/asterisk/sip.conf設定使用者資訊
~~~
[general]
;context=incoming
allow=ulaw
allow=alaw
allow=gsm
bindport=5061

[SIP_trunk]
type=friend
qualify=yes
username=*****
secret=*****
insecure=very
host=dynamic
canreinvite=no
dtmfmode=rfc2833
nat=yes

[user]
type=friend
username=*****
secret=*****
host=dynamic
canreinvite=no
context=myphones
nat=yes


~~~
[general] 表示設定全域的變數
bindport  表示要進行VoIP交換的port，一般預設是5060，若出現錯誤訊息說該port號被佔了，可以           自行更改，或更換不同的port號，較不容易被破解。
[要註冊使用者的ID] -> [1000]
context   非常重要，表示使用者在撥號後，需要查詢dialplan規則的index
* nat     有4種情況可以處理NAT內的使用者，避免因NAT而掉封包
	* yes -> 若使用者在NAT內，傳統會使用STUN技術，來躲避symmetric nat，所造成的封包回              不到nat的使用者的問題，這邊不使用STUN而是直接忽略封包的IP header，直接收                下。
	* comedia
	* no  -> 就是使用者不再NAT內。
	* force_rport
* 由/etc/asterisk/extension.conf設定dialplan的規則
~~~
[general]
tatic=yes
writeprotect=no
clearglobalvars=no

[myphones]
; Call POTS numbers through Foo Provider (any number 10 digits)
exten => _0XXXXXXXXX,1,Dial(SIP/${EXTEN}@voip gateway_ip)
exten => _0XXXXXXXXX,n,Playtones(congestion)
exten => _0XXXXXXXXX,n,Hangup()
~~~

* 系統進入asterisk
	* 指令: asterisk -r/-crv/-rvvvvv
![](https://i.imgur.com/VCIiB2i.png)
* 寫完sip.conf及extension.conf，就必須將剛修正完的結果重新載入。
	![](https://i.imgur.com/YhMlmSj.png)
* 一些有用的指令
	* sip show peers
	* sip show settings
	* sip set debug on
	* ...

## 測試Asterisk-經由softphone[zoiper]
* SIP的APP有非常多種，如3CX、Sipdroid、Zoiper...
* 該網站有非常詳細的介紹，這邊就不多贅述。
	* http://www.faraway.com.tw/zoiper

## 遭遇的問題
* FXO FXS傻傻分不清楚
	* FXO :接局端的孔，一般不會供電
	* FXS :接話機的孔，通常會供電給話機
	* 把電話線接在FXS，會發生兩邊都在供電，最後會造成GW掛掉!(小心)
* SIP proxy 與 SIP server差異性，最大的差別在於，服務的使用者的量，SIP proxy可以服務較多的使用者，相較於SIP server，穩定度我推薦SIP server，也較不會出現單邊通話的窘境!
* 有鑑於被盜打的危險，會將VoIP gateway放在內網!
* 需要註冊SIP trunk在asterisk server上，才能讓voip gateway可以走此一trunk出去FXO，這個是最關鍵的問題。
 
## __難題__
 * 目前直接連AP內的通話皆可以正常雙向
 * 若是外部使用者，就會遭遇到NAT的問題，封包在進入NAT就會被砍掉，無法傳輸使用者的封包給要接收者


### TP-Link 刷 DD-wrt
1. 首先登入[DD-WRT](http://www.dd-wrt.com/site/index)網站
2. 點選Router-Database，搜尋要準備刷機的TP-Link 740n
3. 選擇正確的740N的版本，版本號在AP上可以看到
4. 下載兩個檔案，「factory-to-ddwrt.bin」及「tl-wr740nv4-webflash.bin」
5. 然後就可以開始刷機，登入TP-link的管理介面（192.168.0.1）
6. 找到韌體更新的選項後，將「factory-to-ddwrt.bin」檔案選到，就開始刷機了！
7. 刷機後要再刷一次才會有中文，選擇「tl-wr740nv4-webflash.bin」，若可以進到網頁介面，就代表刷機完成。
8. DD-WRT default account:root pass:admin


## 參考資料
* http://www.voip-info.org/wiki/view/Asterisk+sip+nat
* https://www.beardy.se/how-to-set-up-a-sip-trunk-in-the-asterisk-pbx
* http://www.asterisk.org/
* http://www.welltech.com/support/Applications_Welltech_Asterisk.PDF