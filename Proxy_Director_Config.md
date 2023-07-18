![](https://hackmd.io/_uploads/r1qk2dXqh.jpg)
# Concept
- Mô hình gồm:
  - 1 imap-director
  - 1 imap-proxy
  - 3 imap-server: imap-server1, imap-server2, imap-server3 
  - mysql: imap-server1
## user
| domain | email |
|--------|-------|
| dinhha.online | ah@dinhha.online |
| dinhha.online | ndt@dinhha.online |
| dinhha.online | ndt1@dinhha.online |
| dinhha.online | ndt2@dinhha.online |
| dinhha.online | kth@dinhha.online |
| dinhha.online | kth1@dinhha.online |
| dinhha.online | kth2@dinhha.online |
## proxy 
| user | host |
|------|------|
| ah@dinhha.online | imap-server3 |
| ndt@dinhha.online | director |
| ndt1@dinhha.online | director |
| ndt2@dinhha.online | director |
| kth@dinhha.online | director |
| kth1@dinhha.online | director |
| kth2@dinhha.online | director |

- login user từ proxy => proxy forward việc auth cho server tương ứng (có thể là server 3 hoặc director)
- Nếu fw sang director => chia user sang các server khác nhau để loadbalance
  - director dùng chung db, lưu chung storage => khi 1 server xảy ra lỗi gì đấy thì sẽ chuyển sang server khác và không bị mất mail
- Khi gửi mail đến => đi qua proxy => qua server tương ứng và lưu vào storage tương ứng
# My config 
## Proxy server
- Cấu hình như một mail server
### Một số điểm khác mailserver:
- File `/etc/postfix/main.cf` `virtual_transport = lmtp:inet:127.0.0.1:24` thành như [này](https://raw.githubusercontent.com/anthanh264/linuxsetupbasic/main/Mail/main_lab_1.cf)
- File `/etc/postfix/master.cf` thành như [này](https://raw.githubusercontent.com/anthanh264/linuxsetupbasic/main/Mail/master_lab1.cf)
- Các file `virtual-domains.cf` `virtual-users.cf` `virtual-aliases.cf` `virtual-email2email.cf` phần `host` để về mysql chung `imap-server1`
- Sửa file `/etc/dovecot/conf.d/20-lmtp.conf` 
```
lmtp_proxy = yes
lmtp_save_to_detail_mailbox = yes
protocol lmtp {
 }

```
### Up lên proxy_server
- Cài gói
```
sudo apt-get install dovecot-submissiond -y
```
- File `/etc/dovecot/dovecot.conf` thêm protocols `submission` 
```
## Dovecot configuration file

# If you're in a hurry, see http://wiki2.dovecot.org/QuickConfiguration
mail_debug = yes
auth_debug = yes

!include_try /usr/share/dovecot/protocols.d/*.protocol
protocols = imap pop3 lmtp submission
listen = *
service lmtp {
   inet_listener lmtp {
      address = 0.0.0.0
      port = 24
   }
#   proxy = yes
   unix_listener lmtp {
      mode = 0666
   }
}
dict {
  #quota = mysql:/etc/dovecot/dovecot-dict-sql.conf.ext
  #expire = sqlite:/etc/dovecot/dovecot-dict-sql.conf.ext
}

!include conf.d/*.conf

!include_try local.conf

```
- File `/etc/dovecot/dovecot-sql.conf`
```
# Database driver: mysql, pgsql
driver = mysql

# Database connect string.
# Only MySQL driver support multiple hosts for now.
connect = host=localhost dbname=maildb user=mailuser password=mailPWD

# Query
password_query = SELECT NULL AS password, 'Y' as nopassword, host, destuser, 'Y' AS proxy FROM proxy WHERE user = '%u'
```

- Config MYSQL
Truy cập vào mysql
```
mysql -u root -p
```
Sử dụng database maildb
```
use maildb;
```
Tạo bảng Proxy
```
CREATE TABLE proxy (
  user varchar(255) NOT NULL,
  host varchar(16) default NULL,
  destuser varchar(255) NOT NULL default '',
  PRIMARY KEY  (user)
);
```
Thêm dữ liệu vào bảng
```
INSERT INTO `proxy` (`user`, `host`, `destuser`) VALUES
('ah@dinhha.online', 'imap-server3', 'ah@dinhha.online'),
('ndt@dinhha.online', 'director', 'ndt@dinhha.online'),
('ndt1@dinhha.online', 'director', 'ndt1@dinhha.online'),
('ndt2@dinhha.online', 'director', 'ndt2@dinhha.online'),
('kth@dinhha.online', 'director', 'kth@dinhha.online'),
('kth1@dinhha.online', 'director', 'kth1@dinhha.online'),
('kth2@dinhha.online', 'director', 'kth2@dinhha.online');
```
- Restart dovecot postfix
```
sudo systemctl restart dovecot postfix
```
## Director Server
- Cấu hình như mail server bình thương
### Những điểm khác:
- Các file `virtual-domains.cf` `virtual-users.cf` `virtual-aliases.cf` `virtual-email2email.cf` phần `host` để về mysql chung `imap-server1` 
-  File `/etc/postfix/main.cf` `virtual_transport = lmtp:inet:127.0.0.1:24` 
-  File `/etc/dovecot/dovecot.conf` thêm phần 
```
service lmtp {
   inet_listener lmtp {
      address = ip-director 127.0.0.1 ::1
      port = 24
   }
   unix_listener lmtp {
  #    mode = 0666
   }
}

```
- File `/etc/dovecot/conf.d/20-lmtp.conf`
```
lmtp_proxy = yes
lmtp_save_to_detail_mailbox = yes
protocol lmtp {
 }


```

- File `/etc/dovecot/conf.d/auth-sql.conf.ext`
```

passdb {
  driver = static
  args = proxy=y nopassword=y

}

userdb {
  driver = static
  args = uid=vmail gid=vmail home=/var/mail/%d/%u
}

```
- File `/etc/dovecot/conf.d/10-director.conf
`
```
service director {
  unix_listener login/director {
    mode = 0666
  }
  fifo_listener login/proxy-notify {
    mode = 0600
    user = $default_login_user
  }
  unix_listener director-userdb {
    mode = 0600
  }
  inet_listener {
    port = 9090
  }
}
director_servers = ip-director
director_mail_servers = imap-server2 imap-server1
service imap-login {
  executable = imap-login director
}
service pop3-login {
  executable = pop3-login director
}

protocol lmtp {
  auth_socket_path = director-userdb
}

service ipc {
  unix_listener ipc {
    user = dovecot # This is already the default in v2.3.1+
  }
}
director_user_expire = 60 min

```

## Mail server 1&2 
- Config như servermail bt
### imap-server1
-  File `/etc/dovecot/dovecot.conf` thêm phần 
```
service lmtp {
   inet_listener lmtp {
      address = imap-server1 127.0.0.1 ::1
      port = 24
   }
   unix_listener lmtp {
  #    mode = 0666
   }
}

```
-  File `/etc/dovecot/conf.d/20-lmtp.conf 
```
lmtp_proxy = yes
lmtp_save_to_detail_mailbox = yes

protocol lmtp {
}
```
### imap-server2
-  File `/etc/dovecot/dovecot.conf` thêm phần 
```
service lmtp {
   inet_listener lmtp {
      address = imap-server2 127.0.0.1 ::1
      port = 24
   }
   unix_listener lmtp {
  #    mode = 0666
   }
}

```
-  File `/etc/dovecot/conf.d/20-lmtp.conf 
```
lmtp_proxy = yes
lmtp_save_to_detail_mailbox = yes

protocol lmtp {
}
```
