FROM ubuntu:16.04
MAINTAINER Dongsoo Lee

# 1. Packages / Dependencies
ENV DEBIAN_FRONTEND noninteractive
RUN apt-get update -y \
    && apt-get upgrade -y \
    && apt-get install sudo -y \
	&& apt-get install -yq --no-install-recommends \
	   vim \
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
	   capt-transport-https \
	   heckinstall \
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
	   postfix \
	&& update-alternatives --set editor /usr/bin/vim.basic

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
    && -u postgres psql -d template1 -c "CREATE USER git CREATEDB;" \
	&& -u postgres psql -d template1 -c "CREATE EXTENSION IF NOT EXISTS pg_trgm;" \
	&& -u postgres psql -d template1 -c "CREATE DATABASE gitlabhq_production OWNER git;" \
	&& -u git -H psql -d gitlabhq_production

# 7. Redis
RUN apt-get install -y redis-server

# 8. GitLab
RUN cd /home/git \
    && -u git -H git clone https://gitlab.com/gitlab-org/gitlab-ee.git -b 10-4-stable-ee gitlab

# GitLab Configure
RUN cd /home/git/gitlab \
    && -u git -H cp config/gitlab.yml.example config/gitlab.yml \
	&& -u git -H cp config/secrets.yml.example config/secrets.yml \
	&& -u git -H chmod 0600 config/secrets.yml \
	&& chown -R git log/ \
	&& chown -R git tmp/ \
	&& chmod -R u+rwX,go-w log/ \
	&& chmod -R u+rwX tmp/ \
	&& chmod -R u+rwX tmp/pids/ \
	&& chmod -R u+rwX tmp/sockets/ \
	&& -u git -H mkdir public/uploads/ \
	&& chmod 0700 public/uploads \
	&& chmod -R u+rwX builds/ \
	&& chmod -R u+rwX shared/artifacts/ \
	&& chmod -R ug+rwX shared/pages/ \
	&& -u git -H cp config/unicorn.rb.example config/unicorn.rb \
	&& -u git -H cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb \
	&& -u git -H git config --global core.autocrlf input \
	&& -u git -H git config --global gc.auto 0 \
	&& -u git -H git config --global repack.writeBitmaps true \
	&& -u git -H git config --global receive.advertisePushOptions true \
	&& -u git -H cp config/resque.yml.example config/resque.yml

# GitLab DB Configure
RUN -u git cp config/database.yml.postgresql config/database.yml \
    && -u git cp config/database.yml.mysql config/database.yml \
	&& -u git -H chmod o-rwx config/database.yml

# Install Gems
RUN -u git -H bundle install --deployment --without development test mysql aws kerberos \
    && -u git -H bundle install --deployment --without development test postgres aws kerberos

# Install GitLab Shell
RUN -u git -H bundle exec rake gitlab:shell:install REDIS_URL=unix:/var/run/redis/redis.sock RAILS_ENV=production SKIP_STORAGE_VALIDATION=true \
	&& export RUBYOPT=--disable-gems

# Install gitlab-workhorse
RUN -u git -H bundle exec rake "gitlab:workhorse:install[/home/git/gitlab-workhorse]" RAILS_ENV=production

# Initialize Database and Activate Advanced Features
RUN -u git -H bundle exec rake gitlab:setup RAILS_ENV=production \
    && -u git -H bundle exec rake gitlab:setup RAILS_ENV=production GITLAB_ROOT_PASSWORD=yourpassword GITLAB_ROOT_EMAIL=youremail

# Install Init Script
RUN cp lib/support/init.d/gitlab /etc/init.d/gitlab \
    && cp lib/support/init.d/gitlab.default.example /etc/default/gitlab \
	&& update-rc.d gitlab defaults 21

# Install Gitaly
RUN -u git -H bundle exec rake "gitlab:gitaly:install[/home/git/gitaly]" RAILS_ENV=production \
    && -u git -H bundle exec rake "gitlab:gitaly:install[/home/git/gitaly,https://example.com/gitaly.git]" RAILS_ENV=production \
	&& chmod 0700 /home/git/gitlab/tmp/sockets/private \
	&& chown git /home/git/gitlab/tmp/sockets/private \
	&& cd /home/git/gitaly \
	&& -u git -H editor config.toml

# Setup Logrotate
RUN cp lib/support/logrotate/gitlab /etc/logrotate.d/gitlab

# Check Application Status
RUN -u git -H bundle exec rake gitlab:env:info RAILS_ENV=production

# Compile GetText PO files
RUN -u git -H bundle exec rake gettext:compile RAILS_ENV=production

# Compile Assets
RUN -u git -H yarn install --production --pure-lockfile \
    && -u git -H bundle exec rake gitlab:assets:compile RAILS_ENV=production NODE_ENV=production

# Start GitLab Instance
# RUN service gitlab restart

# 9. Nginx
RUN apt-get install -y nginx \
    && cp lib/support/nginx/gitlab /etc/nginx/sites-available/gitlab \
	&& ln -s /etc/nginx/sites-available/gitlab /etc/nginx/sites-enabled/gitlab \
	&& nginx -t \
	&& service nginx restart


# Manage SSHD through runit
RUN mkdir -p /opt/gitlab/sv/sshd/supervise \
    && mkfifo /opt/gitlab/sv/sshd/supervise/ok \
    && printf "#!/bin/sh\nexec 2>&1\numask 077\nexec /usr/sbin/sshd -D" > /opt/gitlab/sv/sshd/run \
    && chmod a+x /opt/gitlab/sv/sshd/run \
    && ln -s /opt/gitlab/sv/sshd /opt/gitlab/service \
    && mkdir -p /var/run/sshd
	
# Disabling use DNS in ssh since it tends to slow connecting
RUN echo "UseDNS no" >> /etc/ssh/sshd_config

# Prepare default configuration
RUN ( \
  echo "" && \
  echo "# Docker options" && \
  echo "# Prevent Postgres from trying to allocate 25% of total memory" && \
  echo "postgresql['shared_buffers'] = '1MB'" ) >> /etc/gitlab/gitlab.rb && \
  mkdir -p /assets/ && \
  cp /etc/gitlab/gitlab.rb /assets/gitlab.rb

# Expose web & ssh
EXPOSE 443 80 22

# Define data volumes
VOLUME ["/etc/gitlab", "/var/opt/gitlab", "/var/log/gitlab"]

# Copy assets
COPY assets/wrapper /usr/local/bin/

# Wrapper to handle signal, trigger runit and reconfigure GitLab
CMD ["/usr/local/bin/wrapper"]
