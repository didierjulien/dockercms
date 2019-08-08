# dockercms
Local Docker stack using docker-compose. Includes PHP-Apache, MySQL, Adminer and NodeJS containers. Initially created to work on WordPress locally.  
If you have any experience running Docker, you can skip to section 1.

# 0. The basics of running WordPress with Docker
Let's start by fetching the WordPress container from the Docker Hub.
```
https://hub.docker.com/_/wordpress
```
You can run the official container with this line:
```
docker run --name some-wordpress --network some-network -d wordpress
```
There is a good chance that didn't work and you instead got fed this error message:  
"docker: Error response from daemon: network some-network not found."  
All hope is lost, WordPress doesn't support Docker!  
But how about a small tweak to the command:
```
docker run --name some-wordpress -d wordpress
```
Okay, that seemed to work but I don't see my site. To access it you would need to go through the IP docker is using to run on your machine. This is getting awkward... Let's check further on the docker hub.
```
docker run --name good-wordpress -p 8080:80 -d wordpress
```
Now we're getting somewhere! I can access the site now from localhost:8080 as defined by the modifier "-p". Start up the 5 minute install and let's go!

![Wordpress install](https://i2.wp.com/wordpress.org/support/files/2018/10/install-step3_v47.png)


Seems we've hit another snag. I don't know what the database credentials are and there doesn't seem to be any default creds.
Thankfully, WordPress provide a docker-compose configuration file to help us out a little.
```
version: '3.1'

services:

  wordpress:
    image: wordpress
    restart: always
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: exampleuser
      WORDPRESS_DB_PASSWORD: examplepass
      WORDPRESS_DB_NAME: exampledb

  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_DATABASE: exampledb
      MYSQL_USER: exampleuser
      MYSQL_PASSWORD: examplepass
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
```

We can see that the WordPress DB information is clearly defined and that this stack adds an external mysql container to run alongside our WordPress.  
First, you're going to need to stop the running container we created earlier. Find it and kill it using this command
```
docker ps -a
docker kill [CONTAINER ID or NAME]
docker rm [CONTAINER ID or NAME]
```
Now let's run the docker-compose stack and see what happens:
```
docker-compose -f stack.yml up
```
Since the Database information was already defined in the compose file, there is no need to go through the DB configuration step of the installation. Just enter the site and user information and you can now access the classic WordPress site admin section.   
This part should now feel very familiar, you've been here before, you know where the washroom is and you recognize your favorite spot.  
You can install your themes and your plugins easily and start entering some content. There are 2 major issues however:  
1. Where do I put my custom code?
2. If I stop and remove the containers, I will lose all my work and have to start over.   

We're going to have to build our own WordPress stack.

# 1.Breaking down the Stack
### 1.1 Defining Services
We'll have to start with a PHP container. I'll be using the one created from the Dockerfiles in this repository.
```
FROM php:7.2.20-apache-buster

WORKDIR /var/www/html/
# CMD [ "php", "./index.php" ]

RUN apt-get update && apt-get upgrade -y && apt-get install -y -q\
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libssl-dev \
        libcurl4-openssl-dev \
        pkg-config \
        libmcrypt-dev \
        libpng-dev \
        mysql-client \
        libxml2-dev \
    && docker-php-ext-install -j$(nproc) iconv mbstring zip curl json mysqli pdo_mysql simplexml xml \
    && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-install -j$(nproc) gd \
    && a2enmod proxy \
    && a2enmod proxy_http \
    && a2enmod proxy_balancer \
    && a2enmod lbmethod_byrequests \
    && a2enmod proxy_ftp \
    && a2enmod proxy_connect \
    && a2enmod proxy_ajp \
    && a2enmod proxy_wstunnel \
    && a2enmod headers \
    && a2enmod rewrite \
    && a2enmod ssl \
    && curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
    && mv wp-cli.phar /usr/local/bin/wp \
    && chmod 777 /usr/local/bin/wp \
    && openssl req -new -newkey rsa:2048 -nodes -out localhost.csr -keyout localhost.key -subj "/C=CA/ST=Quebec/L=Montreal/O=myself/OU=Montreal/CN=localhost" \
    && openssl x509 -req -sha256 -days 365 -in localhost.csr -signkey localhost.key -out localhost.crt \
    && mv localhost.* /root \
    && service apache2 restart \
    && rm -rf /var/cache/apt/* /tmp/*
```
This will install all the PHP packages to run WordPress smoothly. Any additional dependencies that get added to WP can also be added here as needed.   
Lines 33-35 also generate a self-signed certificate tied to "localhost". To change the domain name associated to the certs, simply change /CN=localhost to whatever works for you and rebuild the image (Remember to update the docker-compose.yml file with the new image).   

We won't be building a MySQL container as there is not much customization needed and we will be mounting the configurations from a local file.   

The essential services are then:
```
version: "3.7"

services:

  html:
    image: didierjulien/phpweb:7.2
    container_name: WP_Local_PHP
    ports:
      - "2080:80"
      - "2443:443"
    volumes:
      - ./html:/var/www/html:cached
      - ./conf/php/php.ini:/usr/local/etc/php/php.ini:cached
      - ./conf/web/apache2.conf:/etc/apache2/apache2.conf:cached
      - ./conf/web/000-default.conf:/etc/apache2/sites-available/000-default.conf:cached

    restart: always
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_USER: dbdata
      WORDPRESS_DB_PASSWORD: dbdata

  mysql:
    image: mysql:5.7.27
    container_name: WP_Local_DB
    expose:
      - "3306"
    volumes:
      - ./sql/data:/var/lib/mysql:delegated
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: dbdata
      MYSQL_DATABASE: dbdata
      MYSQL_USER: dbdata
      MYSQL_PASSWORD: dbdata

volumes:
  sql:
  html:
  conf:
```

### 1.2 Mounting files and data persistence
Lines 11-15 and line 29 define the volume mounts to be used in the corresponding containers.
```
volumes:
  - ./html:/var/www/html:cached
  - ./conf/php/php.ini:/usr/local/etc/php/php.ini:cached
  - ./conf/web/apache2.conf:/etc/apache2/apache2.conf:cached
  - ./conf/web/000-default.conf:/etc/apache2/sites-available/000-default.conf:cached

...

  - ./sql/data:/var/lib/mysql:delegated
```
- html/ is where you can drop in the CMS PHP files. This is where you'll be doing all your code changes and moving all your files to and from. This mount ensures that the themes, plugins and media that you download from the dashboard get copied to your local machine.

- conf/ is where we can define the different configurations for PHP and Apache. The php.ini configs are very loose as is so there shouldn't be many changes to be made but your mileage may vary.

- sql/data/ is our database persistence folder. If you want to start over with a fresh site, you can delete everything in data/ but beware!

At the bottom of the yml file, we need to define the volumes used by the stack - usually defined by the top directory.
```
volumes:
    sql:
    html:
    conf:
```

### 1.3 Caching options for mounted volumes
These affect the view of the mounted files for the host(your machine) and/or the container. There are 3 possible options:
- consistent: the default option, this allows the container to view the files in real time with the host. On Linux-based machines, this option works well and does not incur much of a performance penalty. On any other OS however, the FS overlay's overhead will be felt, especially on I/O heavy operations.

- cached: updates on the host will be reflected in the container after a delay. Updating a PHP file for example will not have an immediate effect at first but with time, most files will be cached and the changes will happen more quickly. This is ideal when working on non-Linux machines.

- delegated: updates on the container will not be reflected immediately on the host. This is ideal for MySQL since the PHP container interacts with the data in the container and not on your machine; so even if it takes a little while to have the copy on your computer, you're not slowing down the WordPress queries.

### 1.4 Defining Environment Variables
One of the issues we'll run into by simply running the WordPress container as shown on the docker hub is connecting to the database. The default credentials aren't defined anywhere so the setup cannot continue.   
On Lines 18-21 and 31-35 we can define the environment variables needed to point our WordPress installation to the Database, and to give that DB the connection credentials we decide.
```
environment:
  WORDPRESS_DB_HOST: mysql
  WORDPRESS_DB_USER: dbdata
  WORDPRESS_DB_PASSWORD: dbdata

  ...

environment:
  MYSQL_ROOT_PASSWORD: dbdata
  MYSQL_DATABASE: dbdata
  MYSQL_USER: dbdata
  MYSQL_PASSWORD: dbdata
```
Reference the Docker Hub for both PHP and MySQL for the exhaustive list of support ENV variables.

# 2. Working with the Stack
### 2.1 Getting started
Once the repo has been cloned, there is little else to do than to launch docker-compose:
```
docker-compose up
```
or to run in in the background
```
docker-compose up -d
```
If you've downloaded the images previously but need the updated version, do
```
docker-compose up --force-recreate
```

The required images will be downloaded and run immediately, PHP takes only a few seconds and MySQL less than 30 usually.   
By default, the images used are on my Docker Hub, but I encourage building and tagging your own images to maintain a stable working version. I will be regularly updating alongside the official PHP image.

### 2.2 Creating MySQL container
We have data persistence with the MySQL container but that is only on our local machine. Deleting those files will put the site back to it's default starting state. If we want to capture a snapshot of our database, we can use the Dockerfile located in /sql.
```
FROM mysql

ADD data.sql /docker-entrypoint-initdb.d
```
Simply export the local database to sql/data.sql. Then from the /sql directory run
```
docker build -t your-repository/mysql:[date] .
```
for example
```
docker build -t didierjulien/mysql:2019.08.08
```
Then the image will have to be pushed to your repository
```
docker push your-repository/mysql:[date]
```
I do __NOT__ recommend pushing database containers to public repositories, hence why I don't do it myself. It's best to keep a local copy or push to a private and secure repo.  
Once we have this image, we'll need to modify the docker-compose.yml to use it
```
mysql:
  image: your-repository/mysql:[date]
```
To test that this worked, we can move all the files out of /sql/data, including hidden ones. Then rebuild the stack. MySQL provisioning should take a little longer as it writes the files to your local machine, but once it's done, you should be able to navigate to your local site and have exactly the same state as the export. 
