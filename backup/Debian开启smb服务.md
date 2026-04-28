1. 安装samba：`sudo apt install samba smbclient cifs-utils`

2. 创建samba目录

   ```shell
   sudo mkdir /mnt/public
   sudo mkdir /mnt/private
   
   sudo vim /etc/samba/smb.conf
   
   [public]
      comment = Public Folder
      path = /mnt/public
      writable = yes
      guest ok = yes
      guest only = yes
      force create mode = 775
      force directory mode = 775
   [private]
      comment = Private Folder
      path = /mnt/private
      writable = yes
      guest ok = no
      valid users = @smbshare
      force create mode = 770
      force directory mode = 770
      inherit permissions = yes
   ```

3. 创建共享用户和组

   ```shell
   sudo groupadd smb
   sudo chgrp -R smb /mnt/private /mnt/public
   sudo chmod 2770 /mnt/private
   sudo chmod 2775 /mnt/public
   # 添加无需登录用户
   sudo useradd -M -s /sbin/nologin smbuser
   # 添加到共享组中
   sudo usermod -aG smb smbuser
   # 创建密码并启用
   sudo smbpasswd -a smbuser    smbpwd
   sudo smbpasswd -e smbuser
   ```

4. 本地测试

   ```shell
   sudo testparm
   ```

5. 局域网连接：`smb://ip`

