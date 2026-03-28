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
FROM httpd:2.4.49

# 1. Bật tính năng Rewrite và Proxy để gọi API xuống Backend
RUN sed -i '/LoadModule rewrite_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i '/LoadModule proxy_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i '/LoadModule proxy_http_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i 's/AllowOverride None/AllowOverride All/g' /usr/local/apache2/conf/httpd.conf

# 2. Cấu hình chuyển hướng mọi request "/api" xuống Backend
RUN echo "ProxyPass /api http://backend:3001/api" >> /usr/local/apache2/conf/httpd.conf && \
    echo "ProxyPassReverse /api http://backend:3001/api" >> /usr/local/apache2/conf/httpd.conf && \
    echo "ProxyPass /images http://backend:3001/images" >> /usr/local/apache2/conf/httpd.conf && \
    echo "ProxyPassReverse /images http://backend:3001/images" >> /usr/local/apache2/conf/httpd.conf

# 3. Xóa code web mặc định của Apache và Copy code dự án vào
RUN rm -rf /usr/local/apache2/htdocs/*
COPY --from=builder /app/dist/client /usr/local/apache2/htdocs/

# 4. Cấu hình để web React không bị lỗi trắng trang (404) khi F5
RUN echo 'FallbackResource /index.html' > /usr/local/apache2/htdocs/.htaccess

EXPOSE 80