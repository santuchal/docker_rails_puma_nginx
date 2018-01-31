# Create Ruby On Rails app using docker puma postgres nginx 

Create Rails Docker file with nginx and puma

At first, create rails container and access it directly.
	
	$ mkdir sample_rails_nginx && cd $_

Directory for create Dockerfile

	$ mkdir -p docker/app

Docker file for rails

	$ nano docker/app/Dockerfile

		# Base ruby image:
		FROM ruby:2.3.3
		# Install ruby dependencies
		RUN apt-get update -qq && apt-get install -y build-essential libpq-dev nodejs
		# create application directory inside ruby image
		RUN mkdir /myapp
		# Set our working directory inside the image
		WORKDIR /myapp
		ADD Gemfile /myapp/Gemfile
		ADD Gemfile.lock /myapp/Gemfile.lock
		RUN bundle install
		ADD . /myapp
		EXPOSE 3000
		CMD [ "bundle", "exec", "puma", "-C", "config/puma.rb" ]

Create docker-compose file

$ nano docker-compose.yml
	
		version: '3'
		services:
		  app:
		    build:
		      context: .
		      dockerfile: ./docker/app/Dockerfile
		    volumes:
		      - .:/myapp
		    depends_on:
		      - db
		    ports:
		      - 3000:3000
		  db:
		    image: postgres


	$ bundle init

Add rails gem in Gemfile 

	$ nano Gemfile

	gem "rails"

Add Gemfile.lock

	$touch Gemfile.lock

Install gems and create rails project.

You may be asked if you wanna overwrite gemfile, then press ‘Y’.

	$ docker-compose run app bundle exec rails new . -d postgresql

Set db host to postgres container.
This is just sample, and define properly username and password.
	$ nano config/database.yml

	   default: &default
	   adapter: postgresql
	   encoding: unicode
	   host: db
	   username: postgres
	   pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

Note: Remember must remove/comment out username and password field in production section on database.yml

Now build docker image and create database.

	$ docker-compose build
	$ docker-compose run app bundle exec rails db:create RAILS_ENV=production

Run application.

	$ docker-compose up

Check your browser with http://localhost:3000 

Open another command window/tab.

Check all containers work well.
	

		$ docker-compose ps
		     Name                   Command               State           Ports
		--------------------------------------------------------------------------------
		samplerailsnginx_web_1   bundle exec puma -C config ...   Up      0.0.0.0:3000->3000/tcp
		samplerailsnginx_db_1    docker-entrypoint.sh postgres    Up      5432/tcp

Then check localhost:3000 which is Rails default welcome page.

Press ctrl+c to stop all those container.

Now add Nginx’s container

Create a directory which is related to nginx container.

	$ mkdir docker/web

Add nginx config file to proxy rails application.

	$ nano docker/web/app.conf

		upstream puma_rails_app {
		  server app:3000;
		}
		server {
		  listen       80;
		  proxy_buffers 64 16k;
		  proxy_max_temp_file_size 1024m;
		  proxy_connect_timeout 5s;
		  proxy_send_timeout 10s;
		  proxy_read_timeout 10s;
		  location / {
		    try_files $uri $uri/ @rails_app;
		  }
		  location @rails_app {
		    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		    proxy_set_header Host $http_host;
		    proxy_redirect off;
		    proxy_pass http://puma_rails_app;
		    # limit_req zone=one;
		    access_log /var/www/sample_rails_nginx/log/nginx.access.log;
		    error_log /var/www/sample_rails_nginx/log/nginx.error.log;
		  }
		}

Create docker file for nginx.

	$ nano docker/web/Dockerfile
	
		# Base nginx image:
		FROM nginx
		# Install nginx dependencies
		RUN apt-get update -qq && apt-get -y install apache2-utils
		# establish where Nginx should look for files
		ENV RAILS_ROOT /var/www/sample_rails_nginx
		# Set our working directory inside the image
		WORKDIR $RAILS_ROOT
		# create log directory
		RUN mkdir log
		# copy over static assets
		COPY public public/
		# Copy Nginx config template
		COPY docker/web/app.conf /tmp/docker_example.nginx
		# substitute variable references in the Nginx config template for real values from the environment
		# put the final config in its place
		RUN envsubst '$RAILS_ROOT' < /tmp/docker_example.nginx > /etc/nginx/conf.d/default.conf
		EXPOSE 80
		# Use the "exec" form of CMD so Nginx shuts down gracefully on SIGTERM (i.e. `docker stop`)
		CMD [ "nginx", "-g", "daemon off;" ]

Change docker-compose file to create nginx container.
Now add the nginx in docker-compose file
	$ nano docker-compose.yml


		version: '3'
		services:
		  web:
		    build:
		      context: .
		      dockerfile: ./docker/web/Dockerfile
		    depends_on:
		      - app
		    ports:
		      - 8080:80
		  app:
		    build:
		      context: .
		      dockerfile: ./docker/app/Dockerfile
		    volumes:
		      - .:/myapp
		    depends_on:
		      - db
			  - ports:
		      - 3000:3000
		  db:
		    image: postgres

Build and run the docker image

	$ docker-compose build
	$ docker-compose up

All containers work well.

		$ docker-compose ps
		     Name                   Command                     State          Ports
		---------------------------------------------------------------------------------
		samplerailsnginx_app_1   bundle exec puma -C config ...   Up     	 3000/tcp
		samplerailsnginx_db_1    docker-entrypoint.sh postgres    Up      	 5432/tcp
		samplerailsnginx_web_1   nginx -g daemon off;             Up       	 0.0.0.0:8080->80/tcp

Now check the localhost:8080, rails’s welcome page.

