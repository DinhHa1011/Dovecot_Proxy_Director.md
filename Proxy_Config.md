![](https://hackmd.io/_uploads/r1f4r3qd2.png)
## Install Dovecot
```
apt-get install dovecot dovecot-imapd dovecot-pop3d dovecot-mysql dovecot-submissiond -y
```
## Config
- File `/etc/dovecot/dovecot.conf`
```
## Dovecot 1.0 configuration file
base_dir = /var/run/dovecot/
protocols = "imap pop3 submission"
disable_plaintext_auth = no
# If you want to trade a bit of security for higher performance, change these settings:
login_process_per_connection = no
login_processes_count = 20

# If you are not moving mailboxes from host to one on daily basis you can
# use authentication cache pretty safely.
auth_cache_size = 4096

auth {
  mechanisms = plain

  # dovecot-auth only needs to be able to connect to SQL
  user = root

  # Userdb settings are not used with proxy but there need to be something.
  userdb {
    driver = static 
    args = uid=0 gid=0
  }
  passdb {
    driver = sql
    args = /etc/dovecot/dovecot-sql.conf
  }
}
service config {
  unix_listener config {
    mode = 0666
  }
}
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
- Config MySQL
```
mysql -u root -p
use maildb;
```
### Tạo bảng Proxy
```
CREATE TABLE proxy (
  user varchar(255) NOT NULL,
  host varchar(16) default NULL,
  destuser varchar(255) NOT NULL default '',
  PRIMARY KEY  (user)
);
```
- Thêm dữ liệu vào bảng
```
INSERT INTO `proxy` (`user`, `host`, `destuser`) VALUES
('ah@dinhha.online', '...39', 'ah@dinhha.online'),
('ha@dinhha.online', '...39', 'ha@dinhha.online'),
('hadt@dinhha.online', '...163', 'hadt@dinhha.online');
```
```
exit
```
- File /etc/dovecot/conf.d/10-master.conf
  - Uncomment 18, 40 mở port imap, pop3
- Restart dovecot postfix
```
systemctl restart dovecot postfix
```
