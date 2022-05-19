FROM ykunisato/paper-r:latest

COPY XXXX.crt /usr/share/ca-certificates/xxxx/
RUN echo "xxxx/XXXX.crt" >> /etc/ca-certificates.conf && update-ca-certificates
ENV PASSWORD=password