#
# Ubuntu Bionic + Systemd + sshd + Docker
#
# Usage:
#
# $ docker run --runtime=sysbox-runc -it --rm -P --name=syscont nestybox/ubuntu-bionic-systemd-docker
#
# This will run systemd and prompt for a user login; the default
# user/password in this image is "admin/admin". Once you log in you
# can run Docker inside as usual. You can also ssh into the image:
#
# $ ssh admin@<host-ip> -p <host-port>
#
# where <host-port> is chosen by Docker and mapped into the system container's sshd port.
#

FROM nestybox/ubuntu-bionic-systemd:latest

# Docker install
RUN apt-get update && apt-get install --no-install-recommends -y \
       apt-transport-https \
       ca-certificates \
       curl \
       gnupg-agent \
       software-properties-common
RUN curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
RUN apt-key fingerprint 0EBFCD88

RUN add-apt-repository \
       "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
       $(lsb_release -cs) \
       stable"
RUN apt-get update && apt-get install --no-install-recommends -y docker-ce docker-ce-cli containerd.io

# Add user "admin" to the Docker group
RUN usermod -a -G docker admin

# sshd install
RUN apt-get update && apt-get install --no-install-recommends -y openssh-server
RUN mkdir /var/run/sshd
RUN echo 'admin:admin' | chpasswd
RUN sed -ri 's/^#?PermitRootLogin\s+.*/PermitRootLogin yes/' /etc/ssh/sshd_config
RUN sed -ri 's/UsePAM yes/#UsePAM yes/g' /etc/ssh/sshd_config
RUN mkdir /root/.ssh
RUN mkdir /home/admin/.ssh
EXPOSE 22

# Set systemd as entrypoint.
ENTRYPOINT [ "/sbin/init", "--log-level=err" ]
