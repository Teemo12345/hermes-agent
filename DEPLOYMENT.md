# Linux 部署指南

本指南详细说明如何在 Linux 服务器上部署 Hermes Agent 文档网站。

## 1. 构建项目

首先在本地或构建服务器上构建项目：

```bash
# 进入 website 目录
cd website

# 安装依赖
npm install

# 构建项目
npm run build

# 构建结果会生成在 build/ 目录中
```

## 2. 上传构建文件

将构建结果上传到 Linux 服务器：npm 

```bash
# 使用 scp 上传
scp -r build/* user@your-server:/var/www/hermes-zh-doc/

# 或使用 rsync
rsync -avz build/ user@your-server:/var/www/hermes-zh-doc/
```

## 3. 配置 Web 服务器

### 使用 Nginx

1. **安装 Nginx**：
   ```bash
   sudo apt update
sudo apt install nginx
   ```

2. **创建站点配置**：
   ```bash
   sudo nano /etc/nginx/sites-available/hermes-zh-doc
   ```

3. **添加配置**：
   ```nginx
   server {
       listen 80;
       server_name your-domain.com;

       root /var/www/hermes-zh-doc;
       index index.html;

       location / {
           try_files $uri $uri/ /index.html;
       }

       # 处理多语言路由
       location /zh-Hans/ {
           try_files $uri $uri/ /zh-Hans/index.html;
       }
   }
   ```

4. **启用站点**：
   ```bash
   sudo ln -s /etc/nginx/sites-available/hermes-zh-doc /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
   ```

### 使用 Apache

1. **安装 Apache**：
   ```bash
   sudo apt update
sudo apt install apache2
   ```

2. **创建站点配置**：
   ```bash
   sudo nano /etc/apache2/sites-available/hermes-zh-doc.conf
   ```

3. **添加配置**：
   ```apache
   <VirtualHost *:80>
       ServerName your-domain.com
       DocumentRoot /var/www/hermes-zh-doc

       <Directory /var/www/hermes-zh-doc>
           Options FollowSymLinks
           AllowOverride All
           Require all granted
       </Directory>

       # 启用 rewrite 模块
       RewriteEngine On
       RewriteBase /
       RewriteRule ^index\.html$ - [L]
       RewriteCond %{REQUEST_FILENAME} !-f
       RewriteCond %{REQUEST_FILENAME} !-d
       RewriteRule . /index.html [L]
   </VirtualHost>
   ```

4. **启用站点**：
   ```bash
   sudo a2enmod rewrite
sudo a2ensite hermes-zh-doc.conf
sudo systemctl reload apache2
   ```

## 4. 配置 SSL（可选）

如果需要 HTTPS，可以使用 Let's Encrypt：

```bash
sudo apt install certbot python3-certbot-nginx

# 为 Nginx 配置 SSL
sudo certbot --nginx -d your-domain.com

# 为 Apache 配置 SSL
sudo certbot --apache -d your-domain.com
```

## 5. 验证部署

1. **检查文件权限**：
   ```bash
   sudo chown -R www-data:www-data /var/www/hermes-zh-doc
sudo chmod -R 755 /var/www/hermes-zh-doc
   ```

2. **测试访问**：
   - 访问 `http://your-domain.com` 查看英文版本
   - 访问 `http://your-domain.com/zh-Hans/` 查看中文版本

3. **检查多语言切换**：
   - 测试语言下拉菜单是否正常工作
   - 确认页面内容正确切换语言

## 6. 自动部署（可选）

可以设置 CI/CD 流水线自动部署：

1. **创建部署脚本**：
   ```bash
   #!/bin/bash
   
   # 构建项目
   cd website
   npm install
   npm run build
   
   # 上传到服务器
   rsync -avz build/ user@your-server:/var/www/hermes-zh-doc/
   
   # 重启 Web 服务器
   ssh user@your-server "sudo systemctl reload nginx"
   ```

2. **设置 GitHub Actions**：
   ```yaml
   # .github/workflows/deploy.yml
   name: Deploy Documentation
   
   on:
     push:
       branches: [ main ]
   
   jobs:
     deploy:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v3
         - name: Set up Node.js
           uses: actions/setup-node@v3
           with:
             node-version: '20'
         - name: Install dependencies
           run: cd website && npm install
         - name: Build documentation
           run: cd website && npm run build
         - name: Deploy to server
           uses: easingthemes/ssh-deploy@v2.1.5
           with:
             SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
             SOURCE: "website/build/"
             REMOTE_HOST: ${{ secrets.REMOTE_HOST }}
             REMOTE_USER: ${{ secrets.REMOTE_USER }}
             TARGET: "/var/www/hermes-zh-doc/"
   ```

## 7. 故障排除

### 常见问题

1. **404 错误**：
   - 检查文件路径是否正确
   - 确保 Web 服务器配置了正确的 root 目录
   - 验证 rewrite 规则是否生效

2. **多语言切换问题**：
   - 检查 i18n 配置是否正确
   - 确认构建时包含了所有语言版本
   - 验证语言文件是否正确生成

3. **搜索功能不工作**：
   - 确认 search-local 插件配置正确
   - 检查构建时是否生成了搜索索引

4. **Mermaid 图表不显示**：
   - 确认 theme-mermaid 插件正确安装
   - 检查浏览器控制台是否有错误

### 日志检查

```bash
# Nginx 日志
sudo tail -f /var/log/nginx/error.log

# Apache 日志
sudo tail -f /var/log/apache2/error.log
```

## 8. 性能优化

1. **启用 GZIP 压缩**：
   - Nginx：在 server 块中添加
     ```nginx
     gzip on;
     gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
     ```
   - Apache：启用 mod_deflate
     ```bash
     sudo a2enmod deflate
sudo systemctl reload apache2
     ```

2. **配置缓存**：
   - Nginx：添加缓存头
     ```nginx
     location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
         expires 30d;
         add_header Cache-Control "public, max-age=2592000";
     }
     ```

3. **使用 CDN**（可选）：
   - 配置 Cloudflare 或其他 CDN 加速静态资源

## 9. 维护

1. **定期更新**：
   ```bash
   # 更新依赖
   cd website
   npm update
   
   # 重新构建
   npm run build
   
   # 重新部署
   rsync -avz build/ user@your-server:/var/www/hermes-zh-doc/
   ```

2. **备份**：
   ```bash
   # 备份构建文件
   tar -czf hermes-zh-doc-backup-$(date +%Y%m%d).tar.gz /var/www/hermes-zh-doc/
   ```

3. **监控**：
   - 设置监控工具（如 Monit、Prometheus）监控服务状态
   - 配置日志监控（如 ELK Stack）跟踪访问和错误

---

按照以上步骤，您可以成功在 Linux 服务器上部署 Hermes Agent 文档网站，并确保多语言支持和搜索功能正常工作。