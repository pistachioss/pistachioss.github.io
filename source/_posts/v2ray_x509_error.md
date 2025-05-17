---
title: 509 certificate signed by unknown authority
date: 2022-03-05
categories: 
    - v2ray
    - cloudflare
tags: wiki
toc: true
description: "Cloudflare v2ray一直 509 certificate signed by unknown authority"

---

“x509: certificate signed by unknown authority” 错误表明 Nginx 或 V2Ray 客户端在验证 Cloudflare 源服务器证书（Origin Server Certificate）时，无法信任该证书的颁发机构（CA）。这通常是因为证书配置不正确、Cloudflare 的 SSL/TLS 设置有误，或者缺少必要的根证书/中间证书。本回答将针对你的 Cloudflare + V2Ray 配置（基于 WebSocket + TLS），提供详细的排查和解决步骤，帮助你修复这个错误。


### **错误原因分析**
在 Cloudflare + V2Ray 的场景中，“certificate signed by unknown authority” 错误可能由以下原因引起：
1. **源服务器证书未正确配置**：Cloudflare 源服务器证书未正确安装到 Nginx，或文件内容有误。
2. **SSL/TLS 模式问题**：Cloudflare 的 SSL/TLS 模式设置为 **Full (strict)**，但 VPS 未使用 Cloudflare 签发的证书，或者证书链不完整。
3. **缺少 Cloudflare 根证书**：如果启用了 **Authenticated Origin Pulls**，Nginx 需要 Cloudflare 的根证书来验证边缘服务器的客户端证书。
4. **客户端配置错误**：V2Ray 客户端的 TLS 设置（如 SNI）与服务器证书不匹配。
5. **证书和私钥不匹配**：Nginx 使用的证书和私钥不对应，导致 TLS 握手失败。

---

### **修复步骤**
以下是针对 Cloudflare + V2Ray 配置的逐步排查和解决方法，假设你的 VPS 使用 Nginx 和 V2Ray，域名是 `v2ray.example.com`。

#### **步骤 1：检查 Nginx 错误日志**
1. 查看 Nginx 错误日志，获取详细错误信息：
   ```bash
   cat /var/log/nginx/error.log
   ```
2. 查找类似以下的错误：
   - `x509: certificate signed by unknown authority`
   - `SSL: error:0B080074:x509 certificate routines`
3. 记录具体错误上下文（例如错误发生在 TLS 握手还是客户端验证阶段）。

#### **步骤 2：验证 Cloudflare 源服务器证书**
你提到之前只看到需要复制粘贴的证书内容（例如 `example.com.pem`），可能未正确保存证书和私钥，或者证书格式有误。让我们重新生成并验证：

1. **重新生成源服务器证书**：
   - 登录 Cloudflare，进入 **SSL/TLS > Origin Server**。
   - 点击 **Create Certificate**：
     - **Key Type**：选择 **ECDSA**（推荐）或 **RSA**。
     - **Certificate Validity**：默认 15 年。
     - **Hostnames**：输入 `v2ray.example.com` 或 `*.example.com`。
   - 点击 **Create**，页面会显示：
     - **Origin Certificate**：PEM 格式，包含 `-----BEGIN CERTIFICATE-----` 和 `-----END CERTIFICATE-----`。
     - **Private Key**：PEM 格式，包含 `-----BEGIN PRIVATE KEY-----` 和 `-----END PRIVATE KEY-----`（可能需点击“Show”）。
   - **保存证书**：
     - 复制 **Origin Certificate** 文本，保存到 VPS：
       ```bash
       mkdir -p /etc/ssl
       nano /etc/ssl/v2ray_example_com_cert.pem
       ```
     - 粘贴证书内容，确保包括 `-----BEGIN CERTIFICATE-----` 和 `-----END CERTIFICATE-----`，保存。
   - **保存私钥**：
     - 复制 **Private Key** 文本，保存到：
       ```bash
       nano /etc/ssl/v2ray_example_com_key.pem
       ```
     - 粘贴私钥内容，确保包括 `-----BEGIN PRIVATE KEY-----` 和 `-----END PRIVATE KEY-----`，保存。

2. **检查文件格式**：
   - 确保证书文件内容正确：
     ```bash
     cat /etc/ssl/v2ray_example_com_cert.pem
     ```
     应显示：
     ```
     -----BEGIN CERTIFICATE-----
     MIID... (Base64 编码)
     -----END CERTIFICATE-----
     ```
   - 检查私钥：
     ```bash
     cat /etc/ssl/v2ray_example_com_key.pem
     ```
     应显示：
     ```
     -----BEGIN PRIVATE KEY-----
     MIIE... (Base64 编码)
     -----END PRIVATE KEY-----
     ```
   - **常见问题**：
     - 缺少 `-----BEGIN/END` 标记。
     - 多余的空行、空格或注释。
     - 证书和私钥合并在一个文件。

