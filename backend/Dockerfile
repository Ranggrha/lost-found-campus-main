FROM php:8.2-apache

# Install system dependencies & PHP extensions
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
    && docker-php-ext-install \
        pdo \
        pdo_mysql \
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

# Set working directory
WORKDIR /var/www/html

# Copy composer files first (Docker layer cache optimization)
COPY composer.json composer.lock ./

# Install PHP dependencies (no dev, optimized)
RUN composer install --no-dev --optimize-autoloader --no-scripts --no-interaction

# Copy all application files
COPY . .

# Run post-install scripts
RUN composer dump-autoload --no-dev --optimize

# Set correct permissions for Laravel
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html/storage \
    && chmod -R 755 /var/www/html/bootstrap/cache

# Configure Apache to point to Laravel's /public folder
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
echo "=== Starting Lost Found Campus API ==="\n\
echo "Running migrations..."\n\
php artisan migrate --force\n\
echo "Linking storage..."\n\
php artisan storage:link --force 2>/dev/null || true\n\
echo "Caching config..."\n\
php artisan config:cache\n\
php artisan route:cache\n\
php artisan view:cache\n\
echo "=== Starting Apache ==="\n\
apache2-foreground\n' > /start.sh \
    && chmod +x /start.sh

EXPOSE 80

CMD ["/start.sh"]
