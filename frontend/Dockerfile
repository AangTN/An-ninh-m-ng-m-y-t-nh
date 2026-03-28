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
# ==========================================
# PHẦN 2: APACHE 2.4.49 - PHIÊN BẢN CÓ LỖ HỔNG
# ==========================================
FROM httpd:2.4.49

# 1. Bật cgid_module (BẮT BUỘC để demo cả RCE - chiếm quyền điều khiển)
RUN sed -i '/LoadModule ssl_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i '/LoadModule socache_shmcb_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i '/LoadModule rewrite_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i '/LoadModule proxy_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i '/LoadModule proxy_http_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i '/LoadModule cgid_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i 's/AllowOverride None/AllowOverride All/g' /usr/local/apache2/conf/httpd.conf

# 2. Cấu hình VirtualHost "Tự sát"
RUN printf 'Listen 443\n\
\n\
<VirtualHost *:80>\n\
    ServerName secretpizza.io.vn\n\
    Redirect permanent / https://secretpizza.io.vn/\n\
</VirtualHost>\n\
\n\
<VirtualHost *:443>\n\
    ServerName secretpizza.io.vn\n\
    DocumentRoot "/usr/local/apache2/htdocs"\n\
\n\
    SSLEngine on\n\
    SSLCertificateFile "/usr/local/apache2/conf/ssl/server.crt"\n\
    SSLCertificateKeyFile "/usr/local/apache2/conf/ssl/server.key"\n\
\n\
    # --- ĐIỂM YẾU 1: Cho phép truy cập vào thư mục gốc của hệ thống ---\n\
    <Directory />\n\
        AllowOverride None\n\
        Require all granted\n\
    </Directory>\n\
\n\
    # --- ĐIỂM YẾU 2: ScriptAlias làm bàn đạp để leo trèo ---\n\
    ScriptAlias /cgi-bin/ "/usr/local/apache2/cgi-bin/"\n\
\n\
    <Directory "/usr/local/apache2/htdocs">\n\
        AllowOverride All\n\
        Require all granted\n\
    </Directory>\n\
</VirtualHost>\n' > /usr/local/apache2/conf/extra/httpd-vhosts.conf && \
echo "Include conf/extra/httpd-vhosts.conf" >> /usr/local/apache2/conf/httpd.conf

# 3. Copy code web
RUN rm -rf /usr/local/apache2/htdocs/*
COPY --from=builder /app/dist/client /usr/local/apache2/htdocs/

# --- LƯU Ý: TẠM THỜI XÓA FallbackResource ---
# Vì cái này sẽ "nuốt" mọi đường dẫn lạ và trả về index.html, 
# khiến ta không xem được file /etc/passwd khi demo.
RUN touch /usr/local/apache2/htdocs/.htaccess

EXPOSE 80 443