3. **验证证书和私钥匹配**：
   - 检查证书和私钥是否匹配：
     ```bash
     openssl x509 -noout -modulus -in /etc/ssl/v2ray_example_com_cert.pem | openssl md5
     openssl rsa -noout -modulus -in /etc/ssl/v2ray_example_com_key.pem | openssl md5
     ```
     - 输出应相同，否则说明证书和私钥不匹配，需重新生成。

4. **设置文件权限**：
   ```bash
   chmod 600 /etc/ssl/v2ray_example_com_*.pem
   chown root:root /etc/ssl/v2ray_example_com_*.pem
   ```

#### **步骤 3：检查 Nginx 配置**
1. **验证 Nginx 配置文件**：
   - 编辑配置文件（例如 `/etc/nginx/sites-available/v2ray`）：
     ```bash
     nano /etc/nginx/sites-available/v2ray
     ```
   - 确保内容正确：
     ```nginx
     server {
         listen 80;
         server_name v2ray.example.com;
         return 301 https://$server_name$request_uri;
     }
     server {
         listen 443 ssl;
         server_name v2ray.example.com;
         ssl_certificate /etc/ssl/v2ray_example_com_cert.pem;
         ssl_certificate_key /etc/ssl/v2ray_example_com_key.pem;
         ssl_protocols TLSv1.2 TLSv1.3;
         ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
         ssl_prefer_server_ciphers off;
     
         location /ray {
             if ($http_upgrade != "websocket") {
                 return 404;
             }
             proxy_redirect off;
             proxy_pass http://127.0.0.1:12345;
             proxy_http_version 1.1;
             proxy_set_header Upgrade $http_upgrade;
             proxy_set_header Connection "upgrade";
             proxy_set_header Host $host;
         }
     }
     ```
   - **关键检查**：
     - `ssl_certificate` 和 `ssl_certificate_key` 路径正确。
     - 没有重复的 `ssl_certificate` 指令。
     - 确保 `server_name` 与你的域名一致。

2. **测试配置**：
   ```bash
   nginx -t
   systemctl restart nginx
   ```
   - 如果报错（例如 `cannot load certificate`），检查文件路径或内容。

#### **步骤 4：检查 Cloudflare SSL/TLS 设置**
1. **确认 SSL/TLS 模式**：
   - 在 Cloudflare 的 **SSL/TLS > Overview**，确保模式为 **Full** 或 **Full (strict)**。
   - **Full (strict)** 要求 VPS 使用 Cloudflare 签发的源服务器证书（你已配置）。
   - 如果设置为 **Flexible**，可能会导致证书验证问题，需改为 **Full**。

2. **确认 DNS 设置**：
   - 在 **DNS** 页面，检查 `v2ray.example.com` 的 A 记录：
     - 指向 VPS IP。
     - **Proxy status** 为 **Proxied**（橙色云）。

3. **检查 Authenticated Origin Pulls**：
   - 如果启用了 **Authenticated Origin Pulls**，可能导致 “unknown authority” 错误，因为 Nginx 需要验证 Cloudflare 边缘服务器的客户端证书。
   - **临时禁用**：
     - 在 **SSL/TLS > Overview**，关闭 **Authenticated Origin Pulls**。
     - 重启 Nginx 和 V2Ray，测试是否解决问题。
   - **正确配置（如果需要启用）**：
     - 下载 Cloudflare 根证书（ECC 或 RSA）：
       - 从 https://developers.cloudflare.com/ssl/origin-configuration/authenticated-origin-pull/set-up/ 下载，保存为 `/etc/ssl/cloudflare_origin_ecc.pem`。
     - 更新 Nginx 配置：
       ```nginx
       ssl_client_certificate /etc/ssl/cloudflare_origin_ecc.pem;
       ssl_verify_client on;
       ```
     - 重启 Nginx：
       ```bash
       systemctl restart nginx
       ```

#### **步骤 5：检查 V2Ray 配置**
1. **验证 V2Ray 配置文件**：
   - 编辑 `/usr/local/etc/v2ray/config.json`：
     ```bash
     nano /usr/local/etc/v2ray/config.json
     ```
   - 确保配置正确（替换 `your-uuid` 和路径）：
     ```json
     {
         "log": {
             "loglevel": "warning"
         },
         "inbounds": [
             {
                 "port": 12345,
                 "listen": "127.0.0.1",
                 "protocol": "vless",
                 "settings": {
                     "clients": [
                         {
                             "id": "your-uuid",
                             "email": "user@example.com"
                         }
                     ],
                     "decryption": "none"
                 },
                 "streamSettings": {
                     "network": "ws",
                     "wsSettings": {
                         "path": "/ray"
                     }
                 }
             }
         ],
         "outbounds": [
             {
                 "protocol": "freedom"
             }
         ]
     }
     ```
   - 重启 V2Ray：
     ```bash
     systemctl restart v2ray
     ```

