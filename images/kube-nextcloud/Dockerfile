# Nextcloud 15 on CentOS 7.6
# Manual installation instructions: https://docs.nextcloud.com/server/15/admin_manual/installation/source_installation.html#example-installation-on-centos-7-server
FROM quay.io/centos/centos:centos7
LABEL name="kube-nextcloud" \
    description="Nextcould container image for Kubernetes/OpenShift, based on CentOS 7" \
    maintainer="Yuxiang Zhu <vfreex@gmail.com>"

# install dependencies (Apache HTTPD 2.4, PHP 7.2, etc)
RUN yum -y install centos-release-scl bzip2 \
    && yum -y install httpd24-httpd rh-php73 rh-php73-php rh-php73-php-gd rh-php73-php-mbstring \
        rh-php73-php-intl rh-php73-php-pecl-apcu rh-php73-php-mysqlnd rh-php73-php-pecl-redis \
        rh-php73-php-opcache sclo-php73-php-smbclient rh-php73-php-pecl-memcached rh-php73-php-imagick rh-php73-php-ldap rh-php73-php-pgsql \
        rh-php73-php-bcmath rh-php73-php-gmp \
    && yum clean all

# install Nextcloud to /var/www/html
ENV NEXTCLOUD_VERSION=21.0.2

RUN DL_DIR=$(mktemp -d) \
    && curl --fail -L -o "${DL_DIR}/nextcloud-${NEXTCLOUD_VERSION}.tar.bz2" \
        "https://download.nextcloud.com/server/releases/nextcloud-${NEXTCLOUD_VERSION}.tar.bz2" \
    && curl --fail -L -o "${DL_DIR}/nextcloud-${NEXTCLOUD_VERSION}.tar.bz2.asc" \
        "https://download.nextcloud.com/server/releases/nextcloud-${NEXTCLOUD_VERSION}.tar.bz2.asc" \
    && curl --fail -L -o "${DL_DIR}/nextcloud.asc" https://nextcloud.com/nextcloud.asc \
    && gpg --import "${DL_DIR}/nextcloud.asc" \
    && gpg --verify "${DL_DIR}/nextcloud-${NEXTCLOUD_VERSION}.tar.bz2.asc" "${DL_DIR}/nextcloud-${NEXTCLOUD_VERSION}.tar.bz2" \
    && rm -rf /var/www && mkdir -p /var/www \
    && tar -xvf "${DL_DIR}/nextcloud-${NEXTCLOUD_VERSION}.tar.bz2" -C /var/www \
    && mv /var/www/nextcloud /var/www/html \
    && chown apache:root -R /var/www/html \
    && rm -rf "${DL_DIR}"

WORKDIR /var/www/html

# configurate httpd
RUN ln -s /opt/rh/httpd24/root/etc/httpd /etc/httpd \
    && sed -i -E 's/^\s*Listen\s+.*/Listen 8080/g' /etc/httpd/conf/httpd.conf \
    && sed -i -E 's#^\s*ErrorLog\s+.*#ErrorLog /proc/self/fd/2#g' /etc/httpd/conf/httpd.conf \
    && sed -i -E 's#^\s*CustomLog\s+.*#CustomLog /proc/self/fd/1 combined#g' /etc/httpd/conf/httpd.conf  \
    && chown apache:root -R /opt/rh/httpd24/root/var/run/httpd/ \
    && chmod g+rwX -R /opt/rh/httpd24/root/var/run/httpd/

COPY httpd-conf.d /etc/httpd/conf.d

# configure PHP
# see https://docs.nextcloud.com/server/15/admin_manual/installation/server_tuning.html#enable-php-opcache
ENV PHP_CONFIG_DIR=/etc/opt/rh/rh-php73 \
    PHP_DATA_DIR=/var/opt/rh/rh-php73/lib/php \
    PATH=$PATH:/opt/rh/rh-php73/root/bin
    
RUN { \
        echo 'opcache.enable=1'; \
        echo 'opcache.enable_cli=1'; \
        echo 'opcache.interned_strings_buffer=8'; \
        echo 'opcache.max_accelerated_files=10000'; \
        echo 'opcache.memory_consumption=128'; \
        echo 'opcache.save_comments=1'; \
        echo 'opcache.revalidate_freq=1'; \
    } > "${PHP_CONFIG_DIR}"/php.d/opcache-recommended.ini; \
    \
    echo 'apc.enable_cli=1' >> "${PHP_CONFIG_DIR}"/php.d/docker-php-ext-apcu.ini; \
    \
    echo 'memory_limit=2048M' > "${PHP_CONFIG_DIR}"/php.d/memory-limit.ini; \
    \
    chown apache:root -R "${PHP_DATA_DIR}"/{session,opcache,wsdlcache};

# configure Nextcloud
COPY config /var/www/html/config

# set up volume directories
RUN mkdir -p /var/www/html/{data,custom_apps,config} \
    && chown apache:root -R /var/www/html/{data,custom_apps,config} \
    && chmod u=rwX,g=rwX,o= -R /var/www/html/{data,custom_apps,config,.htaccess,.user.ini} \
    && chmod a=rwx /var/www/html/occ

# startup scripts
COPY scripts /usr/local/bin
ENTRYPOINT [ "/usr/local/bin/docker-entrypoint.sh" ]
CMD [ "/usr/local/bin/run-httpd.sh" ]
EXPOSE 8080
# be compatible with OpenShift random UID!
USER 9999:0
