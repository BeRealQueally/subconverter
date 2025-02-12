FROM alpine:3.16 AS builder
LABEL maintainer="tindy.it@gmail.com"
ARG THREADS="4"
ARG SHA=""

# build minimized
WORKDIR /
RUN set -xe && \
    apk add --no-cache --virtual .build-tools git g++ build-base linux-headers cmake python3 && \
    apk add --no-cache --virtual .build-deps curl-dev rapidjson-dev pcre2-dev yaml-cpp-dev && \
    git clone https://github.com/ftk/quickjspp --depth=1 && \
    cd quickjspp && \
    git submodule update --init && \
    cmake -DCMAKE_BUILD_TYPE=Release . && \
    make quickjs -j $THREADS && \
    install -d /usr/lib/quickjs/ && \
    install -m644 quickjs/libquickjs.a /usr/lib/quickjs/ && \
    install -d /usr/include/quickjs/ && \
    install -m644 quickjs/quickjs.h quickjs/quickjs-libc.h /usr/include/quickjs/ && \
    install -m644 quickjspp.hpp /usr/include && \
    cd .. && \
    git clone https://github.com/PerMalmberg/libcron --depth=1 && \
    cd libcron && \
    git submodule update --init && \
    cmake -DCMAKE_BUILD_TYPE=Release . && \
    make libcron -j $THREADS && \
    install -m644 libcron/out/Release/liblibcron.a /usr/lib/ && \
    install -d /usr/include/libcron/ && \
    install -m644 libcron/include/libcron/* /usr/include/libcron/ && \
    install -d /usr/include/date/ && \
    install -m644 libcron/externals/date/include/date/* /usr/include/date/ && \
    cd .. && \
    git clone https://github.com/ToruNiina/toml11 --branch="v4.3.0" --depth=1 && \
    cd toml11 && \
    cmake -DCMAKE_CXX_STANDARD=11 . && \
    make install -j $THREADS && \
    cd .. && \
    git clone https://github.com/tindy2013/subconverter --depth=1 && \
    cd subconverter && \
    [ -n "$SHA" ] && sed -i 's/\(v[0-9]\.[0-9]\.[0-9]\)/\1-'"$SHA"'/' src/version.h;\
    sed -i '/TW/s/🇨🇳/🇹🇼/g' base/snippets/emoji.txt && \
    python3 -m ensurepip && \
    python3 -m pip install gitpython && \
    python3 scripts/update_rules.py -c scripts/rules_config.conf && \
    cmake -DCMAKE_BUILD_TYPE=Release . && \
    make -j $THREADS

# build final image
FROM alpine:3.16
LABEL maintainer="tindy.it@gmail.com"

RUN apk add --no-cache --virtual subconverter-deps pcre2 libcurl yaml-cpp git dcron python3 py3-pip tzdata

COPY --from=builder /subconverter/subconverter /usr/bin/
COPY --from=builder /subconverter/base /base/
COPY --from=builder /subconverter/scripts/rules_config.conf /scripts/rules_config.conf
COPY --from=builder /subconverter/scripts/update_rules.py /scripts/update_rules.py

ENV TZ=Africa/Abidjan
RUN ln -sf /usr/share/zoneinfo/$TZ /etc/localtime
RUN echo $TZ > /etc/timezone

RUN mkdir -p /var/spool/cron/

RUN python3 -m ensurepip && python3 -m pip install gitpython

RUN sed -i '/^\[ACL4SSR_config\]$/,/^\[/ { /^\[ACL4SSR_config\]$/d; /^\[/!d }' scripts/rules_config.conf
RUN echo "#!/bin/sh" > /etc/periodic/daily/update_rules.sh && \
    echo "cd / && python3 scripts/update_rules.py -c scripts/rules_config.conf" >> /etc/periodic/daily/update_rules.sh && \
    chmod +x /etc/periodic/daily/update_rules.sh

# set entry
WORKDIR /base
CMD crond -L /var/log/cron.log && tail -f /var/log/cron.log & exec subconverter

EXPOSE 25500/tcp
