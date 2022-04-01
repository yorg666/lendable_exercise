FROM alpine:3.8

RUN addgroup www
RUN adduser -D -g 'www' nginx

RUN apk update && \
    apk add nginx && \
    apk add git && \
    apk add openssh


RUN mkdir /www && \
    chown -R nginx:www /var/lib/nginx && \
    chown -R nginx:www /www

RUN mkdir -p /run/nginx

RUN mkdir -m 700 /root/.ssh; \
  touch -m 600 /root/.ssh/known_hosts; \
  ssh-keyscan github.com > /root/.ssh/known_hosts

RUN --mount=type=ssh,id=github git clone git@github.com:yorg666/lendable_exercise.git

WORKDIR ./lendable_exercise
COPY ./www/ /www/
COPY nginx/default.conf /etc/nginx/conf.d/default.conf

ENTRYPOINT nginx -g 'daemon off;'