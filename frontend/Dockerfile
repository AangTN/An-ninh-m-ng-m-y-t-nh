# Phần 1: Build source code (Giữ nguyên của bạn)
FROM node:22-bookworm-slim AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
ARG VITE_API_BASE_URL
ARG VITE_MAP_KEY
ARG VITE_CLIENT_ID
ENV VITE_API_BASE_URL=${VITE_API_BASE_URL}
ENV VITE_MAP_KEY=${VITE_MAP_KEY}
ENV VITE_CLIENT_ID=${VITE_CLIENT_ID}
RUN npm run build

# ==========================================
# Phần 2: Apache Server - Chỉ nhận HTTPS qua Domain
# ==========================================
FROM httpd:2.4.49

# 1. Bật các module: SSL, Rewrite, Proxy và Headers (để bảo mật)
RUN sed -i '/LoadModule ssl_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i '/LoadModule socache_shmcb_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i '/LoadModule rewrite_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i '/LoadModule proxy_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i '/LoadModule proxy_http_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i '/LoadModule headers_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i 's/AllowOverride None/AllowOverride All/g' /usr/local/apache2/conf/httpd.conf

# 2. Xóa cấu hình cũ và ghi đè cấu hình VirtualHost nghiêm ngặt
# Cấu hình này sẽ:
# - Cổng 80: Chuyển hướng 100% sang HTTPS secretpizza.io.vn
# - Cổng 443: Nếu không phải tên miền secretpizza.io.vn -> Trả về lỗi 403 (Chặn IP)
RUN printf 'Listen 443\n\
\n\
# --- CẤU HÌNH CỔNG 80: REDIRECT SANG HTTPS ---\n\
<VirtualHost *:80>\n\
    ServerName secretpizza.io.vn\n\
    Redirect permanent / https://secretpizza.io.vn/\n\
</VirtualHost>\n\
\n\
# --- CẤU HÌNH CỔNG 443: CHỈ NHẬN DOMAIN, CHẶN IP ---\n\
<VirtualHost *:443>\n\
    ServerName secretpizza.io.vn\n\
    DocumentRoot "/usr/local/apache2/htdocs"\n\
\n\
    SSLEngine on\n\
    SSLCertificateFile "/usr/local/apache2/conf/ssl/server.crt"\n\
    SSLCertificateKeyFile "/usr/local/apache2/conf/ssl/server.key"\n\
\n\
    # Chống truy cập bằng IP: Nếu Host không phải là domain thì trả về 403\n\
    RewriteEngine On\n\
    RewriteCond %%{HTTP_HOST} !^secretpizza\\.io\\.vn$ [NC]\n\
    RewriteRule ^ - [F]\n\
\n\
    # Ép trình duyệt luôn dùng HTTPS (HSTS)\n\
    Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"\n\
\n\
    # Proxy xuống Backend\n\
    ProxyPass /api http://backend:3001/api\n\
    ProxyPassReverse /api http://backend:3001/api\n\
    ProxyPass /images http://backend:3001/images\n\
    ProxyPassReverse /images http://backend:3001/images\n\
\n\
    <Directory "/usr/local/apache2/htdocs">\n\
        AllowOverride All\n\
        Require all granted\n\
    </Directory>\n\
</VirtualHost>\n' > /usr/local/apache2/conf/extra/httpd-vhosts.conf && \
echo "Include conf/extra/httpd-vhosts.conf" >> /usr/local/apache2/conf/httpd.conf

# 3. Copy code web vào
RUN rm -rf /usr/local/apache2/htdocs/*
COPY --from=builder /app/dist/client /usr/local/apache2/htdocs/

# 4. Cấu hình .htaccess để tránh lỗi 404 khi F5 React
RUN echo 'FallbackResource /index.html' > /usr/local/apache2/htdocs/.htaccess

EXPOSE 80 443