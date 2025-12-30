# 老王vps脚本下的f老sba环境下配置webdav
折腾日记webdav借sba脚本的nginx和cf隧道访问。


## 0. 前置准备

#### 教程默认nginx端口为10660，webdav服务端口22331。
 0.1. 在cf平台Zero Trust-网络-连接器 创建对应域名和端口。  
 
例如：
- nginx的是:   xxx.yourdomain.com http://localhost:10660  
- webdav的: dav.yourdamain.com http://localhost:22331 

  类型选*http*。端口要写*localhost:10660*。webdav的也一样。
  记得把令牌token记下来。
  如果你改了端口，后面写nginx.conf也要改。

	
0.2. 在老王脚本安装完 `sing-box` 和 `Nginx` 后，首先升级 Nginx 以支持 WebDAV 高级扩展。

   ```bash
   apt update && apt install nginx-extras -y

   ```

## 1. 建立空中走廊（Cloudflare 隧道）

利用老王脚本现有的二进制文件创建专属 WebDAV 隧道服务。

### 执行 EOF 脚本创建并启动服务

```bash
cat > /etc/systemd/system/webdav-tunnel.service <<EOF
[Unit]
Description=Cloudflare Tunnel for WebDAV
After=network.target

[Service]
ExecStart=/etc/sing-box/cloudflared tunnel run --token 你的_CLOUDFLARE_TOKEN
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF

# 立即加载并启动服务
systemctl daemon-reload
systemctl enable webdav-tunnel
systemctl start webdav-tunnel

```

### 排查命令

* **查看隧道状态**：`systemctl status webdav-tunnel`（应显示 active/running）
* **查看隧道日志**：`journalctl -u webdav-tunnel -f`

---

## 2. 安全与权限配置

创建验证用户并准备存储目录。

### 创建密码与目录

```bash
# 安装工具（如未安装）
apt install apache2-utils -y

# 创建 WebDAV 账号（将 username 换成你的名字）
htpasswd -c /etc/sing-box/.webdav_passwd username

# 创建并授权目录
mkdir -p /var/www/dav
chown -R root:root /var/www/dav

```

### 排查命令

* **确认密码文件**：`cat /etc/sing-box/.webdav_passwd`
* **确认目录权限**：`ls -ld /var/www/dav`

---

## 3. 编写大统一 Nginx 配置文件

修改 `/etc/sing-box/nginx.conf`，将以下内容原样覆盖。

```nginx
# 1. 加载 WebDAV 扩展模块
load_module /usr/lib/nginx/modules/ngx_http_dav_ext_module.so;

user root;
worker_processes auto;
error_log /etc/sing-box/nginx_error.log crit; # 只记录严重错误
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    # 保持老王的分流逻辑
    map $http_user_agent $path1 {
        default /;
        ~*v2rayN /v2rayn;
        ~*clash /clash;
        ~*Neko|Throne /neko;
        ~*ShadowRocket /shadowrocket;
        ~*SFM /sing-box-pc;
        ~*SFI|SFA /sing-box-phone;
    }

    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    sendfile on;
    keepalive_timeout 65;

    # 关键：解除全局上传大小限制（解决 Kazumi 报错）
    client_max_body_size 0;
    proxy_request_buffering off;
    
    # 开启访问日志以监控 Kazumi 动态
    access_log /etc/sing-box/nginx_access.log;

    # --- Server 1: 老王的节点与订阅 (监听 10660) ---
    server {
        listen 10660;
        server_name _; 

        location /你的_VLESS_路径-vless {
            if ($http_upgrade != websocket) { return 404; }
            proxy_http_version 1.1;
            proxy_pass https://127.0.0.1:10601;
            proxy_ssl_protocols TLSv1.3; # 匹配老王脚本 TLS 版本
            proxy_ssl_verify off;       # 关键：跳过自签证书校验
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection upgrade;
            proxy_set_header Host $host;
        }

        # 订阅分流逻辑...
        location ~ ^/你的_路径/auto {
            default_type 'text/plain; charset=utf-8';
            alias /etc/sing-box/subscribe/$path1;
        }
    }

    # --- Server 2: WebDAV 修正层 (监听 22331) ---
    # 专门处理 Cloudflare 隧道流量并修正 Destination 请求头
    server {
        listen 22331 default_server;
        server_name _;

        location / {
            set $fixed_dest $http_destination;
            if ($http_destination ~* ^https?://[^/]+(/.*)$ ) {
                set $fixed_dest http://localhost:22332$1;
            }
            proxy_set_header Destination $fixed_dest;
            proxy_set_header Host localhost;
            proxy_pass http://127.0.0.1:22332;
        }
    }

    # --- Server 3: WebDAV 核心存储层 (监听 22332) ---
    server {
        listen 127.0.0.1:22332;
        location / {
            root /var/www/dav;
            dav_methods PUT DELETE MKCOL COPY MOVE;
            dav_ext_methods PROPFIND OPTIONS;
            create_full_put_path on;
            auth_basic "WebDAV Storage";
            auth_basic_user_file /etc/sing-box/.webdav_passwd;
            autoindex on;
            # 容错处理：让已存在的文件夹返回成功
            error_page 405 =201 /success_response;
        }
        location = /success_response { return 201; }
    }
}

```

---

## 4. 启动与终极排查

### 执行命令

```bash
# 测试语法
/usr/sbin/nginx -t -c /etc/sing-box/nginx.conf

# 强制重启 Nginx
pkill -9 nginx
/usr/sbin/nginx -c /etc/sing-box/nginx.conf

```

### 排查命令

1. **检查端口监听**：
`netstat -tulpn | grep nginx`
* 预期：应看到 `10660` (节点), `22331` (入口), `22332` (存储) 都在 LISTEN。


2. **实时监控 Kazumi 同步**：
`tail -f /etc/sing-box/nginx_access.log`
* 预期：看到 `PUT` 或 `MOVE` 返回 `201` 或 `204` 即代表同步成功。


3. **Cloudflare 配置检查**：
* 确保 Public Hostname 的 Service 指向 `http://localhost:22331`。



---

*恭喜！你现在拥有一套完美融合老王脚本、全能 WebDAV 和安全隧道的服务器系统。*
