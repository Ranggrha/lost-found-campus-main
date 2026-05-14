# ============================================================
# Stage 1: Build frontend assets (Vite + TailwindCSS)
# ============================================================
FROM node:20-alpine AS frontend-builder

WORKDIR /app

# Copy package files first for layer caching
COPY package.json package-lock.json* ./

# Install Node dependencies
RUN npm ci

# Copy all source files needed for Vite build
COPY vite.config.js ./
COPY resources/ ./resources/
COPY public/ ./public/

# Build production frontend assets
RUN npm run build

# ============================================================
# Stage 2: PHP Application
# ============================================================
FROM php:8.2-apache

# Install system packages and PHP extensions
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    libzip-dev \
    zip \
    unzip \
    libcurl4-openssl-dev \
    libpq-dev \
    && docker-php-ext-install \
        pdo \
        pdo_mysql \
        pdo_pgsql \
        mbstring \
        exif \
        pcntl \
        bcmath \
        gd \
        zip \
        curl \
    && a2enmod rewrite headers \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install Composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

# Copy composer files first (Docker layer cache)
COPY composer.json composer.lock ./

# Install PHP dependencies (production only)
RUN composer install --no-dev --optimize-autoloader --no-scripts --no-interaction

# Copy all application source files
COPY . .

# Copy built frontend assets from stage 1
COPY --from=frontend-builder /app/public/build ./public/build

# Run composer post-install scripts
RUN composer dump-autoload --no-dev --optimize

# Set correct file permissions for Laravel
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html/storage \
    && chmod -R 755 /var/www/html/bootstrap/cache

# Configure Apache — point document root to /public
RUN echo '<VirtualHost *:80>\n\
    ServerAdmin webmaster@localhost\n\
    DocumentRoot /var/www/html/public\n\
    <Directory /var/www/html/public>\n\
        Options Indexes FollowSymLinks\n\
        AllowOverride All\n\
        Require all granted\n\
    </Directory>\n\
    ErrorLog ${APACHE_LOG_DIR}/error.log\n\
    CustomLog ${APACHE_LOG_DIR}/access.log combined\n\
</VirtualHost>' > /etc/apache2/sites-available/000-default.conf

# Create startup script
RUN printf '#!/bin/bash\n\
set -e\n\
echo "================================================"\n\
echo " Lost Found Campus — Starting Up"\n\
echo "================================================"\n\
\n\
echo "[1/4] Running database migrations..."\n\
php artisan migrate --force\n\
\n\
echo "[2/4] Seeding database (if empty)..."\n\
php artisan db:seed --force 2>/dev/null || true\n\
\n\
echo "[3/4] Linking storage..."\n\
php artisan storage:link --force 2>/dev/null || true\n\
\n\
echo "[4/4] Caching configuration..."\n\
php artisan config:cache\n\
php artisan route:cache\n\
php artisan view:cache\n\
\n\
echo "================================================"\n\
echo " Starting Apache Web Server..."\n\
echo "================================================"\n\
apache2-foreground\n' > /start.sh \
    && chmod +x /start.sh

EXPOSE 80

CMD ["/start.sh"]
