#!/usr/bin/env bash
#=================｜基础环境｜=======================      
export DEBIAN_FRONTEND=noninteractive

LOG="/root/init.log"
exec 1>>"$LOG" 2> >(tee -a "$LOG" >&2)

#====================｜函数｜======================      
#----------------------log---------------------------
log() {
  echo "$@" >&2
}
#-------------progress_bar_task--------------------
progress_bar_task() {
  local pid="$1"
  local label="$2"
  local width=40
  local percent=0

  while kill -0 "$pid" 2>/dev/null; do
    percent=$((percent + 1))
    ((percent > 99)) && percent=99

    local filled=$((percent * width / 100))
    local empty=$((width - filled))

    printf "\r\033[K" >&2

    printf "[%s] [" "$label" >&2
    printf "\033[42m%${filled}s\033[0m" "" | tr ' ' ' ' >&2
    printf "%${empty}s" "" >&2
    printf "] %3d%%" "$percent" >&2

    sleep 0.2
  done

  # 结束时补满 + 换行
  printf "\r\033[K[%s] [" "$label" >&2
  printf "\033[42m%${width}s\033[0m" "" | tr ' ' ' ' >&2
  printf "] 100%%\n" >&2
}
#----------------------彩色文字-----------------------
cecho() {
  local color="$1"
  local style="$2"
  local text

  if [[ $# -eq 2 ]]; then
    text="$2"
    style=""
  else
    text="$3"
  fi

  local code=""

  case "$color" in
    black)   code="30" ;;
    red)     code="31" ;;
    green)   code="32" ;;
    yellow)  code="33" ;;
    blue)    code="34" ;;
    purple)  code="35" ;;
    cyan)    code="36" ;;
    white)   code="37" ;;
    *)       code="0"  ;;
  esac

  case "$style" in
    bold)    code="1;${code}" ;;
    dim)     code="2;${code}" ;;
    underline) code="4;${code}" ;;
  esac

  printf "\033[%sm%s\033[0m\n" "$code" "$text" >&2
}
#---------------------retry-----------------------
retry() {
  local max=3
  local n=0

  until "$@"; do
    ((n++))
    if [[ $n -ge $max ]]; then
      cecho red bold "[FAIL] $*（已重试 $n 次）"
      return 1
    fi
    cecho yellow "[RETRY] $*（第 $n 次）"
    sleep 2
  done
}


#------------------------apt_run-------------------
apt_run() {
  local msg="$1"
  shift
  apt -o Dpkg::Progress-Fancy="0" "$@" >/dev/null 2>&1 &
  local pid=$!
  
  progress_bar_task "$pid" "$msg"
  wait "$pid"
}

cecho blue bold "===== PPanel-docker后端部署 ====="
cecho green  "api后端域名SSL证书配置 "
cecho white  "请确保DNS已正确指向此服务器，且已申请了SSL证书 "
read -p "请输入后端api域名 >" apidomain && export apidomain
read -p "[1]SSL证书路径 -- [直接回车]粘贴SSL证书内容>" SSLchoice && export SSLchoice ||SSLchoice=$2 && export SSLchoice
if test $SSLchoice = "1"; then
   cecho yellow "下面请正确输入证书路径"
   read -p "源文件路径>" fullchain && export fullchain
   read -p "密钥路径>" privkey && export privkey
else
   read -p "[1]单证书 -- [直接回车]通配符证书> " SSLchoices && export SSLchoices ||SSLchoices=$2 && export SSLchoices
   if test $SSLchoices = "1"; then
      read -p "请输入通配符主域>" SSLdomain && export SSLdomain
   else
      SSLdomain=$apidomain && export SSLdomain
   fi
   read -p "请粘贴源文件内容（.pem）>" fullchaintext && export fullchaintext
   read -p "请粘贴密钥内容（.pem）>" privkeytext && export privkeytext
fi
cecho green  "请输入MySQL数据库密码 "
read -p "直接回车默认随机生成 >" MYSQLpassword && export MYSQLpassword ||apt install -y uuid-runtime >/dev/null 2>&1 && MYSQLpassword=$(uuidgen) && export MYSQLpassword
cecho green  "请输入JWT密码 "
read -p "直接回车默认随机生成 >" JWTpassword && export JWTpassword ||apt install -y uuid-runtime >/dev/null 2>&1 && JWTpassword=$(uuidgen) && export JWTpassword



