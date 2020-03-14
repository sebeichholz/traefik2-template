# traefik2-template

I have been using traefik [v1](https://docs.traefik.io/v1.7/) together with [letsencrypt](https://letsencrypt.org/) for [more than two years](https://www.facebook.com/Oregami.org/photos/a.193090257436658/1598428220236181/?type=1&theater) now (current date is 2020-03-14).

Last week I got an email from letsencrypt which said:

> According to our records, the software client you're using to get Let's
Encrypt TLS/SSL certificates issued or renewed at least one HTTPS certificate
in the past two weeks using the ACMEv1 protocol.  [...]
> Beginning June 1, 2020, we will stop allowing new domains to validate using
the ACMEv1 protocol. You should upgrade to an ACMEv2 compatible client before
then, or certificate issuance will fail. For most people, simply upgrading to
the latest version of your existing client will suffice.

Well, as I said above I was using traefik together with letsencrypt for automatic renewal of SSL certificates. You should also know that I serving multiple websites on my server, and that I am using docker with namespaces for [security reasons](https://docs.docker.com/engine/security/userns-remap/). Using docker with namespaces forced me to use a complete static traefik v1 config, using docker compose with automatic detection of services is not possible.

Sadly a "simple update" to traefik v2 was not that simple. Some major principles [changed](https://docs.traefik.io/migration/v1-to-v2/) and I therefore had to invest quiet a few hours to figure out how to get things working.

So this repository contains the following things:

- traefik v2 config template with two TOML files (traefik.toml and traefik-sites.toml)
  - automatic redirection from HTTP to HTTPS
  - automatic SSL certificate generation with letsencrypt (ACMEv2 protocol as v1 support will go away soon)
- for a little more comfort a libre office calc document is included where you only need to enter the name of the docker container and the domain name and the relevant traefik config is generated
- docker tipps for using the whole thing with namespaced docker and minimum capabilities (both for security reasons)

## traefik templates

### traefik.toml
	[log]
	  level = "DEBUG"
	[entryPoints]
	  [entryPoints.http]
		address = ":80"
	  [entryPoints.https]
		address = ":443"
	[providers.file]
	  filename = "/etc/traefik/traefik-sites.toml"
	[certificatesResolvers.letsencrypt.acme]
	  email = "mymail@domain1.xy"
	  storage = "acme.json"
	  [certificatesResolvers.letsencrypt.acme.httpChallenge]
		entryPoint = "http"
    
    
### traefik-sites.toml

	[http]
	  # Redirect to https
	  [http.middlewares]
		[http.middlewares.httpsredirect.redirectScheme]
		  scheme = https
		  permanent = true
	  # routers
	  [http.routers]
		[http.routers.router_[domain1docker]_http]
		   entrypoints = ['http']
		   service = 'service-[domain1docker]'
		   middlewares = ['httpsredirect']
		   rule = 'host(`[domain1serverURL]`)'
		 [http.routers.router_[domain1docker]_https]
		   entrypoints = ['https','http']
		   service = 'service-[domain1docker]'
		   rule = 'host(`dev.domain1.xy`)'
		   [http.routers.router_[domain1docker]_https.tls]
			 certresolver = 'letsencrypt'
			 [[http.routers.router_[domain1docker]_https.tls.domains]]
			 main = '[domain1serverURL]'
       
		# Add the services
		[http.services]
		  [http.services.service-domain1]
			[http.services.service-domain1.loadBalancer]
			  [[http.services.service-domain1.loadBalancer.servers]]
				url = 'http://[domain1docker]'

## using the template files
- change the mail address in traefik.toml to your address
- replace [domain1serverURL] in traefik-sites.toml with the docker container name (must not contain '.')
- replace [domain1serverURL] in traefik-sites.toml with the website mein URL (e.g. www.mydomain.xy)


## docker tipps

Here are some examplary docker commands which I used to try out my config with two subdomains:

    #first star two simple nginx websites. Note the docker container names "domain1" and "domain2".
    docker run --name domain1 -d --memory=100m -v /var/www/v1:/usr/share/nginx/html:ro -d nginx:1.13-alpine
    docker run --name domain2 -d --memory=100m -v /var/www/v1:/usr/share/nginx/html:ro -d nginx:1.13-alpine

    #create and start traefik with docker and minimal capabilities
    docker run --name traefik -d -p 80:80 -p 443:443 \
      --memory=100m --cap-drop=all --cap-add=NET_BIND_SERVICE \
      -v /docker/traefik/traefik.toml:/etc/traefik/traefik.toml \
      -v /docker/traefik/traefik-sites.toml:/etc/traefik/traefik-sites.toml \
      traefik:2.1.6

    #create docker network for internal communication and connect everything:
    docker network create traefik
    docker network connect traefik domain1
    docker network connect traefik domain2
    docker network connect traefik traefik

Have fun!
