# Docker-Moodle
FROM moodlehq/moodle-php-apache:8.3-bookworm
LABEL maintainer="EC <me@ceduth.dev>"

VOLUME ["/var/moodledata"]
EXPOSE 80 443
COPY moodle-config.php /var/www/html/config.php

# Let the container know that there is no tty
ENV DEBIAN_FRONTEND noninteractive

# Database info and other connection information derived from env variables. See README.
# Set ENV Variables externally Moodle_URL should be overridden.
ENV MOODLE_URL http://127.0.0.1

# Enable when using external SSL reverse proxy
# Default: false
ENV SSL_PROXY false

COPY ./foreground.sh /etc/apache2/foreground.sh

RUN apt-get update && apt-get upgrade -y && \
	apt-get install -y cron ssl-cert && \
	cd /tmp && \
	git clone -b MOODLE_404_STABLE git://git.moodle.org/moodle.git --depth=1 && \
	mv /tmp/moodle/* /var/www/html/ && \
	# rm /var/www/html/index.html && \
	chown -R www-data:www-data /var/www/html && \
	chmod +x /etc/apache2/foreground.sh

# Cron
COPY moodlecron /etc/cron.d/moodlecron
RUN chmod 0644 /etc/cron.d/moodlecron

# Enable SSL
# If using proxy, we don't actually need a secure connection
RUN if [ "$SSL_PROXY" = "false" ]; then \
	a2enmod ssl && a2ensite default-ssl ; \
fi

# Cleanup, this is ran to reduce the resulting size of the image.
RUN apt-get clean autoclean && apt-get autoremove -y \
	&& rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* /var/lib/dpkg/* /var/lib/cache/* /var/lib/log/*


ENTRYPOINT ["/etc/apache2/foreground.sh"]