#===================｜系统更新｜======================
curl -fsSL https://get.docker.com | sh
systemctl start docker || cecho red  "docker 启动失败"
systemctl enable docker|| cecho red  "docker 启动失败"
mkdir -p /root/ppanel/
mkdir -p /root/ppanel/ppanel-config/
touch /root/ppanel/ppanel-config/ppanel.yaml
touch /root/ppanel/docker-compose.yml
cat > /root/ppanel/ppanel-config/ppanel.yaml <<'EOF'
# 数据库配置
database:
  type: mysql
  host: localhost
  port: 3306
  username: ppanel
  password: $MYSQLpassword
  database: ppanel

# Redis 配置
redis:
  host: localhost
  port: 6379
  password: ""
  db: 0

# 服务配置
server:
  host: 0.0.0.0
  port: 8080

# CORS 配置
cors:
  allow_origins:
    - "https://$apidomain"
    - "http://localhost:3000"  # 开发环境
  allow_methods:
    - GET
    - POST
    - PUT
    - DELETE
    - OPTIONS
  allow_headers:
    - "*"

# JWT 配置
jwt:
  secret: "$JWTpassword"
  expire: 7200  # 2小时

# API 配置
api:
  prefix: "/api"
  version: "v1"
  
EOF
cd /root/ppanel/
docker run -d \
  --name ppanel-mysql \
  -e MYSQL_ROOT_PASSWORD=Cici080306 \
  -e MYSQL_DATABASE=ppanel \
  -e MYSQL_USER=ppanel \
  -e MYSQL_PASSWORD=Cici080306 \
  -p 3306:3306 \
  -v ppanel-mysql-data:/var/lib/mysql \
  mysql:5.7 \
  >/dev/null 2>&1 || cecho red  "MySQL 容器启动失败" && exit 1
sleep 10
docker run -d \
  --name ppanel-redis \
  -p 6379:6379 \
  -v ppanel-redis-data:/data \
  redis:7-alpine \
  >/dev/null 2>&1 || cecho red  "Redis 容器启动失败" && exit 1
docker pull ppanel/ppanel:latest >/dev/null 2>&1 || cecho red "镜像拉取失败"
docker run -d \
  --name ppanel-backend \
  -p 8080:8080 \
  -v $(pwd)/config.yaml:/app/config.yaml \
  --link ppanel-mysql:mysql \
  --link ppanel-redis:redis \
  ppanel/ppanel:latest \
     >/dev/null 2>&1 || cecho red  "启动失败" && exit 1
cd /root/
docker exec ppanel-backend ./gateway migrate >/dev/null 2>&1  || cecho red "数据库迁移取失败"
docker ps
apt update -y >/dev/null 2>&1 && apt upgrade -y >/dev/null 2>&1
apt install -y nginx >/dev/null 2>&1 && systemctl start nginx >/dev/null 2>&1
systemctl enable nginx >/dev/null 2>&1
cd /root/
if test $SSLchoice = "1"; then
   DomainSSLfullchain=fullchain
   DomainSSLprivkey=privkey
else
   DomainSSLfullchain=/etc/nginx/ssl/$SSLdomain/fullchain.pem
   DomainSSLprivkey=/etc/nginx/ssl/$SSLdomain/privkey.pem
mkdir -p /etc/nginx/ssl/
mkdir -p /etc/nginx/ssl/$SSLdomain/
touch /etc/nginx/ssl/$SSLdomain/fullchain.pem
touch /etc/nginx/ssl/$SSLdomain/privkey.pem
cat > /etc/nginx/ssl/$SSLdomain/fullchain.pem <<'EOF'
$ fullchaintext
EOF
cat > /etc/nginx/ssl/$SSLdomain/privkey.pem <<'EOF'
$ privkeytext
EOF
chmod 600 /etc/nginx/ssl/$SSLdomain/*
fi
touch /etc/nginx/sites-available/ppanel.conf
cat > /etc/nginx/sites-available/ppanel.conf <<'EOF'
server {
    listen 80;
    server_name $apidomain;

    return 301 https://$host$request_uri;
}


server {
    listen 443 ssl http2;
    server_name $apidomain;

    ssl_certificate     DomainSSLfullchain;
    ssl_certificate_key DomainSSLprivkey;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;

    location / {
        proxy_pass http://127.0.0.1:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
EOF
ln -s /etc/nginx/sites-available/ppanel.conf \
      /etc/nginx/sites-enabled/ppanel.conf\
          >/dev/null 2>&1
nginx -t >/dev/null 2>&1
systemctl reload nginx >/dev/null 2>&1
