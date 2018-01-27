FROM dongsoolee/gitlab-deps:latest
MAINTAINER Dongsoo Lee

# 1. Packages / Dependencies
ENV DEBIAN_FRONTEND noninteractive

# 2. GitLab
RUN cd /home/git \
    && sudo -u git -H git clone https://gitlab.com/gitlab-org/gitlab-ee.git -b 10-4-stable-ee gitlab

# GitLab Configure
RUN cd /home/git/gitlab \
    && sudo -u git -H cp config/gitlab.yml.example config/gitlab.yml \
	&& sudo -u git -H cp config/secrets.yml.example config/secrets.yml \
	&& sudo -u git -H chmod 0600 config/secrets.yml \
	&& chown -R git log/ \
	&& chown -R git tmp/ \
	&& chmod -R u+rwX,go-w log/ \
	&& chmod -R u+rwX tmp/ \
	&& chmod -R u+rwX tmp/pids/ \
	&& chmod -R u+rwX tmp/sockets/ \
	&& sudo -u git -H mkdir public/uploads/ \
	&& chmod 0700 public/uploads \
	&& chmod -R u+rwX builds/ \
	&& chmod -R u+rwX shared/artifacts/ \
	&& chmod -R ug+rwX shared/pages/ \
	&& sudo -u git -H cp config/unicorn.rb.example config/unicorn.rb \
	&& sudo -u git -H cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb \
	&& sudo -u git -H git config --global core.autocrlf input \
	&& sudo -u git -H git config --global gc.auto 0 \
	&& sudo -u git -H git config --global repack.writeBitmaps true \
	&& sudo -u git -H git config --global receive.advertisePushOptions true \
	&& sudo -u git -H cp config/resque.yml.example config/resque.yml

# GitLab DB Configure
RUN sudo -u git cp config/database.yml.postgresql config/database.yml \
    && sudo -u git cp config/database.yml.mysql config/database.yml \
	&& sudo -u git -H chmod o-rwx config/database.yml

# Install Gems
RUN sudo -u git -H bundle install --deployment --without development test mysql aws kerberos \
    && sudo -u git -H bundle install --deployment --without development test postgres aws kerberos

# Install GitLab Shell
RUN sudo -u git -H bundle exec rake gitlab:shell:install REDIS_URL=unix:/var/run/redis/redis.sock RAILS_ENV=production SKIP_STORAGE_VALIDATION=true \
	&& export RUBYOPT=--disable-gems

# Install gitlab-workhorse
RUN sudo -u git -H bundle exec rake "gitlab:workhorse:install[/home/git/gitlab-workhorse]" RAILS_ENV=production

# Initialize Database and Activate Advanced Features
RUN sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production \
    && sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production GITLAB_ROOT_PASSWORD=yourpassword GITLAB_ROOT_EMAIL=youremail

# Install Init Script
RUN cp lib/support/init.d/gitlab /etc/init.d/gitlab \
    && cp lib/support/init.d/gitlab.default.example /etc/default/gitlab \
	&& update-rc.d gitlab defaults 21

# Install Gitaly
RUN sudo -u git -H bundle exec rake "gitlab:gitaly:install[/home/git/gitaly]" RAILS_ENV=production \
    && sudo -u git -H bundle exec rake "gitlab:gitaly:install[/home/git/gitaly,https://example.com/gitaly.git]" RAILS_ENV=production \
	&& chmod 0700 /home/git/gitlab/tmp/sockets/private \
	&& chown git /home/git/gitlab/tmp/sockets/private \
	&& cd /home/git/gitaly \
	&& sudo -u git -H editor config.toml

# Setup Logrotate
RUN cp lib/support/logrotate/gitlab /etc/logrotate.d/gitlab

# Check Application Status
RUN sudo -u git -H bundle exec rake gitlab:env:info RAILS_ENV=production

# Compile GetText PO files
RUN sudo -u git -H bundle exec rake gettext:compile RAILS_ENV=production

# Compile Assets
RUN sudo -u git -H yarn install --production --pure-lockfile \
    && sudo -u git -H bundle exec rake gitlab:assets:compile RAILS_ENV=production NODE_ENV=production

# Start GitLab Instance
RUN service gitlab restart

# 9. Nginx
RUN cp lib/support/nginx/gitlab /etc/nginx/sites-available/gitlab \
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
