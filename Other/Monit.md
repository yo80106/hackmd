# Monit使用教學
這是一篇關於使用Monit的文章，Go、Go、Go!
## 安裝與初始化Monit
如果你是使用ubuntu，使用apt-get安裝Monit，如下：
```
sudo apt-get install monit
```
Monit安裝好後，就可以開始編輯控制檔(control file)。這個控制檔是使用Monit的核心，所有要透過Monit執行的監控或作業