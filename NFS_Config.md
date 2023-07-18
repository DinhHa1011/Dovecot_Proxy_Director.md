## My config
- Host: `ip-host`
- Client: `ip-client`
### Step 1 - Download and Install the Components
#### On the host
```
sudo apt update
sudp apt install nfs-kernel-server
```
#### On the client
```
sudo apt update
sudo apt install nfs-common
```
### Step 2 - Creating the Share Directories on the Host
```
ls -la /var/mail
```
- Output 
```
total 8
drwxr-xr-x 2 root root 4096 Feb  7 23:21 .
drwxr-xr-x 3 root root 4096 Feb  7 23:21 ..
```
- NFS sẽ dịch bất kì hoạt động root nào trên client thành thông tin đăng nhập `nobody:nogroup` như một biện pháp bảo mật. Do đó, bạn cần thay đổi quyền sở hữu thư mục để phù hợp vớ các thông tin đăng nhập đó:
```
sudo chown nobody:nogroup /nfs/mail
```
### Step 3 - Config NFS Exports on the Host Server
```
vim /etc/exports
```
```
/var/mail       ip-client(rw,sync,no_root_squash,no_subtree_check)
```
```
systemctl restart nfs-kernel-server
```
### Step 4 - Adjusting the Firewall on the Host
```
sudo ufw status
sudo ufw allow from ip-client to any port nfs
sudo ufw status
```
- Output 
```
To                         Action      From
--                         ------      ----
Apache                     ALLOW       Anywhere                  
110                        ALLOW       Anywhere                  
22                         ALLOW       Anywhere                  
25                         ALLOW       Anywhere                  
80                         ALLOW       Anywhere                  
443                        ALLOW       Anywhere                  
587                        ALLOW       Anywhere                  
24                         ALLOW       Anywhere                               
2049                       ALLOW       ip-client                
Apache (v6)                ALLOW       Anywhere (v6)             
110 (v6)                   ALLOW       Anywhere (v6)             
22 (v6)                    ALLOW       Anywhere (v6)             
25 (v6)                    ALLOW       Anywhere (v6)             
80 (v6)                    ALLOW       Anywhere (v6)             
443 (v6)                   ALLOW       Anywhere (v6)             
587 (v6)                   ALLOW       Anywhere (v6)             
24 (v6)                    ALLOW       Anywhere (v6)             
```
### Step 5 - Creating Mount Point and Mounting Directories on the Client
- Tạo director cho mount của bạn
```
sudo mkdir -p /nfs/mail/dinhha.online
```
- mount the share bằng cách sử dụng địa chỉ IP của host server 
```
sudo mount ip-host:/var/mail /nfs/mail/dinhha.online
```
- Check
```
df -h 
```
- Out put 
```
Filesystem                    Size  Used Avail Use% Mounted on
udev                          2.0G     0  2.0G   0% /dev
tmpfs                         394M  1.1M  393M   1% /run
/dev/vda1                      39G  9.3G   30G  25% /
tmpfs                         2.0G   88K  2.0G   1% /dev/shm
tmpfs                         5.0M     0  5.0M   0% /run/lock
tmpfs                         2.0G     0  2.0G   0% /sys/fs/cgroup
tmpfs                         394M     0  394M   0% /run/user/0
/dev/loop5                     54M   54M     0 100% /snap/snapd/19122
/dev/loop1                     64M   64M     0 100% /snap/core20/1879
/dev/loop4                    167M  167M     0 100% /snap/lxd/24846
/dev/loop8                     74M   74M     0 100% /snap/core22/634
/dev/loop3                     64M   64M     0 100% /snap/core20/1891
/dev/loop0                     56M   56M     0 100% /snap/core18/2751
/dev/loop9                     54M   54M     0 100% /snap/snapd/19361
/dev/loop10                   171M  171M     0 100% /snap/lxd/24918
/dev/loop2                     74M   74M     0 100% /snap/core22/750
/dev/loop6                     56M   56M     0 100% /snap/core18/2785
ip-host:/var/mail          39G   13G   26G  33% /nfs/mail/dinhha.online
```
```
du -sh /nfs/mail
```
- Output
```
2.1G    /nfs/mail
```
### Step 7 - Mounting the Remote NFS Directories at Boot
``` 
vim /etc/fstab
```
```
ip-host:/var/mail       /nfs/mail/dinhha.online      nfs auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0
```
### Step 8 - Unmouting an NFS Remote Share
```
umount /nfs/home
```
### Fix bug
- Nếu không umount được => check thử lệnh ` lsof /nfs/mail/dinhha.online`
- Sau đó kill process
```
kill -9 <PIP>
