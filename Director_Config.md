### Tổng quan
- Director có thể được sử dụng bởi proxy IMAP/POP3/LMTP của Dovecot để giữ người dùng tạm thời -> ánh xạ server. Miễn là người dùng có kết nối đồng thời, người dùng luôn được chuyển hướng đến cùng một máy chủ. 
- Mỗi máy chủ proxy đang chạy Director riêng và các Director đang truyền đạt trạng thái cho nhau
- Các backend được kết nối với cùng 1 nơi lưu trữ mail. Minh họa cho câu"tất cả các máy chủ đều nhìn thấy tất cả bộ lưu trữ thư"
### Quy trình xử lý của Director
- Từ laptop tiến hành đăng nhập trên director 1
- Director phân về backend 1 xử lý
- Từ smartphone đăng nhập mail trên director 2 
- Do các director truyền đạt trạng thái cho nhau (ring) nên director 2 chuyển hướng xử lý về backend 1
#### Hoạt động theo đúng lý thuyết
- AH login vào RCB director 45.124.93.82 trỏ kết nối về 39
- AH login vào RCB director 45.124.93.39 trỏ kết nối về đúng 39 trước đó đang xử lí
### My config
- Config director trên ip 45.124.93.82
`vim /etc/dovecot/conf.d/10-director`
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
director_servers = 45.124.93.82
director_mail_servers = 45.124.93.39 103.56.160.210
service imap-login {
  executable = imap-login director
}
service pop3-login {
  executable = pop3-login director
}
lmtp_proxy = yes

protocol lmtp {
  auth_socket_path = director-userdb
}

service ipc {
  unix_listener ipc {
    user = dovecot # This is already the default in v2.3.1+
  }
}
director_user_expire = 60 min
passdb {
  driver = static
  args = proxy=y nopassword=y
}
```
- Cài roundcube trên ip 45.124.93.82 (http://45.124.93.82/mail)
- Login user trên rcb http://45.124.93.82/mail
  - Khi login, user sẽ được chia về 1 trong 2 ip 45.124.93.39 hoặc 103.56.160.210
  - Nếu được chia về 45.124.93.39 => login được trên http://45.124.93.82/mail
  - Nếu được chia về 103.56.160.210 => không login được trên http://45.124.93.82/mail => vẫn login trên 45.124.93.39 và gửi nhận mail bình thường
