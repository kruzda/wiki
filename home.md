<!-- TITLE: Wiki Home -->
<!-- SUBTITLE: wiki.kuk.ac home page -->

# Welcome

This wiki was written within a node.js application called Wiki.js ( https://github.com/Requarks/wiki ). I wrote the following docker-compose file based on the one that they provide as an example here: https://github.com/Requarks/wiki/blob/master/tools/docker-compose.yml

```
version: '3.1'
services:
  wikidb:
    image: mongo
    expose:
      - '27017'
    command: '--smallfiles'
    volumes:
      - data-volume:/data/db
  wikijs:
    image: 'requarks/wiki:latest'
    links:
      - wikidb
    expose:
      - '3000'
    environment:
      WIKI_ADMIN_EMAIL: admin@wiki.kuk.ac
    volumes:
      - .\config.yml:/var/wiki/config.yml
      - wikijs-persistent-storage:/mnt/wikijs-persistent-storage
  wikiweb:
    image: nginx
    links:
      - wikijs
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - .\wikijs.conf:/etc/nginx/conf.d/wikijs.conf
      - .\wikijs-privkey.pem:/etc/letsencrypt/live/wiki.kuk.ac/privkey.pem
      - .\wikijs-fullchain.pem:/etc/letsencrypt/live/wiki.kuk.ac/fullchain.pem

volumes:
  data-volume:
  wikijs-persistent-storage:
```

This shows how there are multiple services interconnected here:
* the application is connected to a mongodb database where for instance the credentials limiting access to the admin interface are stored
* nginx sits in front and is configured to send all enquiries to the application listening on port `3000` - the main use of this is to allow for TLS encryption

Wiki.js is configured via its `config.yaml` file, where settings such as the database connection are defined:

```
# ---------------------------------------------------------------------
# Database Connection String
# ---------------------------------------------------------------------
# You can also use an ENV variable by using $ENV_VAR_NAME as the value

db: mongodb://wikidb:27017/wiki
```

Along with the Git repository defintions:

```
# ---------------------------------------------------------------------
# Git Connection Info
# ---------------------------------------------------------------------

git:
  url: git@github.com:kruzda/wiki.git
  branch: master
  auth:

    # Type: basic or ssh
    type: ssh

    # Only for SSH authentication:
    privateKey: /mnt/wikijs-persistent-storage/github-privkey.pem

    sslVerify: true

  # Default email to use as commit author
  serverEmail: wiki@kuk.ac

  # Whether to use user email as author in commits
  showUserEmail: false
```

nginx is configured the following way:

```
server {
	listen 80 default_server;
	server_name wiki.kuk.ac;
	return 301 https://$server_name$request_uri;
}

server {
	listen 443;
	server_name wiki.kuk.ac;

	ssl on;
	ssl_certificate /etc/letsencrypt/live/wiki.kuk.ac/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/wiki.kuk.ac/privkey.pem;

	location / {
		proxy_pass http://wikijs:3000;
	}
}

```

This configuration file, the SSL certificate and the private key are all provided to nginx via the file-based `volumes` attribute. Everything gets redirected to HTTPS to make use of the encryption provided by TLS.

There were plenty of issues encountered during the implementation of this container, most of which came from the lack of proper support for Windows by Docker. The main issue seems to have been the inability to change permissions of files within a volume attachment that had its source on the Windows filesystem. When the volume is a named volume - so is presumably created on the virtual machine running Docker (boot2docker) - the issue no longer persists.