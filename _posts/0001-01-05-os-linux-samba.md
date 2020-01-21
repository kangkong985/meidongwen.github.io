##### 软件

- samba：這個軟體主要提供了 SMB 伺服器所需的各項服務程式 (smbd 及 nmbd)、 的文件檔、以及其他與 SAMBA 相關的 logrotate 設定檔及開機預設選項檔案等；
- samba-client：這個軟體則提供了當 Linux 做為 SAMBA Client 端時，所需要的工具指令，例如掛載 SAMBA 檔案格式的 mount.cifs、 取得類似網芳相關樹狀圖的 smbtree 等等；
- samba-common：這個軟體提供的則是伺服器與用戶端都會使用到的資料，包括 SAMBA 的主要設定檔 (smb.conf)、語法檢驗指令 (testparm)等等；



相關的設定檔：

- /etc/samba/smb.conf： 這是 Samba 的主要設定檔，基本上，咱們的 Samba 就僅有這個設定檔而已，且這個設定檔本身就是很詳細的說明文件了，請用vim 去查閱它吧！主要的設定項目分為伺服器的相關設定 (global)，如工作群組、NetBIOS 名稱與密碼等級等，以及分享的目錄等相關設定，如實際目錄、分享資源名稱與權限等等兩大部分。
- /etc/samba/lmhosts：早期的 NetBIOS name 需額外設定，因此需要這個 lmhosts 的 NetBIOS name 對應的 IP 檔。事實上它有點像是 /etc/hosts 的功能！只不過這個 lmhosts 對應的主機名稱是 NetBIOS name 喔！不要跟 /etc/hosts 搞混了！目前 Samba 預設會去使用你的本機名稱 (hostname) 作為你的 NetBIOS  name，因此這個檔案不設定也無所謂。
- /etc/sysconfig/samba：提供啟動 smbd, nmbd 時，你還想要加入的相關服務參數。
- /etc/samba/smbusers：由於 Windows 與 Linux 在管理員與訪客的帳號名稱不一致，例如： administrator (windows) 及 root(linux)，為了對應這兩者之間的帳號關係，可使用這個檔案來設定
- /var/lib/samba/private/{passdb.tdb,secrets.tdb}：管理 Samba 的使用者帳號/密碼時，會用到的資料庫檔案；
- /usr/share/doc/samba-<版本>：這個目錄包含了 SAMBA 的所有相關的技術手冊喔！也就是說，當你安裝好了 SAMBA 之後，你的系統裡面就已經含有相當豐富而完整的 SAMBA 使用手冊了！值得高興吧！ ^_^，所以，趕緊自行參考喔！

至於常用的指令檔案方面，若分為伺服器與用戶端功能，則主要有底下這幾個資料：

- /usr/sbin/{smbd,nmbd}：伺服器功能，就是最重要的權限管理 (smbd) 以及 NetBIOS name 查詢 (nmbd) 兩個重要的服務程式；
- /usr/bin/{tdbdump,tdbtool}：伺服器功能，在 Samba 3.0以後的版本中，使用者的帳號與密碼參數已經轉為使用資料庫了！Samba 使用的資料庫名稱為 TDB (Trivial DataBase)。既然是使用資料庫，當然要使用資料庫的控制指令來處理囉。tdbdump 可以察看資料庫的內容，tdbtool 則可以進入資料庫操作介面直接手動修改帳密參數。不過，你得要安裝 tdb-tools這個軟體才行；
- /usr/bin/smbstatus：伺服器功能，可以列出目前 Samba 的連線狀況，包括每一條 Samba  連線的 PID, 分享的資源，使用的用戶來源等等，讓你輕鬆管理 Samba 啦；
- /usr/bin/{smbpasswd,pdbedit}：伺服器功能，在管理 Samba 的使用者帳號密碼時，早期是使用 smbpasswd 這個指令，不過因為後來使用 TDB 資料庫了，因此建議使用新的 pdbedit 指令來管理用戶資料；
- /usr/bin/testparm：伺服器功能，這個指令主要在檢驗設定檔 smb.conf  的語法正確與否，當你編輯過 smb.conf 時，請務必使用這個指令來檢查一次，避免因為打字錯誤引起的困擾啊！
- /sbin/mount.cifs：用戶端功能，在 Windows 上面我們可以設定『網路磁碟機』來連接到自己的主機上面。在 Linux 上面，我們則是透過 mount (mount.cifs)來將遠端主機分享的檔案與目錄掛載到自己的 Linux 主機上面哪！
- /usr/bin/smbclient：用戶端功能，當你的 Linux 主機想要藉由『網路上的芳鄰』的功能來查看別台電腦所分享出來的目錄與裝置時，就可以使用 smbclient來查看啦！這個指令也可以使用在自己的 SAMBA 主機上面，用來查看是否設定成功哩！
- /usr/bin/nmblookup：用戶端功能，有點類似 nslookup 啦！重點在查出NetBIOS name 就是了。
- /usr/bin/smbtree：用戶端功能，這玩意就有點像 Windows 系統的網路上的芳鄰顯示的結果，可以顯示類似『靠近我的電腦』之類的資料，能夠查到工作群組與電腦名稱的樹狀目錄分佈圖！