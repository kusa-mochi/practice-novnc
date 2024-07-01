FROM ubuntu:24.04 as su_exec
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y curl make gcc
RUN curl -s -L https://github.com/ncopa/su-exec/archive/v0.2.tar.gz | tar zx -C /opt/ \
    && mv /opt/su-exec* /opt/su-exec \
    && cd /opt/su-exec \
    && make

FROM ubuntu:20.04
ARG ADDITIONAL_APT_GET_OPTS=""
RUN echo 'path-include=/usr/share/locale/ja/LC_MESSAGES/*.mo' > /etc/dpkg/dpkg.cfg.d/includes \
    add-apt-repository "deb http://archive.ubuntu.com/ubuntu/ jammy main universe" \
    && apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y $ADDITIONAL_APT_GET_OPTS \
    dbus-x11 \
    fonts-noto-cjk \
    ibus \
    ibus-gtk \
    ibus-gtk3 \
    ibus-mozc \
    im-config \
    language-pack-ja \
    language-pack-ja-base \
    lightdm \
    lxqt \
    net-tools \
    novnc \
    papirus-icon-theme \
    sudo \
    supervisor \
    tzdata \
    x11vnc \
    xvfb \
    build-essential libglib2.0-dev git wget
RUN cd /tmp \
    && mkdir ibus \
    && cd ibus \
    && wget https://download.gnome.org/sources/gtk/4.0/gtk-4.0.0.tar.xz \
    && tar -xf gtk-4.0.0.tar.xz \
    && cd gtk-4.0.0 \
    && ./configure \
    && make \
    && make install \
    && git clone https://github.com/ibus/ibus.git \
    && cd ibus \
    && ./autogen.sh \
    && ./configure \
    && make \
    && make install \
    && ldconfig \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
COPY --from=su_exec \
    /opt/su-exec/su-exec /usr/sbin/su-exec
RUN cp /usr/share/zoneinfo/Asia/Tokyo /etc/localtime \
    && echo 'Asia/Tokyo' > /etc/timezone \
    && locale-gen ja_JP.UTF-8 \
    && echo 'LC_ALL=ja_JP.UTF-8' > /etc/default/locale \
    && echo 'LANG=ja_JP.UTF-8' >> /etc/default/locale
ENV LANG=ja_JP.UTF-8 \
    LANGUAGE=ja_JP:ja \
    LC_ALL=ja_JP.UTF-8
ENV DEFAULT_USER=developer \
    DEFAULT_PASSWD=vncpasswd
RUN echo 'ALL ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers.d/ALL
RUN chmod u+s /usr/sbin/useradd \
    && chmod u+s /usr/sbin/groupadd
EXPOSE 5900
EXPOSE 80
RUN install -o root -g root -m 0755 -d /var/run/dbus \
    && { \
        echo 'if [ -z "$XDG_RUNTIME_DIR" ] && [ -d "/run/user/$(id -u)" ]; then'; \
        echo '  export XDG_RUNTIME_DIR=/run/user/$(id -u)'; \
        echo 'fi'; \
    } > /etc/X11/Xsession.d/00xdg_runttime \
    && chmod 644 /etc/X11/Xsession.d/00xdg_runttime \
    && if [ -e /usr/share/lxqt/wallpapers/origami-light.png ]; then \
         update-alternatives --install \
           /usr/share/images/desktop-base/desktop-background desktop-background \
           /usr/share/lxqt/wallpapers/origami-light.png 99; \
       fi
RUN sed -i.org 's/plugins=.*/plugins=mainmenu, showdesktop, desktopswitch, quicklaunch, taskbar, tray, statusnotifier, worldclock/' /etc/xdg/lxqt/panel.conf \
    && mv /etc/xdg/autostart/lxqt-powermanagement.desktop /etc/xdg/autostart/lxqt-powermanagement.desktop.disabled
RUN { \
    echo '[supervisord]'; \
    echo 'user=root'; \
    echo 'nodaemon=true'; \
    echo 'logfile=/var/log/supervisor/supervisord.log'; \
    echo 'childlogdir=/var/log/supervisor'; \
    echo '[program:dbus]'; \
    echo 'priority=10'; \
    echo 'command=/usr/bin/dbus-daemon --system --nofork --nopidfile'; \
    echo '[program:lightdm]'; \
    echo 'priority=20'; \
    echo 'command=/usr/local/bin/start-lightdm.sh'; \
    echo '[program:x11vnc]'; \
    echo 'priority=30'; \
    echo 'startretries=5'; \
    echo 'command=/usr/bin/x11vnc -display :0 -auth /var/run/lightdm/root/:0 -rfbauth /etc/x11vnc.passwd -xkb -forever -shared -repeat -capslock'; \
    echo '[program:novnc]'; \
    echo 'priority=40'; \
    echo 'user=${USER}'; \
    echo 'command=/usr/share/novnc/utils/novnc_proxy --vnc localhost:5900 --listen 80'; \
    } > /etc/supervisor/vnc.conf.template
RUN { \
    # This script waits for D-Bus to be ready, then starts lightdm.
    # If lightdm starts before D-Bus is active, the startup of lightdm may fail.
    # Therefore, it is necessary to wait until D-Bus is ready before starting lightdm.
    # This script was created to solve this problem.
    echo '#!/bin/sh'; \
    echo 'while true'; \
    echo 'do'; \
    echo '    busctl > /dev/null 2>&1 && break'; \
    echo '    sleep 0.1'; \
    echo 'done'; \
    echo 'exec /usr/sbin/lightdm'; \
    } > /usr/local/bin/start-lightdm.sh \
    && chmod +x /usr/local/bin/start-lightdm.sh \
    && { \
    # This script starts Xvfb with the given resolution
    # Invoking Xvfb directly from lightdm may lead to errors due to unsupported options,
    # hence this script is used as a wrapper.
    echo '#!/bin/sh'; \
    echo 'RESOLUTION="$1"'; \
    echo 'exec /usr/bin/Xvfb :0 -screen 0 "$RESOLUTION" -nolisten tcp'; \
    } > /usr/local/bin/start-Xvfb.sh \
    && chmod +x /usr/local/bin/start-Xvfb.sh \
    && { \
    # This is a template for lightdm configuration
    echo '[Seat:*]'; \
    echo 'xserver-command=/usr/local/bin/start-Xvfb.sh "${RESOLUTION}"'; \
    echo 'autologin-user=${USER}'; \
    echo 'autologin-user-timeout=0'; \
    echo 'autologin-session=lxqt'; \
    } > /etc/lightdm/lightdm.conf.template
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["docker-entrypoint.sh"]
