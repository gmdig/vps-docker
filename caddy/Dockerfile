#ssh不支持CDN！！
FROM node:20-alpine

# 安装最小系统依赖
RUN apk add --no-cache \
    dropbear \
    inotify-tools \
    bash \
    curl \
    less \
    s6-overlay

# 安装 Caddy (layer4 版本)
COPY ./caddy /usr/local/bin/caddy
RUN chmod +x /usr/local/bin/caddy
RUN caddy version

# ---------- Node.js 日志服务 ----------
WORKDIR /app
COPY package.json . 
RUN npm install --omit=dev --no-fund --no-audit && npm cache clean --force
COPY server.js .

# ---------- 配置目录 ----------
RUN mkdir -p /etc/caddy /etc/services.d /var/log/s6

# ---------- 默认 Caddyfile ----------
RUN cat <<'EOF' > /etc/caddy/Caddyfile
{
    servers {
        listener_wrappers {
            layer4 {
                @ssh ssh
                route @ssh {
                    proxy 127.0.0.1:22
                }
                # 其他流量自动 fallthrough 到 HTTP 层
            }
        }
    }
}

:8888 {
    reverse_proxy 127.0.0.1:8080
}
EOF
RUN caddy fmt --overwrite /etc/caddy/Caddyfile

# ---------- Dropbear 服务 ----------
RUN mkdir -p /etc/services.d/dropbear /etc/services.d/dropbear/log && \
    cat <<'EOF' > /etc/services.d/dropbear/run
#!/bin/sh
if [ -n "$root_password" ]; then
  echo "root:$root_password" | chpasswd
  echo "[info] root password set from env root_password"
else
  echo "[warn] root_password not set; using default password"
  echo "root:changeme" | chpasswd
fi
rm -f /root/.ssh/authorized_keys || true
exec dropbear -R -E -F -p 22
EOF
RUN chmod +x /etc/services.d/dropbear/run

RUN cat <<'EOF' > /etc/services.d/dropbear/log/run
#!/bin/sh
mkdir -p /var/log/s6/dropbear
exec s6-log n20 s1000000 T /var/log/s6/dropbear
EOF
RUN chmod +x /etc/services.d/dropbear/log/run

RUN echo "longrun" > /etc/services.d/dropbear/type

# ---------- Caddy 服务 ----------
RUN mkdir -p /etc/services.d/caddy /etc/services.d/caddy/log && \
    cat <<'EOF' > /etc/services.d/caddy/run
#!/bin/sh
exec env HOME=/root caddy run --config /etc/caddy/Caddyfile --adapter caddyfile
EOF
RUN chmod +x /etc/services.d/caddy/run

RUN cat <<'EOF' > /etc/services.d/caddy/log/run
#!/bin/sh
mkdir -p /var/log/s6/caddy
exec s6-log n20 s1000000 T /var/log/s6/caddy
EOF
RUN chmod +x /etc/services.d/caddy/log/run

RUN echo "longrun" > /etc/services.d/caddy/type

# ---------- Watchdog 服务 ----------
RUN mkdir -p /etc/services.d/watchdog /etc/services.d/watchdog/log && \
    cat <<'EOF' > /etc/services.d/watchdog/run
#!/bin/sh
exec inotifywait -r -m -e modify,create,delete,move /etc/caddy/Caddyfile /etc/services.d \
    --format "%e %w%f" | while read event; do
    echo "[watchdog] $event"
    s6-svc -t /etc/services.d/caddy
    sleep 2
done
EOF
RUN chmod +x /etc/services.d/watchdog/run

RUN cat <<'EOF' > /etc/services.d/watchdog/log/run
#!/bin/sh
mkdir -p /var/log/s6/watchdog
exec s6-log n10 s100000 T /var/log/s6/watchdog
EOF
RUN chmod +x /etc/services.d/watchdog/log/run

RUN echo "longrun" > /etc/services.d/watchdog/type

# ---------- Node.js Log Viewer ----------
RUN mkdir -p /etc/services.d/log-viewer /etc/services.d/log-viewer/log && \
    cat <<'EOF' > /etc/services.d/log-viewer/run
#!/bin/sh
exec node /app/server.js
EOF
RUN chmod +x /etc/services.d/log-viewer/run

RUN cat <<'EOF' > /etc/services.d/log-viewer/log/run
#!/bin/sh
mkdir -p /var/log/s6/log-viewer
exec s6-log n20 s1000000 T /var/log/s6/log-viewer
EOF
RUN chmod +x /etc/services.d/log-viewer/log/run

RUN echo "longrun" > /etc/services.d/log-viewer/type

# ---------- 清理临时文件 ----------
RUN rm -rf /tmp/* /var/tmp/*

# ---------- 启动容器 ----------
ENTRYPOINT ["/init"]
