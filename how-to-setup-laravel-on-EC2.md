# Hướng dẫn cài đặt Laravel + PHP8.1 + Nginx trên EC2 (Amazon linux 2)

1. Cài đặt Nginx

```bash
# cập nhật repo cho hệ điều hành
sudo yum update -y

# kiểm tra xem nginx có sẵn trong repository không
sudo amazon-linux-extras list | grep nginx
# bật package nginx
sudo amazon-linux-extras enable nginx1
# cài đặt nginx
sudo yum clean metadata && sudo yum install nginx -y
# kiểm tra version nginx
nginx -v
# Khởi động nginx
sudo systemctl start nginx
# Bật chế độ tự khởi động nginx khi server bị restart
sudo systemctl enable nginx

```

2. Cài đặt PHP 8.1

```bash
# kiểm tra xem php8.1 có sẵn trong repository không
sudo amazon-linux-extras list | grep php8.1
# cài đặt PHP 8.1
sudo amazon-linux-extras enable php8.1
yum clean metadata
# Cài đặt các extension cần thiết cho PHP
sudo yum install php php-cli php-fpm php-mysqlnd php-pdo php-common php-json php-mysqlnd php-bcmath -y
sudo yum install php-gd php-mbstring php-xml php-dom php-intl php-simplexml -y

# Khởi chạy php-fpm
sudo systemctl start php-fpm
# Bật chế độ tự khởi động khi server bị restart
sudo systemctl enable php-fpm

# tạo folder chứa source code
cd /var/www/
sudo mkdir app
cd /var/www/app

# Cấp quyền truy cập
sudo chown -R ec2-user:nginx /var/www
sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
find /var/www -type f -exec sudo chmod 0664 {} \;
```

3. Cài đặt Mysql

```bash
# Thêm MySQL Community Repository
sudo wget https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
sudo rpm -ivh mysql57-community-release-el7-11.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022

# Cài đặt MYSQL
sudo yum install -y mysql-community-server 
# Bật chế độ tự khởi động khi server restart
sudo systemctl enable mysqld 
# Khởi chạy dịch vụ mysql
sudo systemctl start mysqld 
# Câu lệnh lấy password tạm thời cho user root
sudo grep 'temporary password' /var/log/mysqld.log 
# Đăng nhập vào database với password lấy được ở phía trên
mysql -u root -p

# Lần đầu hệ thống sẽ yêu cầu đổi mật khẩu, chạy lệnh sau để đổi mật khẩu root
SET PASSWORD = PASSWORD('password@1234Aa');
 
# Tạo schema mới
CREATE DATABASE laravel_app;
# user và gán quyền truy cập vào database
# trường hợp không cho remote access
CREATE USER 'php_user'@'localhost' IDENTIFIED BY 'php_pass@Aa123';
# trường hợp cho phép remote access (dùng tool để remote vào DB) thì đổi localhost thành %
CREATE USER 'php_user'@'%' IDENTIFIED BY 'php_pass@Aa123';
# gán quyền cho user mới tạo
GRANT ALL PRIVILEGES ON *.* TO 'php_user'@'%' WITH GRANT OPTION;
# reload quyền
FLUSH PRIVILEGES;

# trường hợp remote đến database không thành công thì cần check lại xem đã mở port 3306 ở security group or firewalld của server chưa.
```

4. cài đặt git 

```bash
sudo yum -y install git
```

5. cài đặt composer

```bash
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
HASH="$(wget -q -O - https://composer.github.io/installer.sig)"
php -r "if (hash_file('SHA384', 'composer-setup.php') === '$HASH') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
sudo chmod 775 -R /usr/local/bin
composer -v
```

6. cài đặt NodeJS & npm

```bash
# Tải nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
. ~/.nvm/nvm.sh
# cài đặt node
nvm install node
# kiểm tra các version node đã cài 
nvm list
# cài đặt node version 
nvm install v16.10.0
# chuyển version node sang 16
nvm use v16.10.0
# chuyển version mặc định sang 16
nvm alias default v16.10.0
```

7. thay đổi timezone của server

