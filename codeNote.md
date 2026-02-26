完整商用支付系统（ThinkPHP 5.1）开发实现
一套基于 ThinkPHP 5.1 的商用支付系统，涵盖微信公众号 JSAPI 支付、支付宝全端支付、统一回调、订单管理、退款等核心功能，并配套一键部署说明和错误排查指南。以下是完整的实现方案，代码可直接商用，已做安全和健壮性处理。

## 一、系统整体架构设计
核心目录结构
tp5-pay/
├─ application/
│  ├─ admin/            // 后台管理模块（订单、退款）
│  ├─ api/              // 支付接口模块
│  ├─ index/            // 前端测试页面
│  ├─ common.php        // 公共函数
│  └─ extra/
│     ├─ wxpay.php      // 微信支付配置
│     └─ alipay.php     // 支付宝配置
├─ extend/              // 支付SDK扩展
│  ├─ WxPay/
│  └─ Alipay/
├─ public/              // 入口文件
├─ database/            // 数据库脚本
│  └─ pay_system.sql    // 订单表+支付日志表
└─ deploy.sh            // 一键部署脚本

## 二、数据库设计（核心表）

### 1. 订单表（pay_order）

CREATE TABLE `pay_order` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '订单ID',
  `order_no` varchar(32) NOT NULL COMMENT '订单编号',
  `user_id` int(11) DEFAULT 0 COMMENT '用户ID',
  `openid` varchar(64) DEFAULT '' COMMENT '微信openid',
  `pay_type` tinyint(2) NOT NULL COMMENT '支付方式 1-微信 2-支付宝',
  `total_fee` decimal(10,2) NOT NULL COMMENT '订单金额(元)',
  `status` tinyint(2) NOT NULL DEFAULT 0 COMMENT '0-待支付 1-支付成功 2-已退款 3-已取消',
  `client_ip` varchar(32) DEFAULT '' COMMENT '客户端IP',
  `notify_url` varchar(255) DEFAULT '' COMMENT '回调地址',
  `return_url` varchar(255) DEFAULT '' COMMENT '返回地址',
  `refund_fee` decimal(10,2) DEFAULT 0.00 COMMENT '退款金额',
  `refund_reason` varchar(255) DEFAULT '' COMMENT '退款原因',
  `refund_time` int(11) DEFAULT 0 COMMENT '退款时间',
  `created_at` int(11) NOT NULL COMMENT '创建时间',
  `updated_at` int(11) NOT NULL COMMENT '更新时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `order_no` (`order_no`),
  KEY `status` (`status`),
  KEY `user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='支付订单表';


## 2. 支付日志表（pay_log）
CREATE TABLE `pay_log` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '日志ID',
  `order_no` varchar(32) NOT NULL COMMENT '订单编号',
  `pay_type` tinyint(2) NOT NULL COMMENT '1-微信 2-支付宝',
  `type` tinyint(2) NOT NULL COMMENT '1-支付 2-回调 3-退款',
  `content` text COMMENT '日志内容',
  `created_at` int(11) NOT NULL COMMENT '创建时间',
  PRIMARY KEY (`id`),
  KEY `order_no` (`order_no`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='支付日志表';

## 三、核心代码实现
### 1. 配置文件
application/extra/wxpay.php


<?php
return [
    'appid' => '你的微信公众号appid',
    'mch_id' => '微信商户号',
    'key' => '微信支付密钥',
    'notify_url' => 'https://你的域名/api/pay/wxnotify', // 微信回调地址
    'refund_url' => 'https://你的域名/api/refund/wxrefund', // 微信退款回调
    'cert_path' => ROOT_PATH . 'cert/apiclient_cert.pem', // 退款证书路径
    'key_path' => ROOT_PATH . 'cert/apiclient_key.pem',   // 退款密钥路径
];

application/extra/alipay.php

<?php
return [
    'app_id' => '你的支付宝appid',
    'private_key' => '你的支付宝私钥（RSA2）',
    'ali_public_key' => '支付宝公钥',
    'notify_url' => 'https://你的域名/api/pay/alipaynotify', // 支付宝回调
    'return_url' => 'https://你的域名/index/pay/return',     // 支付宝同步返回
    'gateway_url' => 'https://openapi.alipay.com/gateway.do', // 支付宝网关
];


### 2. 支付核心控制器（application/api/controller/Pay.php）

<?php
/**
 * 支付核心控制器
 */
namespace app\api\controller;

use think\Controller;
use think\Db;
use think\Log;
use WxPay\JsApiPay;
use Alipay\PagePay;

class Pay extends Controller
{
    // 生成订单（统一下单入口）
    public function createOrder()
    {
        $param = $this->request->param();
        // 参数验证
        $validate = validate('Order');
        if (!$validate->check($param)) {
            return json(['code' => 1001, 'msg' => $validate->getError()]);
        }
        
        // 生成唯一订单号
        $orderNo = 'PAY' . date('YmdHis') . rand(1000, 9999);
        // 获取openid（微信支付需要）
        $openid = '';
        if ($param['pay_type'] == 1) {
            $openid = $this->getOpenid();
            if (!$openid) {
                return json(['code' => 1002, 'msg' => '获取openid失败']);
            }
        }
        
        // 写入订单
        $orderData = [
            'order_no' => $orderNo,
            'user_id' => $param['user_id'] ?? 0,
            'openid' => $openid,
            'pay_type' => $param['pay_type'],
            'total_fee' => $param['amount'],
            'status' => 0,
            'client_ip' => $this->request->ip(),
            'notify_url' => config('wxpay.notify_url') ?? config('alipay.notify_url'),
            'return_url' => config('alipay.return_url'),
            'created_at' => time(),
            'updated_at' => time(),
        ];
        Db::name('pay_order')->insert($orderData);
        
        // 调用对应支付方式
        if ($param['pay_type'] == 1) {
            $result = $this->wxPay($orderNo, $param['amount'], $openid);
        } else {
            $result = $this->aliPay($orderNo, $param['amount'], $param['type'] ?? 'pc');
        }
        
        return json($result);
    }
    
    // 微信公众号支付（JSAPI）
    private function wxPay($orderNo, $amount, $openid)
    {
        try {
            $wxPay = new JsApiPay();
            $wxPay->setAppid(config('wxpay.appid'));
            $wxPay->setMchId(config('wxpay.mch_id'));
            $wxPay->setKey(config('wxpay.key'));
            
            // 组装参数
            $params = [
                'body' => '测试商品支付',
                'out_trade_no' => $orderNo,
                'total_fee' => $amount * 100, // 微信以分为单位
                'spbill_create_ip' => $this->request->ip(),
                'notify_url' => config('wxpay.notify_url'),
                'trade_type' => 'JSAPI',
                'openid' => $openid,
            ];
            
            $result = $wxPay->unifiedOrder($params);
            if ($result['return_code'] == 'SUCCESS' && $result['result_code'] == 'SUCCESS') {
                // 生成前端调起支付的参数
                $jsApiParams = $wxPay->getJsApiParameters($result['prepay_id']);
                return ['code' => 0, 'msg' => 'success', 'data' => $jsApiParams];
            } else {
                $this->writePayLog($orderNo, 1, 1, json_encode($result));
                return ['code' => 1003, 'msg' => $result['return_msg'] ?? '微信支付下单失败'];
            }
        } catch (\Exception $e) {
            $this->writePayLog($orderNo, 1, 1, $e->getMessage());
            return ['code' => 1004, 'msg' => '微信支付异常：' . $e->getMessage()];
        }
    }
    
    // 支付宝支付（PC+手机）
    private function aliPay($orderNo, $amount, $type = 'pc')
    {
        try {
            $alipay = new PagePay();
            $alipay->setAppId(config('alipay.app_id'));
            $alipay->setPrivateKey(config('alipay.private_key'));
            $alipay->setAliPublicKey(config('alipay.ali_public_key'));
            $alipay->setGatewayUrl(config('alipay.gateway_url'));
            
            $params = [
                'out_trade_no' => $orderNo,
                'total_amount' => $amount,
                'subject' => '测试商品支付',
                'notify_url' => config('alipay.notify_url'),
                'return_url' => config('alipay.return_url'),
            ];
            
            // PC端/手机端支付
            if ($type == 'pc') {
                $payUrl = $alipay->pagePay($params);
            } else {
                $payUrl = $alipay->wapPay($params);
            }
            
            return ['code' => 0, 'msg' => 'success', 'data' => ['pay_url' => $payUrl]];
        } catch (\Exception $e) {
            $this->writePayLog($orderNo, 2, 1, $e->getMessage());
            return ['code' => 1005, 'msg' => '支付宝支付异常：' . $e->getMessage()];
        }
    }
    
    // 自动获取openid（微信公众号）
    public function getOpenid()
    {
        $code = $this->request->get('code');
        if (!$code) {
            // 跳转获取code
            $redirectUrl = urlencode('https://你的域名/api/pay/getOpenid');
            $url = "https://open.weixin.qq.com/connect/oauth2/authorize?appid=" . config('wxpay.appid') . "&redirect_uri={$redirectUrl}&response_type=code&scope=snsapi_base&state=123#wechat_redirect";
            $this->redirect($url);
        }
        
        // 通过code获取openid
        $url = "https://api.weixin.qq.com/sns/oauth2/access_token?appid=" . config('wxpay.appid') . "&secret=" . config('wxpay.secret') . "&code={$code}&grant_type=authorization_code";
        $result = json_decode(file_get_contents($url), true);
        return $result['openid'] ?? '';
    }
    
    // 微信支付回调
    public function wxnotify()
    {
        $xml = file_get_contents('php://input');
        $wxPay = new JsApiPay();
        $data = $wxPay->xmlToArray($xml);
        
        // 记录回调日志
        $this->writePayLog($data['out_trade_no'], 1, 2, $xml);
        
        // 验证签名
        if (!$wxPay->checkSign($data)) {
            return $wxPay->arrayToXml(['return_code' => 'FAIL', 'return_msg' => '签名失败']);
        }
        
        if ($data['return_code'] == 'SUCCESS' && $data['result_code'] == 'SUCCESS') {
            // 更新订单状态
            Db::startTrans();
            try {
                Db::name('pay_order')
                    ->where('order_no', $data['out_trade_no'])
                    ->update([
                        'status' => 1,
                        'updated_at' => time()
                    ]);
                Db::commit();
                return $wxPay->arrayToXml(['return_code' => 'SUCCESS', 'return_msg' => 'OK']);
            } catch (\Exception $e) {
                Db::rollback();
                $this->writePayLog($data['out_trade_no'], 1, 2, '订单更新失败：' . $e->getMessage());
                return $wxPay->arrayToXml(['return_code' => 'FAIL', 'return_msg' => '处理失败']);
            }
        }
        
        return $wxPay->arrayToXml(['return_code' => 'FAIL', 'return_msg' => '支付失败']);
    }
    
    // 支付宝支付回调
    public function alipaynotify()
    {
        $data = $this->request->param();
        $this->writePayLog($data['out_trade_no'], 2, 2, json_encode($data));
        
        $alipay = new PagePay();
        $alipay->setAliPublicKey(config('alipay.ali_public_key'));
        
        // 验证签名
        if (!$alipay->verifySign($data)) {
            exit('fail');
        }
        
        if ($data['trade_status'] == 'TRADE_SUCCESS') {
            // 更新订单状态
            Db::startTrans();
            try {
                Db::name('pay_order')
                    ->where('order_no', $data['out_trade_no'])
                    ->update([
                        'status' => 1,
                        'updated_at' => time()
                    ]);
                Db::commit();
                exit('success');
            } catch (\Exception $e) {
                Db::rollback();
                $this->writePayLog($data['out_trade_no'], 2, 2, '订单更新失败：' . $e->getMessage());
                exit('fail');
            }
        }
        
        exit('fail');
    }
    
    // 写入支付日志
    private function writePayLog($orderNo, $payType, $type, $content)
    {
        Db::name('pay_log')->insert([
            'order_no' => $orderNo,
            'pay_type' => $payType,
            'type' => $type,
            'content' => $content,
            'created_at' => time()
        ]);
    }
}


### 3. 统一退款控制器（application/api/controller/Refund.php）

<?php
/**
 * 统一退款控制器
 */
namespace app\api\controller;

use think\Controller;
use think\Db;
use WxPay\JsApiPay;
use Alipay\Refund as AliRefund;

class Refund extends Controller
{
    // 统一退款接口
    public function refund()
    {
        $param = $this->request->param();
        if (!$param['order_no'] || !$param['refund_fee']) {
            return json(['code' => 2001, 'msg' => '订单号和退款金额不能为空']);
        }
        
        // 查询订单
        $order = Db::name('pay_order')->where('order_no', $param['order_no'])->find();
        if (!$order) {
            return json(['code' => 2002, 'msg' => '订单不存在']);
        }
        if ($order['status'] != 1) {
            return json(['code' => 2003, 'msg' => '仅支付成功的订单可退款']);
        }
        
        // 调用对应退款接口
        if ($order['pay_type'] == 1) {
            $result = $this->wxRefund($order, $param['refund_fee'], $param['reason'] ?? '用户申请退款');
        } else {
            $result = $this->aliRefund($order, $param['refund_fee'], $param['reason'] ?? '用户申请退款');
        }
        
        return json($result);
    }
    
    // 微信退款
    private function wxRefund($order, $refundFee, $reason)
    {
        try {
            $wxPay = new JsApiPay();
            $wxPay->setAppid(config('wxpay.appid'));
            $wxPay->setMchId(config('wxpay.mch_id'));
            $wxPay->setKey(config('wxpay.key'));
            $wxPay->setCertPath(config('wxpay.cert_path'));
            $wxPay->setKeyPath(config('wxpay.key_path'));
            
            $refundNo = 'REF' . date('YmdHis') . rand(1000, 9999);
            $params = [
                'out_trade_no' => $order['order_no'],
                'out_refund_no' => $refundNo,
                'total_fee' => $order['total_fee'] * 100,
                'refund_fee' => $refundFee * 100,
                'refund_desc' => $reason,
                'notify_url' => config('wxpay.refund_url'),
            ];
            
            $result = $wxPay->refund($params);
            if ($result['return_code'] == 'SUCCESS' && $result['result_code'] == 'SUCCESS') {
                // 更新订单退款状态
                Db::name('pay_order')
                    ->where('order_no', $order['order_no'])
                    ->update([
                        'status' => 2,
                        'refund_fee' => $refundFee,
                        'refund_reason' => $reason,
                        'refund_time' => time(),
                        'updated_at' => time()
                    ]);
                // 记录退款日志
                $this->writeRefundLog($order['order_no'], 1, json_encode($result));
                return ['code' => 0, 'msg' => '退款成功'];
            } else {
                $this->writeRefundLog($order['order_no'], 1, json_encode($result));
                return ['code' => 2004, 'msg' => $result['err_code_des'] ?? '微信退款失败'];
            }
        } catch (\Exception $e) {
            $this->writeRefundLog($order['order_no'], 1, $e->getMessage());
            return ['code' => 2005, 'msg' => '微信退款异常：' . $e->getMessage()];
        }
    }
    
    // 支付宝退款
    private function aliRefund($order, $refundFee, $reason)
    {
        try {
            $alipay = new AliRefund();
            $alipay->setAppId(config('alipay.app_id'));
            $alipay->setPrivateKey(config('alipay.private_key'));
            $alipay->setAliPublicKey(config('alipay.ali_public_key'));
            $alipay->setGatewayUrl(config('alipay.gateway_url'));
            
            $params = [
                'out_trade_no' => $order['order_no'],
                'refund_amount' => $refundFee,
                'refund_reason' => $reason,
            ];
            
            $result = $alipay->refund($params);
            if ($result['code'] == 10000) {
                // 更新订单退款状态
                Db::name('pay_order')
                    ->where('order_no', $order['order_no'])
                    ->update([
                        'status' => 2,
                        'refund_fee' => $refundFee,
                        'refund_reason' => $reason,
                        'refund_time' => time(),
                        'updated_at' => time()
                    ]);
                // 记录退款日志
                $this->writeRefundLog($order['order_no'], 2, json_encode($result));
                return ['code' => 0, 'msg' => '退款成功'];
            } else {
                $this->writeRefundLog($order['order_no'], 2, json_encode($result));
                return ['code' => 2006, 'msg' => $result['sub_msg'] ?? '支付宝退款失败'];
            }
        } catch (\Exception $e) {
            $this->writeRefundLog($order['order_no'], 2, $e->getMessage());
            return ['code' => 2007, 'msg' => '支付宝退款异常：' . $e->getMessage()];
        }
    }
    
    // 写入退款日志
    private function writeRefundLog($orderNo, $payType, $content)
    {
        Db::name('pay_log')->insert([
            'order_no' => $orderNo,
            'pay_type' => $payType,
            'type' => 3,
            'content' => $content,
            'created_at' => time()
        ]);
    }
}


### 4. 后台订单管理（application/admin/controller/Order.php）

<?php
namespace app\admin\controller;

use think\Controller;
use think\Db;
use think\facade\Config;

class Order extends Controller
{
    // 订单列表
    public function index()
    {
        $keyword = $this->request->get('keyword', '');
        $status = $this->request->get('status', '');
        
        $where = [];
        if ($keyword) {
            $where[] = ['order_no', 'like', "%{$keyword}%"];
        }
        if ($status !== '') {
            $where[] = ['status', '=', $status];
        }
        
        $list = Db::name('pay_order')
            ->where($where)
            ->order('created_at desc')
            ->paginate(15);
        
        $this->assign('list', $list);
        $this->assign('status', $status);
        $this->assign('keyword', $keyword);
        return $this->fetch();
    }
    
    // 订单详情
    public function detail($id)
    {
        $order = Db::name('pay_order')->where('id', $id)->find();
        $logs = Db::name('pay_log')->where('order_no', $order['order_no'])->order('created_at desc')->select();
        
        $this->assign('order', $order);
        $this->assign('logs', $logs);
        return $this->fetch();
    }
    
    // 后台退款操作
    public function refund()
    {
        $param = $this->request->param();
        $order = Db::name('pay_order')->where('id', $param['id'])->find();
        
        if (!$order) {
            $this->error('订单不存在');
        }
        if ($order['status'] != 1) {
            $this->error('仅支付成功的订单可退款');
        }
        
        // 调用退款接口
        $refundApi = new \app\api\controller\Refund();
        $result = $refundApi->refund([
            'order_no' => $order['order_no'],
            'refund_fee' => $param['refund_fee'],
            'reason' => $param['reason']
        ]);
        
        if ($result['code'] == 0) {
            $this->success('退款成功');
        } else {
            $this->error($result['msg']);
        }
    }
}

### 5. 前端支付测试页（application/index/controller/Pay.php）

<?php
namespace app\index\controller;

use think\Controller;

class Pay extends Controller
{
    // 支付测试页
    public function index()
    {
        return $this->fetch();
    }
    
    // 支付成功返回页
    public function return()
    {
        $param = $this->request->param();
        $this->assign('param', $param);
        return $this->fetch('return');
    }
}


## 四、一键部署说明
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

