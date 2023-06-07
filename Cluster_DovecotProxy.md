- Trong các thiết lập lớn => không phải mọi server đều có quyền truy cập vào tất cả các email - đơn giản vì số lượng lớn người dùng và size của file storage
- IMAP proxy như Perdition được sử dụng cho mục đích này, chuyển IMAP session của user khác tới đúng host mục tiêu trên cơ sở các mục host trong phần authentication backend
- Với Dovecot, bạn không cần sử dụng bất kỳ IMAP proxy bổ sung nào. Dovecot module imap, pop3, lmtp tự thực hiện nhiệm vụ một cách hoàn hảo. Chúng sử dụng các trường bổ sung Passdb trong một authentication data của người dùng để chỉ định mọi người dùng cho một máy chủ riêng lẻ mà sau đó tất cả các kết nối POP3, IMAP, LMTP liên quan đến người dùng đó sau đó vận chuyển
- Vì mục đích này, các thuộc tính proxy_maybe hoặc proxy phải set các trường bổ sung Passdb của người dùng.
`proxy_maybe=yes`
- Dovecot checks xem bản thân Dovecot có được liệt kê trong thuộc tính host của người dùng không. Nếu đúng như vậy, kết nối được phục vụ cục bộ như bình thường. Nếu một máy chủ khác được liệt kê, kết nối POP3/IMAP/LMTP được chuyển tiếp tới host này như một connect TCP/IP. Do đó, proxy_maybe cho phép Dovecot đồng thời trở thành proxy server và backend server ("automatic proxying")
![](https://hackmd.io/_uploads/SyUEKvT83.png)
`proxy=yes`
- Trong trường hợp này, Dovecot luôn forwards connect tới server đã liệt kê trong thuộc tính host. Nếu Dovecot server được chỉ định ở đó, điều này sẽ dẫn đến một vòng lặp vô tận và không thể login. 