```bash
# kiểm tra timezone hiện tại
timedatectl
# danh sách timezone
timedatectl list-timezones
# thay đổi sang timezone nhật
sudo timedatectl set-timezone Asia/Tokyo
# kiểm tra lại xem đã thay đổi chưa
date
# với cấu hình trên khi reset server, timezone sẽ quay trở về mặc đinh, để cài đặt tự đổi sang timezone mới cho dù reset server thì chạy lệnh dưới
sudo rm -rf /etc/localtime
sudo ln -s /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
```

8. clone source code

```bash
# ở đây sử dụng source code bagisto cho việc demo cài đặt
composer create-project bagisto/bagisto
# thay đổi thông tin kết nối trong file env
sudo nano .env
# chạy lệnh cài đặt
php artisan bagisto:install

#Xem thêm hướng dẫn ở đây: https://webkul.com/blog/laravel-ecommerce-website/
```

9. cấu hình vhost nginx

```bash
# tạo file vhost mới
sudo nano /etc/nginx/laravel_app.conf

# thay đổi nội dung và paste vào file
server {
    listen 80;
    listen [::]:80 ipv6only=on;

    root /var/www/app/bagisto/public;
    index index.php index.html index.htm;

    server_name _;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri /index.php =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php-fpm/www.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}

# restart lại nginx
sudo systemctl restart nginx

# truy cập bằng IP thử xem vào dc chưa, trường hợp ko vào được thì kiểm tra xem đã mở port 80 ở security group or firewalld chưa nhé

```

10. cấp quyền

```bash
# cấp quyền cho folder
sudo chown -R ec2-user:nginx /var/www
sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
find /var/www -type f -exec sudo chmod 0664 {} \;

# thêm user ec2-user vào ->  /etc/nginx/nginx.conf
user ec2-user nginx; 

# đổi lại user, group trong php-fpm -> /etc/php-fpm.d/www.conf
user = ec2-user
group = nginx

# khởi động lại nginx và php-fpm
systemctl restart nginx
systemctl restart php-fpm

# cài đặt thư viện semanage
sudo yum install policycoreutils-python -y

# cấu hình SELinux
semanage fcontext -a -t httpd_sys_content_t "/var/www/app/bagisto(/.*)?"
semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/app/bagisto/storage(/.*)?"
semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/app/bagisto/bootstrap/cache(/.*)?"
semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/app/bagisto/vendor(/.*)?"
restorecon -Rv /var/www/app/bagisto
```

11. cài đặt supervisord để quản lý background job

```bash
# cài đặt thêm gói mở rộng 
sudo amazon-linux-extras install epel
# cài đặt supervisord
sudo yum install -y supervisor
# khởi động service
sudo systemctl start supervisord.service
# khởi động service cùng server
sudo systemctl enable supervisord.service
# kiểm tra trạng thái
sudo systemctl status supervisord.service
# thêm cấu hình mới
cd /etc/supervisord.d/
sudo nano laravel_app_queue.ini
# dán config sau vào file (thay đổi tương ứng với project của bạn)
[program:laravel_app_queue]
process_name=%(program_name)s_%(process_num)02d
command=/usr/bin/php /var/www/app/bagisto/artisan queue:work --queue=high,medium,default,mail --tries=3
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=ec2-user
#numprocs=1
redirect_stderr=true
stdout_logfile=/var/www/app/bagisto/storage/logs/worker.log
stopwaitsecs=3600

# khởi động lại supervisord
sudo systemctl restart supervisord.service
```

12. cài đặt ssl (chỉ dành cho domain) (mở rộng)

```bash
sudo yum-config-manager --enable epel
sudo yum install certbot python-certbot-nginx
sudo certbot --nginx
sudo certbot certonly --nginx
```

13. cài đặt tự động renew ssl
# cách 1: tạo file certbot trong folder /etc/cron.d với nội dung sau
```bash
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
0*/12***root certbot -q renew --nginx
```
# cách 2
```bash
sudo crontab -e
0 12 * * * /usr/bin/certbot renew --quiet
```
