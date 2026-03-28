# Phần 1: Build source code (Giữ nguyên của bạn)
FROM node:22-bookworm-slim AS builder

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .

ARG VITE_API_BASE_URL=http://localhost
ARG VITE_MAP_KEY=
ARG VITE_CLIENT_ID=

ENV VITE_API_BASE_URL=${VITE_API_BASE_URL}
ENV VITE_MAP_KEY=${VITE_MAP_KEY}
ENV VITE_CLIENT_ID=${VITE_CLIENT_ID}

RUN npm run build

# ==========================================
# Phần 2: Đưa code lên Web Server Apache bị lỗi (Thay cho Nginx)
# ==========================================
# ... (Giữ nguyên Phần 1: Build source code của bạn) ...

FROM httpd:2.4.49

# 1. Bật các module cần thiết cho SSL và Proxy
RUN sed -i '/LoadModule ssl_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i '/LoadModule socache_shmcb_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i '/LoadModule rewrite_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i '/LoadModule proxy_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i '/LoadModule proxy_http_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i 's/AllowOverride None/AllowOverride All/g' /usr/local/apache2/conf/httpd.conf

# 2. Tạo file cấu hình SSL VirtualHost trực tiếp
# Lưu ý: Thay "yourdomain.com" bằng tên miền thật của bạn
RUN echo 'Listen 443\n\
<VirtualHost *:443>\n\
    DocumentRoot "/usr/local/apache2/htdocs"\n\
    SSLEngine on\n\
    SSLCertificateFile "/usr/local/apache2/conf/ssl/server.crt"\n\
    SSLCertificateKeyFile "/usr/local/apache2/conf/ssl/server.key"\n\
    \n\
    ProxyPass /api http://backend:3001/api\n\
    ProxyPassReverse /api http://backend:3001/api\n\
    ProxyPass /images http://backend:3001/images\n\
    ProxyPassReverse /images http://backend:3001/images\n\
    \n\
    <Directory "/usr/local/apache2/htdocs">\n\
        AllowOverride All\n\
        Require all granted\n\
    </Directory>\n\
</VirtualHost>' >> /usr/local/apache2/conf/httpd.conf

# 3. Giữ nguyên cấu hình cũ (Proxy cổng 80, Copy code, .htaccess)
RUN echo "ProxyPass /api http://backend:3001/api" >> /usr/local/apache2/conf/httpd.conf && \
    echo "ProxyPassReverse /api http://backend:3001/api" >> /usr/local/apache2/conf/httpd.conf && \
    echo "ProxyPass /images http://backend:3001/images" >> /usr/local/apache2/conf/httpd.conf && \
    echo "ProxyPassReverse /images http://backend:3001/images" >> /usr/local/apache2/conf/httpd.conf

RUN rm -rf /usr/local/apache2/htdocs/*
COPY --from=builder /app/dist/client /usr/local/apache2/htdocs/
RUN echo 'FallbackResource /index.html' > /usr/local/apache2/htdocs/.htaccess

EXPOSE 80 443