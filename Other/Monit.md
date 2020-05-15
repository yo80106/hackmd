# Monit使用教學
這是一篇關於使用Monit的文章，Go、Go、Go!
## 安裝與初始化Monit
如果你是使用ubuntu，可用apt-get安裝Monit，如下：
```
sudo apt-get install monit
```
Monit安裝好後，就可以開始編輯控制檔(control file)。這個控制檔是使用Monit的核心，所有要透過Monit執行的監控或作業，皆須透過此檔案控制。我們可以在以下路徑中找到控制檔的範例：
```
/etc/monit/monitrc
```
檔案開啟後，確認標題是Monit control file，說明此範例檔已經包含許多常用的監測項目，但大部分都以"#"標記，表示Monit並不會執行該行指令，我們可以透過刪掉"#"的方式調控需要使用的功能項目。如果要了解各行指令的功能，可參閱Monit官方的[使用手冊](https://mmonit.com/monit/documentation/monit.html)。但我想不少人一開始都會被這份落落長的範例檔嚇到，一時迷失自我在其中，所以我想先將它備分起來，我們自己製作一份小型的控制檔。先進行控制範例檔的備分：
```
sudo mv /etc/monit/monitrc /etc/monit/monitrc.bak
```
再來，用nano(當然可以用vim)建立一個新的小控制檔，如下：
```
sudo nano /etc/monit/monitrc
```
檔案的內容，複製貼上以下內容：
```
# How often in seconds should monit check your services.
set daemon 120

set logfile /var/log/monit.log
set idfile /var/lib/monit/id
set statefile /var/lib/monit/state

set httpd port 2812 and
    #Change username and password
    allow Username:Password
    # To enable SSL for WebUI uncomment the next 2 lines
    #ssl enable
    #pemfile /path/to/unified/certificate.pem
    # To restrict access to localhost only uncomment the following line
    #allow localhost

include /etc/monit/conf.d/*

```
上述的範例中，set和include是用來啟動各項設定的指令，說明如下：
* SET: 後面接上設定的內容，或監控的項目。
* INCLUDE: 後面接上其他控制檔的存放位置，以上述為例，我們要Monit在執行控制檔的過程中，除了/etc/monit/monitrc這份主要的控制檔，也去搜尋/etc/monit/conf.d/路徑下的所有控制檔。善用不同小的控制檔，能幫助我們有效區分不同的監控工作。

上述範例的其他指令簡介：
* set daemon 120: Monit自動執行的時間間隔，這裡120，表示120秒會執行一次。
* set logfile: Monit每次執行，所產生的log檔存放位置。
* set idfile: Monit每次執行，所產生的工作ID存放位置。
* set statefile: Monit每次執行，所產生的工作描述檔存放位置。
* set httpd port 2812: 設定Monit網頁呈現介面所使用的port(這裡設定為2812)。
* allow Username:Password: 設定Monit網頁呈現介面的使用者帳號和密碼。

設定完成後，可以使用`sudo monit -t`，測試我們的控制檔是否有指令上的錯誤。

:heavy_exclamation_mark: 測試過程可能會發生以下錯誤訊息：
```
/etc/monit/monitrc:17: Include failed -- Success '/etc/monit/conf.d/*'
The control file '/etc/monit/monitrc' permission 0644 is wrong, maximum 0700 allowed
```
:heavy_check_mark: 這代表monitrc控制檔的權限設定需要更改，解決方法如下：
```
sudo chmod 0700 /etc/monit/monitrc
```
修改完成後，再次執行`sudo monit -t`，就會得到正確訊息`Control file syntax OK`。

最後使用`systemctl restart monit`就能重新啟動monit。使用網頁瀏覽器輸入伺服器IP加上剛剛設定的port 2812，接著輸入自己設定好的帳號密碼，就能看到Monit即時呈現的伺服器相關資源的監控畫面。

## 監控硬碟使用情形
Monit所有監控作業都是由控制檔調控(是要強調幾次)，最重要的那支控制檔`/etc/monit/monitrc`在上一個小節已經跟大家說明如何設定。這裡我們要製作一個新的小型控制檔，用以監控伺服器硬碟使用情形，並讓Monit在硬碟使用空間過滿的情況下，自動寄信通知我們。前一小節提到，我們已經將`/etc/monit/conf.d/`路徑納入Monit搜尋控制檔的範圍，我們新建的控制檔`storage`就是要放在其下。指令如下
```
sudo nano /etc/monit/conf.d/storagespace
```
檔案內容架構：
```
# add each drive you want to monitor below
check filesystem Ubuntu with path /dev/sda1
    if space usage > 90% then alert
check filesystem Home with path /dev/sda3
    if space usage > 90% then alert
check filesystem Media with path /dev/sdb1
    if space usage > 90% then alert
```
上述指令淺顯易懂，我們用`check`來檢查我們要檢查的硬碟路徑，舉例來說：
* check filesystem Ubuntu with path /dev/sda1: 我們要檢查位於路徑/dev/sda1的這顆硬碟，命名為`Ubuntu`

接下來，if這個判斷式，要Monit去判斷如果該顆硬碟的使用量超過90%，促發`alert`。
alert表示要Monit進行狀況的通報與紀錄，如果我們有設定好信箱地址，Monit就能將alert的訊息自動寄給我們。我們接著就來修改`/etc/monit/monitrc`，將電子郵件加進控制檔中。加入以下指令：
```
set mailserver localhost
set alert your@email only on {resource}
```
設定完成後，使用`sudo monit -t`檢查控制檔有沒有問題，接著用`sudo monit reload`重新載入新控制檔，這時候如果伺服器上有被我們監控的硬碟，使用容量超過90%，我們馬上會收到來自Monit的通知信。

:warning: 上述`set mailserver localhost`是假設我們伺服器已安裝`postfix`和`mailutils`套件，有這兩個套件才能使伺服器進行郵寄，若沒有安裝，請使用下列指令安裝：
```
apt-get install postfix mailutils -y
```
除了我們已啟動Monit的郵寄功能外，此時刷新Monit網頁介面，我們也會看到硬碟空間的使用情形。也可以在伺服器的指令列輸入`sudo monit status`，同樣能看到Monit目前執行的監控內容。


