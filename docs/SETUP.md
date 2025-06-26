# Vibe-App 開発環境セットアップガイド (Docker Compose)

このドキュメントは、LaravelバックエンドとNext.jsフロントエンドで構成されるVibe-Appのローカル開発環境をDocker Composeを使用してセットアップする手順を説明します。

## 1. 前提条件

*   Docker Desktop (またはDocker EngineとDocker Compose) がインストールされていること。
*   Gitがインストールされていること。

## 2. プロジェクトのクローン (初回のみ)

```bash
git clone <リポジトリのURL> vibe-app
cd vibe-app
```

## 3. ディレクトリ構造の作成

プロジェクトのルートディレクトリ (`vibe-app`) に移動し、以下のディレクトリを作成します。

```bash
mkdir backend frontend nginx docs
```

## 4. Docker Compose ファイルの作成

`vibe-app` ディレクトリ直下に `docker-compose.yml` ファイルを作成し、以下の内容を記述します。

```yaml
version: '3.8'

services:
  # PHP-FPM Service (for Laravel)
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    volumes:
      - ./backend:/var/www/html
    ports:
      - "9000:9000"
    depends_on:
      - mysql
    networks:
      - app-network

  # Nginx Service
  nginx:
    image: nginx:stable-alpine
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
      - ./backend:/var/www/html
      - ./frontend/out:/var/www/frontend
    ports:
      - "8080:80" # ホストの8080ポートをコンテナの80ポートにマッピング
      - "8443:443" # ホストの8443ポートをコンテナの443ポートにマッピング
    depends_on:
      - backend
      - frontend
    networks:
      - app-network

  # MySQL Database Service
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: vibe_app_db
      MYSQL_USER: user
      MYSQL_PASSWORD: password
    volumes:
      - db_data:/var/lib/mysql
    ports:
      - "3306:3306"
    networks:
      - app-network

  # Node.js Service (for Next.js development/build)
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    volumes:
      - ./frontend:/app
      - /app/node_modules # Anonymous volume to prevent host node_modules from overwriting container's
    ports:
      - "3000:3000"
    networks:
      - app-network
    command: npm run dev # Or npm run build && npm run start for production

networks:
  app-network:
    driver: bridge

volumes:
  db_data:
```

## 5. Dockerfile の作成

### 5.1. `backend/Dockerfile`

`backend` ディレクトリ内に `Dockerfile` を作成し、以下の内容を記述します。

```dockerfile
FROM php:8.2-fpm-alpine

WORKDIR /var/www/html

RUN apk add --no-cache \
    nginx \
    mysql-client \
    git \
    curl \
    zip \
    unzip \
    nodejs \
    npm

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

RUN docker-php-ext-install pdo_mysql

EXPOSE 9000
CMD ["php-fpm"]
```

### 5.2. `frontend/Dockerfile`

`frontend` ディレクトリ内に `Dockerfile` を作成し、以下の内容を記述します。

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "run", "dev"]
```

## 6. Nginx 設定ファイルの作成

`nginx` ディレクトリ内に `nginx.conf` ファイルを作成し、以下の内容を記述します。

```nginx
server {
    listen 80;
    server_name localhost;
    root /var/www/html/public; # Laravel public directory

    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri /index.php =404;
        fastcgi_pass backend:9000; # Connect to the backend service
        fastcgi_index index.php;
        fastcgi_buffers 16 16k;
        fastcgi_buffer_size 32k;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # Next.js static files (if built to /frontend/out)
    location /_next/static/ {
        alias /var/www/frontend/_next/static/;
        expires 1y;
        access_log off;
        add_header Cache-Control "public";
    }

    # Next.js API routes (if any, proxy to Next.js dev server or built server)
    # This is a placeholder. For production, you might build Next.js and serve it directly,
    # or proxy to a running Next.js server.
    location /api/nextjs/ {
        proxy_pass http://frontend:3000/api/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

## 7. Next.js プロジェクトの作成

`frontend` ディレクトリにNext.jsプロジェクトを作成します。

```bash
npx create-next-app@latest frontend --ts --eslint --tailwind --app --src-dir --use-npm --no-import-alias --yes
```

## 8. Dockerコンテナのビルドと起動

プロジェクトのルートディレクトリ (`vibe-app`) で以下のコマンドを実行し、Dockerコンテナをビルドして起動します。

```bash
docker-compose up -d --build
```

## 9. Laravel プロジェクトのインストール

`backend` コンテナ内でLaravelプロジェクトをインストールします。

```bash
docker-compose exec backend composer create-project laravel/laravel .
```

## 10. Laravel の `.env` 設定

`backend/.env` ファイルを開き、以下のデータベース接続情報を設定します。

```dotenv
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=vibe_app_db
DB_USERNAME=user
DB_PASSWORD=password
```

## 11. データベースマイグレーションの実行

`backend` コンテナ内でLaravelのデータベースマイグレーションを実行し、テーブルを作成します。

```bash
docker-compose exec backend php artisan migrate
```

## 12. 環境の確認

*   Nginx: `http://localhost:8080` にアクセスして、Laravelのウェルカムページが表示されることを確認します。
*   Next.js: `http://localhost:3000` にアクセスして、Next.jsのウェルカムページが表示されることを確認します。

---

これで、Vibe-AppのDocker Compose開発環境のセットアップが完了しました。
