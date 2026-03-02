# pay all in 一键部署说明
### 1. 部署环境要求
服务器系统：Linux（CentOS 7+/Ubuntu 18.04+）
Web 服务：Nginx/Apache
PHP 版本：7.1-7.4（ThinkPHP 5.1 推荐版本）
扩展：openssl、curl、mbstring、pdo_mysql
MySQL 版本：5.6+
必须配置 HTTPS（支付回调要求）

### 2. 一键部署脚本（deploy.sh）,以下是文件内容：


#!/bin/bash
# 商用支付系统一键部署脚本

# 颜色定义
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# 检查是否为root用户
if [ $UID -ne 0 ]; then
    echo -e "${RED}错误：请使用root用户执行此脚本！${NC}"
    exit 1
fi

# 配置项
APP_DIR="/var/www/tp5-pay"
DOMAIN="your-domain.com" # 替换为你的域名
DB_HOST="localhost"
DB_USER="root"
DB_PASS="your-db-pass" # 替换为你的数据库密码
DB_NAME="pay_system"

echo -e "${YELLOW}===== 开始部署商用支付系统 =====${NC}"

# 1. 安装依赖
echo -e "${YELLOW}1. 安装系统依赖${NC}"
yum install -y epel-release
yum install -y nginx php php-fpm php-mysql php-openssl php-curl php-mbstring unzip wget
systemctl start nginx php-fpm
systemctl enable nginx php-fpm

# 2. 创建目录
echo -e "${YELLOW}2. 创建应用目录${NC}"
mkdir -p $APP_DIR
chmod -R 755 $APP_DIR
chown -R nginx:nginx $APP_DIR

# 3. 解压代码包（假设代码包为pay-system.zip）
echo -e "${YELLOW}3. 解压代码包${NC}"
if [ -f "pay-system.zip" ]; then
    unzip -o pay-system.zip -d $APP_DIR
else
    echo -e "${RED}错误：未找到pay-system.zip代码包！${NC}"
    exit 1
fi

# 4. 配置数据库
echo -e "${YELLOW}4. 配置数据库${NC}"
# 创建数据库
mysql -h$DB_HOST -u$DB_USER -p$DB_PASS -e "CREATE DATABASE IF NOT EXISTS $DB_NAME DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;"
# 导入数据表
mysql -h$DB_HOST -u$DB_USER -p$DB_PASS $DB_NAME < $APP_DIR/database/pay_system.sql

# 5. 配置数据库连接
echo -e "${YELLOW}5. 配置数据库连接${NC}"
sed -i "s/localhost/$DB_HOST/g" $APP_DIR/application/database.php
sed -i "s/root/$DB_USER/g" $APP_DIR/application/database.php
sed -i "s/your-db-pass/$DB_PASS/g" $APP_DIR/application/database.php
sed -i "s/pay_system/$DB_NAME/g" $APP_DIR/application/database.php

# 6. 配置Nginx
echo -e "${YELLOW}6. 配置Nginx${NC}"
cat > /etc/nginx/conf.d/pay.conf << EOF
server {
    listen 80;
    server_name $DOMAIN;
    rewrite ^(.*)$ https://\$host\$1 permanent;
}

server {
    listen 443 ssl;
    server_name $DOMAIN;
    
    ssl_certificate /etc/nginx/ssl/$DOMAIN.crt; # 替换为你的证书路径
    ssl_certificate_key /etc/nginx/ssl/$DOMAIN.key; # 替换为你的私钥路径
    ssl_protocols TLSv1.2 TLSv1.3;
    
    root $APP_DIR/public;
    index index.php index.html;
    
    location / {
        if (!-e \$request_filename) {
            rewrite ^(.*)$ /index.php?s=\$1 last;
            break;
        }
    }
    
    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME \$document_root\$fastcgi_script_name;
        include fastcgi_params;
    }
    
    # 禁止访问敏感文件
    location ~ /\.ht {
        deny all;
    }
}
EOF

# 7. 重启服务
echo -e "${YELLOW}7. 重启服务${NC}"
systemctl restart nginx php-fpm

# 8. 设置权限
echo -e "${YELLOW}8. 设置目录权限${NC}"
chmod -R 777 $APP_DIR/runtime
chmod -R 777 $APP_DIR/public/uploads

echo -e "${GREEN}===== 部署完成 =====${NC}"
echo -e "${GREEN}前端测试地址：https://$DOMAIN${NC}"
echo -e "${GREEN}后台管理地址：https://$DOMAIN/admin${NC}"
echo -e "${YELLOW}请务必修改application/extra/下的微信/支付宝配置文件！${NC}"


### 3. 部署步骤
	将代码包（pay-system.zip）和部署脚本（deploy.sh）上传到服务器根目录
	赋予脚本执行权限：chmod +x deploy.sh
	修改脚本中的配置项（域名、数据库密码等）
	执行部署脚本：./deploy.sh
	部署完成后，修改application/extra/下的微信 / 支付宝配置文件
	访问测试地址：https:// 你的域名


## 五、常见错误排查

### 1. 微信支付相关错误
	错误现象	可能原因	解决方案
	获取 openid 失败	公众号配置错误 / 授权域名未配置	1. 检查公众号 appid 和 secret 是否正确
	2. 在公众号后台配置网页授权域名
	3. 确保授权回调地址已备案
	支付签名失败	密钥错误 / 参数格式错误	1. 检查微信支付密钥是否与商户后台一致
	2. 确保订单金额单位为分（微信）
	3. 检查回调地址是否为 HTTPS
	回调订单未更新	回调地址不可达 / 签名验证失败	1. 检查服务器防火墙是否放行支付回调 IP
	2. 查看 pay_log 日志表排查具体错误
	3. 确保回调接口返回正确的 XML 格式

### 2. 支付宝支付相关错误
	错误现象	可能原因	解决方案
	支付页面无法打开	支付宝网关错误 / 参数错误	
	1. 检查支付宝 appid 和密钥是否正确
	2. 确保使用 RSA2 密钥格式
	3. 检查订单金额格式（保留 2 位小数）
	回调验签失败	公钥配置错误	1. 检查支付宝公钥是否为商户公钥（不是应用公钥）
	2. 确保回调参数未被篡改
	3. 检查字符编码为 UTF-8

### 4. 通用排查方法
	查看支付日志表（pay_log）：记录了支付、回调、退款的所有细节
	开启 ThinkPHP 调试模式：修改config/app.php中app_debug为 true
	检查服务器日志：Nginx 日志（/var/log/nginx/）、PHP 错误日志
	验证接口连通性：使用 Postman 测试支付 / 退款接口

## 总结
	本系统基于 ThinkPHP 5.1 开发，实现了微信公众号 JSAPI 支付、支付宝全端支付的统一接入，包含完整的订单管理、自动回调、统一退款功能。
	一键部署脚本适配 Linux 系统，涵盖环境安装、代码部署、配置修改全流程，部署后需替换微信 / 支付宝的真实配置。
	常见错误排查需重点关注密钥配置、回调地址、证书权限三大核心点，支付日志表是定位问题的关键工具。
	注意：商用前需完成微信 / 支付宝商户资质认证，配置正确的商户号、密钥和回调地址，并做好数据加密和安全防护。
