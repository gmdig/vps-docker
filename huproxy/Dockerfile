#本地连接方式，可能需要重复：
#ssh root@localhost -o PubkeyAuthentication=no -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o 'ProxyCommand=./huproxyclient -insecure_conn wss://yourapp.railway.app/proxy/%h/%p'

FROM node:20-alpine

# 构建参数
ARG GO_VERSION=1.25.1
ARG PYTHON_VERSION=3.12

ENV GO_VERSION=$GO_VERSION

# 安装依赖：Python、构建工具、inotify、dropbear、额外包
RUN apk add --no-cache \
    python${PYTHON_VERSION%%.*} \
    py${PYTHON_VERSION%%.*}-pip \
    curl \
    ca-certificates \
    tar \
    inotify-tools \
    dropbear \
    shadow \
    gcc \
    musl-dev \
    sqlite-dev xz caddy

# 安装 Go
RUN curl -fsSLO "https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz" && \
    tar -C /usr/local -xzf "go${GO_VERSION}.linux-amd64.tar.gz" && \
    rm "go${GO_VERSION}.linux-amd64.tar.gz"

ENV PATH="/usr/local/go/bin:$PATH"

# 验证安装
RUN go version && python --version && node --version && npm --version

##############################
# 安装 s6-overlay (noarch + arch)
##############################
ARG S6_OVERLAY_VERSION=v3.2.1.0
ADD https://github.com/just-containers/s6-overlay/releases/download/${S6_OVERLAY_VERSION}/s6-overlay-noarch.tar.xz /tmp/
ADD https://github.com/just-containers/s6-overlay/releases/download/${S6_OVERLAY_VERSION}/s6-overlay-x86_64.tar.xz /tmp/
RUN tar -C / -Jxpf /tmp/s6-overlay-noarch.tar.xz && \
    tar -C / -Jxpf /tmp/s6-overlay-x86_64.tar.xz && \
    rm /tmp/s6-overlay-noarch.tar.xz /tmp/s6-overlay-x86_64.tar.xz


COPY ./huproxyserver /usr/local/bin/huproxy
RUN chmod +x /usr/local/bin/huproxy /usr/local/bin/huproxyclient && \
    dropbearkey -t rsa -f /etc/dropbear/dropbear_rsa_host_key

# ==============================
# 配置 caddy 服务（s6 管理）
# ==============================
RUN mkdir -p /etc/services.d/caddy
RUN printf '#!/bin/sh\n\
exec caddy run --config /etc/caddy/Caddyfile --adapter caddyfile\n' \
    > /etc/services.d/caddy/run && \
    chmod +x /etc/services.d/caddy/run

# 默认 Caddyfile（为 huproxy 使用 /proxy/*）
RUN mkdir -p /etc/caddy /var/www/html
RUN echo "Caddy works!" > /var/www/html/index.html
RUN cat > /etc/caddy/Caddyfile << 'EOF'
:8080 {
  root * /var/www/html

  @websockets {
    header Connection *Upgrade*
    header Upgrade websocket
  }

  reverse_proxy @websockets /proxy/* 127.0.0.1:8081

  file_server

  log {
    output stdout
  }
}
EOF

# ==============================
# 配置 dropbear 服务（支持 ROOT_PASSWORD 或自动生成随机密码）
# ==============================
RUN mkdir -p /etc/dropbear /etc/services.d/dropbear

RUN cat > /etc/services.d/dropbear/run << 'EOF'
#!/bin/sh
if [ -z "$ROOT_PASSWORD" ]; then
  RANDOM_PASS=$(tr -dc '0-9' < /dev/urandom | head -c 6)
  echo "root:$RANDOM_PASS" | chpasswd
  echo "========================================="
  echo "WARNING: ROOT_PASSWORD not set."
  echo "Generated random SSH password for root: $RANDOM_PASS"
  echo "This password is only shown once in logs!"
  echo "========================================="
else
  echo "root:$ROOT_PASSWORD" | chpasswd
  echo "SSH password set via ROOT_PASSWORD environment variable."
fi

# Ensure Dropbear host key exists
[ -f /etc/dropbear/dropbear_rsa_host_key ] || dropbearkey -t rsa -f /etc/dropbear/dropbear_rsa_host_key

# Start Dropbear in foreground with logging to stderr
exec dropbear -F -E -p 22
EOF

RUN chmod +x /etc/services.d/dropbear/run

# ==============================
# 配置 huproxy 服务（使用 -listen 标志）
# ==============================
RUN mkdir -p /etc/services.d/huproxy
RUN printf '#!/bin/sh\n\
echo "Starting huproxy without auth (use Caddy or client-side auth if needed)."\n\
exec huproxy -listen=0.0.0.0:8081\n' \
    > /etc/services.d/huproxy/run && \
    chmod +x /etc/services.d/huproxy/run

# ==============================
# 配置 inotify watcher 服务（自动重载新服务）
# ==============================
RUN mkdir -p /etc/services.d/.svc-watcher
RUN cat > /etc/services.d/.svc-watcher/run << 'EOF'
#!/bin/sh
echo "Watching /etc/services.d for new services..."
inotifywait -m -e create,moved_to --format "%f" /etc/services.d \
| while read dir; do
    if [ "$dir" != ".svc-watcher" ] && [ -d "/etc/services.d/$dir" ] && [ -x "/etc/services.d/$dir/run" ]; then
        echo "New service detected: $dir, reloading s6..."
        s6-svscanctl -a /etc/services.d
    fi
done
EOF
RUN chmod +x /etc/services.d/.svc-watcher/run

# ==============================
# 设置工作目录
# ==============================
WORKDIR /app
RUN rm -rf /tmp/* /var/tmp/*
# 使用 s6-overlay 入口点
ENTRYPOINT ["/init"]
