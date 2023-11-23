# 玩客雲 / 比特米盒使用說明
在電視盒子刷入安裝最新版的 Armbian 以及進行將外部儲存格式化成 ext4 並將 docker 遷移至外部儲存空間，同時說明如何正確安裝 Supervisd 版本 Home Assistant。

# Armbian 燒錄操作順序(下面以比特米盒做說明)
1. 根據網站 [點我詳閱](http://www.replace.com) 來進行玩客雲/比特米盒刷入客製韌體，往後即可藉由該韌體來刷入 Armbian
2. 前往 [Armbian-ophub Github 倉庫](https://github.com/ophub/amlogic-s9xxx-armbian/releases) 下載最新版本鏡像檔，根據今日最新版本為 **Armbian_bookworm_save_2023.11**，下載版本為 **Armbian_23.11.0_amlogic_s905x_bookworm_5.15.138_server_2023.11.12.img.gz** 此時須注意不得下載 **6.1.X** 版本的鏡像檔，否則將無法正常藉由 SDCARD/隨身碟讀取系統。
   * Armbian 有眾多版本，以下為簡略說明
    - lunar = Ubuntu 23.04
    - jammy = Ubuntu 22.04
    - bullseye = Debian 11
    - bookwrom = Debian 12
3. 使用 [balenaEtcher](https://etcher.balena.io/) 來將 *.gz 刷入SDCARD或是隨身碟。
4. 將燒錄好的 SDCARD/隨身碟重新插拔來進行讀取，進入資料夾 **extlinux** 將 **extlinux.conf.bak** 重新命名為 **extlinux.conf**。
  * 如果在步驟 5 無法正常進入系統，需重新進入 SDCARD/隨身碟內資料夾 **dtb\amlogic** 將 **meson-gxl-s905x-p212.dtb** 複製到根目錄並將其重新命名為 **u-boot.ext**。
5. 同時將網路線和編輯好的SDCARD/隨身碟插入盒子後，重新插拔電源線將會由外部系統進入 Armbian，如無法進入請重新檢查步驟 2 是否下載錯誤版本或是步驟 4 有錯誤操作。
6. 藉由 [putty](https://www.putty.org/) 等工具以 SSH 盒子 ip 來進行遠端操作。
  - 預設用戶名稱：root
  - 預設密碼：1234
7. 正確進入後，輸入指令 `armbian-install` 將會把 Armbian 燒入盒子的 EMMC，過程中會詢問幾個問題：
  - u-boot 編號：請輸入 109。
  - 檔案系統：請輸入 1，該選項內容應為 ext4。
8. 燒入完成後輸入 `poweroff`，再將 SDCARD/隨身碟拔除，重新插拔盒子電源線後即會進入 Armbian。
9. 開機後以 SSH 連入，再透過指令 `apt update && upgrade -y` 來將系統更新置最新狀態。
10. 執行 `armbian-update -k 6.1` 會今 kernel 更新至最新版本，如此步驟發生問題如無法重新進入系統則重新指定穩定版本如 `armbian-update -k 6.1.63`。

# 將外部空間隔是化為檔案系統 ext4
1. SSH 進入系統後插入外部儲存，輸入以下指令查詢磁碟路徑 `df -h`
   * 輸入後會出現類似如下畫面：
`
tmpfs           5.0M  8.0K  5.0M   1% /run/lock
tmpfs           463M  4.0K  463M   1% /tmp
/dev/mmcblk1     59G  272M   55G   1% /mnt/sdcard1
/dev/mmcblk2p1  157M  143M   14M  92% /boot
/dev/zram1       47M  2.6M   41M   6% /var/log
`
2. 其中 /dev/mmcblk1 為外部儲存空間位置，因此接下來將會以 **/dev/mmcblk1** 的路徑為範例進行格式化步驟，該位置需更換成您的路徑。
3. 將外部儲存空間格式化成gpt 系統 `parted /dev/mmcblk1 --script -- mklabel gpt`。
4. 將外部儲存空間"整個"建立為 ext4 磁區 `parted /dev/mmcblk1 --script -- mkpart primary ext4 0% 100%`。
5. 將外部儲存空間的磁區格式化成 ext4 `mkfs.ext4 -F /dev/mmcblk1`。
6. 至此外部儲存空間格式化 ext4 完成。
7. 接著將開始進行開機後系統自動掛載，在開始前須新增一位置供系統掛載，輸入以下指令來建立掛載點 `cd /mnt && mkdir sdcard1` 此處的 sdcard1 可自由命名。
8. 輸入以下指令來將外部儲存空間掛載至掛載點 `mount /dev/mmcblk1 /mnt/sdcard1` 輸入後再輸入以下確認掛載成功 `lsblk`。
   * 如成功掛載將會出現如下畫面（可見 mmcblk1 之 MOUNTPOINT 為 /mnt/sdcard1）：
`
NAME         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
mmcblk1      179:0    0  59.5G  0 disk /mnt/sdcard1
mmcblk2      179:32   0   7.3G  0 disk
├─mmcblk2p1  179:33   0   159M  0 part /boot
└─mmcblk2p2  179:34   0   6.4G  0 part /var/log.hdd
`
  * 注意！ 如以上步驟未成功掛載，則往下步驟執行將會造成系統無法開機情況。
9. 接著我們需要將其資訊寫入 fstab 來使系統自動於開機時掛載，先使用以下指令查詢外部儲存空間 ID `blkid /dev/mmcblk1`。
   - 正確執行後會出現如下資訊：
     **root@armbian:/mnt# blkid /dev/mmcblk1
     /dev/mmcblk1: UUID="23179e87-c679-4603-a994-02b589a7a054" BLOCK_SIZE="4096" TYPE="ext4"**
10. 將其 UUID 紀錄後，輸入指令 `nano /etc/fstab` 來開啟編輯檔，以鍵盤的方向鍵移動到文件末，輸入以下資訊：
    - `UUID=23179e87-c679-4603-a994-02b589a7a054 /mnt/sdcard1 ext4 defaults 0 2 `
    - 以上資訊輸入完成後請按下鍵盤的 CTRL+O 來進行文件儲存，再按下 CTRL+X 來退出文件編輯。
11. 以上步驟完成後建議入指令 `reboot` 來進行一次重開機，並在重開機後再次執行 `lsblk` 來確認掛載狀態。

# 安裝 Docker 後將預設目錄改為外部儲存空間。
1. SSH 進入系統後輸入 `curl -fsSL get.docker.com | sh` 將會開始安裝 docker。
2. 安裝完成後，輸入 `docker info | grep 'Docker Root Dir'` 確認 **Docker Root Dir: /var/lib/docker**。
3. 接著依序輸入 `systemctl stop docker.service` & `systemctl stop docker.socket` 將 Docker 終止。
4. 輸入以下指令來編輯 Docker 掛載檔案 `nano /etc/docker/daemon.json`。
   - 開啟後應為空白文件，輸入以下資訊：
     `
{
  "data-root": "/mnt/sdcard1"
}
`
完成後依序按下 CTRL+O 與 CTRL+X 來儲存並關閉檔案。
5. 輸入以下指令來將 Docker 相關文件複製到外部儲存空間 `rsync -aqxP /var/lib/docker/ /mnt/sdcard1`
6. 以上步驟完成後，依序執行以下指令來重新啟動 Docker `systemctl daemon-reload` & `systemctl start docker`
7. 稍待片刻後再輸入以下指令確認 Docker 預設目錄是否更動為目標 `docker info | grep 'Docker Root Dir'` 確認應為 **Docker Root Dir: /mnt/sdcard1**
8. 至此 Docker 的預設路徑以指定至外部儲存空間，如要更好的使用圖形化管理 Docker 建議可以輸入以下指令安裝 portainer 2.0 `docker run -d -p 8000:8000 -p 9000:9000 -p 9443:9443 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest` 安裝完成後在同網域電腦輸入 **http://IP地址:9000** 即可進入。

# 安裝 home Assistant Supervised。
1. 


