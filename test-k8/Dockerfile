FROM ubuntu:latest

RUN apt-get update && apt-get install -y \
    postgresql-client \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

COPY RootCA.pem /etc/ssl/certs/


ENV DB_HOST=rc1b-j03a218khoxjlwdi.mdb.yandexcloud.net
ENV DB_PORT=6432
ENV DB_NAME=edu-reverent-mestorf
ENV DB_USER=edu-reverent-mestorf
ENV DB_PASSWORD=tZtu31a_zxmWuBVm

CMD ["bash"]
