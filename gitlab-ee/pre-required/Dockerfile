FROM ubuntu:16.04
MAINTAINER Dongsoo Lee

# 1. Packages / Dependencies
ENV DEBIAN_FRONTEND noninteractive
RUN apt-get update -y \
    && apt-get upgrade -y \
    && apt-get install sudo -y \
	&& apt-get install -y vim \
	&& update-alternatives --set editor /usr/bin/vim.basic \
	&& apt-get install -y \
	   build-essential \
	   zlib1g-dev \
	   libyaml-dev \
	   libssl-dev \
	   libgdbm-dev \
	   libre2-dev \
	   libreadline-dev \
	   libncurses5-dev \
	   libffi-dev \
	   curl \
	   ca-certificates \
	   openssh-server \
	   wget \
	   apt-transport-https \
	   checkinstall \
	   libxml2-dev \
	   libxslt-dev \
	   libcurl4-openssl-dev \
	   libicu-dev \
	   logrotate \
	   rsync \
	   python-docutils \
	   pkg-config \
	   cmake \
	   libkrb5-dev \
	   git-core \
	   postfix 

# 2. Ruby
RUN mkdir /tmp/ruby && cd /tmp/ruby \
    && curl --remote-name --progress https://cache.ruby-lang.org/pub/ruby/2.3/ruby-2.3.6.tar.gz \
	&& echo '4e6a0f828819e15d274ae58485585fc8b7caace0  ruby-2.3.6.tar.gz' | shasum -c - && tar xzf ruby-2.3.6.tar.gz \
	&& cd ruby-2.3.6 \
	&& ./configure --disable-install-rdoc \
	&& make \
	&& make install \
	&& gem install bundler --no-ri --no-rdoc

# 3. Go
RUN rm -rf /usr/local/go \
    && curl --remote-name --progress https://storage.googleapis.com/golang/go1.8.3.linux-amd64.tar.gz \
	&& echo '1862f4c3d3907e59b04a757cfda0ea7aa9ef39274af99a784f5be843c80c6772  go1.8.3.linux-amd64.tar.gz' | shasum -a256 -c - \
    && tar -C /usr/local -xzf go1.8.3.linux-amd64.tar.gz \
    && ln -sf /usr/local/go/bin/{go,godoc,gofmt} /usr/local/bin/ \
    && rm go1.8.3.linux-amd64.tar.gz

# 4. Node
RUN curl --location https://deb.nodesource.com/setup_7.x | bash - \
    && apt-get install -y nodejs \
	&& curl --silent --show-error https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
	&& echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list \
	&& apt-get update \
	&& apt-get install yarn

# 5. System users
RUN adduser --disabled-login --gecos 'GitLab' git

# 6. Database
RUN apt-get install -y postgresql postgresql-client libpq-dev postgresql-contrib \
    && service postgresql restart \
    && sudo -u postgres psql -d template1 -c "CREATE USER git CREATEDB;" \
	&& sudo -u postgres psql -d template1 -c "CREATE EXTENSION IF NOT EXISTS pg_trgm;" \
	&& sudo -u postgres psql -d template1 -c "CREATE DATABASE gitlabhq_production OWNER git;" \
	&& sudo -u git -H psql -d gitlabhq_production

# 7. Redis
RUN apt-get install -y redis-server

# 8. Nginx
RUN apt-get install -y nginx
