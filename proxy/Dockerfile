FROM centos:7
RUN su -

# Set Proxy
# ARG HTTP_PROXY
# ARG HTTPS_PROXY

RUN yum -y update
RUN yum -y install squid

RUN yum clean all

COPY ./squid.conf /etc/squid/squid.conf

EXPOSE 8080

CMD /sbin/init