2. **检查客户端配置**：
   - 确保 V2Ray 客户端（例如 v2rayN、v2rayNG）配置正确：
     ```
     vless://your-uuid@v2ray.example.com:443?encryption=none&security=tls&type=ws&path=/ray&sni=v2ray.example.com#V2Ray-WS-TLS
     ```
   - **关键点**：
     - **SNI**：必须设置为 `v2ray.example.com`，与证书的域名匹配。
     - **TLS**：启用。
     - **Path**：`/ray`，与服务器一致。

#### **步骤 6：测试 TLS 连接**
1. **服务器端测试**：
   ```bash
   openssl s_client -connect v2ray.example.com:443 -servername v2ray.example.com
   ```
   - 检查输出：
     - 如果显示证书链且无 `verify error`，证书配置正确。
     - 如果报 `unknown CA`，说明证书链不完整或 Cloudflare 未信任。

2. **客户端测试**：
   - 使用 V2Ray 客户端连接，检查是否仍报 “unknown authority”。
   - 如果客户端报错，尝试临时禁用 TLS 验证（仅用于调试，不推荐生产环境）：
     - 在 v2rayN 或 v2rayNG 中，设置 “Allow Insecure” 为 true。
     - 如果连接成功，说明问题出在证书信任链。

#### **步骤 7：修复 “unknown authority” 错误**
根据排查结果，针对具体原因修复：

1. **证书文件错误**：
   - 如果证书或私钥内容不完整，重新生成并保存（步骤 2）。
   - 使用 `openssl` 验证证书：
     ```bash
     openssl x509 -in /etc/ssl/v2ray_example_com_cert.pem -text -noout
     ```
     - 如果报错，说明证书文件损坏，需重新复制。

2. **证书和私钥不匹配**：
   - 重新生成证书，确保使用同一组证书和私钥。
   - 验证匹配（步骤 2.3）。

3. **Authenticated Origin Pulls 导致问题**：
   - 如果启用了此功能但未配置根证书，Nginx 会报 “unknown authority”。
   - 按照步骤 4.3 正确配置，或临时禁用。

4. **Cloudflare SSL/TLS 模式**：
   - 确保模式为 **Full (strict)**，并使用 Cloudflare 源服务器证书。
   - 如果使用其他证书（如 Let’s Encrypt），需将模式设为 **Full**。

5. **客户端 SNI 错误**：
   - 确保 V2Ray 客户端的 SNI 与证书的域名（`v2ray.example.com`）一致。
   - 检查证书的 Subject 和 SAN：
     ```bash
     openssl x509 -in /etc/ssl/v2ray_example_com_cert.pem -noout -text
     ```
     - 确认 `Subject` 或 `Subject Alternative Name` 包含 `v2ray.example.com`。

---

### **常见问题与解答**
- **Q：为什么总是报 “unknown authority”？**
  - 可能是证书未由 Cloudflare 的 CA 签发，或 Nginx 未正确加载证书。重新生成并验证文件内容。
- **Q：Authenticated Origin Pulls 必须启用吗？**
  - 不必须。建议先解决证书问题，确认 V2Ray 正常工作后再启用以增强安全性。
- **Q：可以用 Let’s Encrypt 证书吗？**
  - 可以，但需将 Cloudflare 的 SSL/TLS 模式设为 **Full**（非 strict），否则 Cloudflare 要求自己的源服务器证书。
- **Q：客户端报错如何调试？**
  - 临时启用 “Allow Insecure” 测试连接。
  - 检查客户端日志，确认 SNI 和 TLS 设置。

---

### **总结**
- **“x509: certificate signed by unknown authority”** 通常由证书配置错误、SSL/TLS 模式不匹配或 Authenticated Origin Pulls 配置不当引起。
- **修复步骤**：
  1. 重新生成 Cloudflare 源服务器证书，正确保存证书和私钥。
  2. 验证 Nginx 配置，确保证书路径和格式正确。
  3. 检查 Cloudflare 的 SSL/TLS 模式，临时禁用 Authenticated Origin Pulls。
  4. 确保 V2Ray 客户端的 SNI 和 TLS 设置正确。
  5. 测试 TLS 连接，必要时启用 Authenticated Origin Pulls。
- **建议**：
  - 检查 Nginx 错误日志（`/var/log/nginx/error.log`）以获取详细信息。
  - 如果仍报错，请提供日志或具体错误描述，我可以进一步分析。

如果你有错误日志、Cloudflare 界面截图，或其他问题（例如 V2Ray 客户端的具体错误），请分享，我会为你提供更精准的解决方案！