Okay folks - going to give an overview of the Traefik 2.0 upgrade in case anyone wants to follow along or review later.
First, what is included in the upgrade? Here's the blog post with the rundown: https://blog.containo.us/traefik-2-0-6531ec5196c2

TL;DR: Full TCP support with SNI routing, Middlewares  including "chaining" (very cool), new dashboard/WebUI, Canary deployments, mirroring services with percentage of traefik sent there, new simplified certresolvers, but most importantly for some of you (but not Swarmites): new K8s IngressRouting CRD.
Oh, and all YAML config if desired.
So no more TOML necessary, but it is supported.

First - let's talk differences - everything is more cleanly laid out. There may be a couple bugs (or mistakes in my usage someone can identify), but otherwise I have all known use cases working.
I think so, yes @Darth-Penguini
Can you can "chain" "chains"
And you can.
The ONLY weakness is that I wish tls.certresolver could be defined in a middleware - but more on that later.

I'll start by explaining the traefik.yml file, highlighting differences.
It's important to note that almost every item has a way to be defined either in ENV variables, command parameters (i.e. --docker etc), or in the "STATIC CONFIG" file.
This static config file is what we used to name the traefik.toml (and I first replaced the old toml with new before upgrading (IMHO) to yaml).
Then, like before, there is dynamic file-based (or docker-based) configurations.
(or really any provider).

so everything can be done in env variables now? Including DNS01 validation magic? Yep.

What do you get from docker version?
I get
Client:
 Version:      17.09.1-ce
 API version:  1.32
 Go version:   go1.8.3
 Git commit:   e398b97
 Built:        Tue May 28 12:02:28 2019
 OS/Arch:      linux/amd64

Server:
 Version:      17.09.1-ce
 API version:  1.32 (minimum version 1.12)
 Go version:   go1.8.3
 Git commit:   e398b97
 Built:        Tue May 28 12:02:28 2019
 OS/Arch:      linux/amd64
 Experimental: false
So the highest composefile I can use (per https://docs.docker.com/compose/compose-file/) is 3.4
So I have all my stacks at version: "3.4" (FYI).
So - have you messed with Docker Swarm secrets?
The image (based on the Dockerfile) needs to be able to handle them, so they won't likely work on non-swarm designed images. But Traefik is.
So here is part of my traefik.yml:
# Traefik 2.0 Recipe
# /share/appdata/config/traefik/traefik.yml
version: "3.4"

secrets:
  namecheap_api_key:
    file: "/share/appdata/config/secrets/namecheap_api_key.secret"
So far - I just made a flat file at /share/appdata/config/secrets/namecheap_api_key.secret with the NAMECHEAP_API_KEY environment variable VALUE ONLY.



And then I have sourced or brought that file into the stack file / docker-compose as a secret called namecheap_api_key
Kinda like a better way to share yml files without the dummy env :slight_smile:
But not universally available.
Next:
services:
  app:
    image: traefik:latest
    secrets:
      - namecheap_api_key
Now I define traefik and I have to source the secret again.
Then my next two lines will help it make sense:
    environment:
      - NAMECHEAP_API_USER=gkoerk
      - NAMECHEAP_API_KEY_FILE=/run/secrets/namecheap_api_key
Notice that you need to change the ENV variable NAME in traefik to append "_FILE"

to reference the secret - but look where it is located....
And it is fully encrypted, and removed with the stack.
Only the flat file is unencrypted.
You can even pass TLS certs as secrets into Traefik 2.0!
Moving on:
    ports:
      - target: 80
        published: 80
        protocol: tcp
        mode: host
      - target: 443
        published: 443
        protocol: tcp
        mode: host
      - target: 8080
        published: 8090
        protocol: tcp
        mode: host
Nothing exciting to see here. Just being very explicit (more than I need to be I'm sure).
Notice I map 8090:8080 - this will come up later. This is technically only possible with an "insecure" dashboard.
Which I have disabled for now, but like to access via SERVER_IP:8090 when traefik isn't doing TLS right.
    volumes:
       - /etc/timezone:/etc/timezone:ro
       - /var/run/docker.sock:/var/run/docker.sock:ro
       - /share/appdata/traefik:/etc/traefik
Nothing too exciting. I will say I had to use the docker sock rather than the tcp: port.
    networks:
      - traefik_public
    command: --configFile=/etc/traefik/traefik-static.yaml
This is the only command I pass in.

... which is the reason you mount /etc/traefik ?

It is because traefik.yml is in the same folder, and by default looks for traefik.toml OR traefik.yaml
Well - since all my certs and configs for middlewares are there.
Yes
You still will have an acme.json, and I have a dynamic folder with middlewares, routers, services, etc.

and if you were doing all your configs with env variables, you wouldn't need it?
needed for traefik-static.yaml(toml) dynamic configuration yaml files, logs, and the critical acme.json

deploy:
  placement:
    constraints: [node.role == manager]
  labels:
And skipping the juicy dynamic config of traefik to explain the changes to traefik.toml (now traefik-static.yaml in my config) :slight_smile:
So on to traefik-static.yml.
Everything now has a home or namespace path.
So no more floating things at the top and bottom.
This is what makes yaml work so well and the docker labels.
@here You guys don't mind code-reviewing as I explain, especially when I say I have no idea why I did that, do you?
# Traefik Static Configuration
# Host Path: /share/appdata/config/traefik/traefik-static.yaml
# Internal Path: /etc/traefik/traefik-static.yaml

global:
checkNewVersion: true
Simple so far.
serversTransport:
insecureSkipVerify: true
Carried over from before. Assumed I needed it, no idea what it does.

-- It lets you connect to backends via HTTPS even if they have invalid certs
(Like good 'ol Unifi Controller)

ALSO - PAY CLOSE ATTENTION TO CAPITALIZATION IN TRAEFIK 2.0
Ahh cool.
Moving along.
Scared to show my ignorance but oh well...
entryPoints:
  http:
    address: ":80"
    # Trust IPv4 Private Address Space
    forwardedHeaders:
      trustedIPs:
      - "172.16.0.0/12"
      - "10.0.0.0/8"
      - "192.168.0.0/16"
That is how you define the http entrypoint now.
You can name it bobs-unsecured-port if you like.
or web vs web-secure
Not sure about my trustedIPs here.

aah, point of reference - you have to use host mode ports if you're going to do that, to bypas swarm routing mesh
if you don't, you'll end up whitelisting all incoming traffic, which passes through swarm and gets source NAT'd to a 10.x.x.x address


Okay - Bit more then I need a brief dinner break as the wife is calling.
  https:
    address: ":443"
    # Trust IPv4 Private Address Space
    forwardedHeaders:
      trustedIPs:
      - "172.16.0.0/12"
      - "10.0.0.0/8"
      - "192.168.0.0/16"
You can tell me if my trustedIPs are dangerous.
For discussion:
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    # Alternative endpoint:
    # endpoint: "tcp://127.0.0.1:2375"
    watch: true
    swarmMode: true
    network: traefik_public
    # Optional defaultRule: "Host(`{{ .Name }}.localhost`)"
    useBindPortIP: false
    exposedByDefault: false

  file:
    # Optional instead of directory:
    # filename: /etc/traefik/traefik-dynamic.yaml
    directory: /etc/traefik/dynamic
    watch: true
    debugLogGeneratedTemplate: true

    New defaultRule!
    api:
      dashboard: true
      insecure: true
      debug: true
    insecure is for dev/test only (so says Traefik),
    metrics:
      prometheus:
        buckets:
        - "0.1"
        - "0.3"
        - "1.2"
        - "5"
        addEntryPointsLabels: true
        addServicesLabels: true
        entryPoint: metrics
    Super cool - not sure how to use it yet :slight_smile:
    ping:
      entryPoint: ping

    log:
      level: ERROR
      filePath: "/etc/traefik/traefik.log"

    accessLog:
      filePath: "/etc/traefik/access.log"
    Same old thing, just cleaner.
    Last bit:

    certificatesResolvers:
  namecheap:
    acme:
      email: "gkoerk@gmail.com"
      storage: "/etc/traefik/acme.json"
      # Alternative ACME Staging CA Server (not ratelimited like prod):
      # caServer: "https://acme-staging-v02.api.letsencrypt.org/directory"
      dnsChallenge:
        provider: namecheap
        delayBeforeCheck: 0
        resolvers:
        - "1.1.1.1:53"
        - "8.8.8.8:53"
This is the new certificatesResolvers - you can have many.
Of course - you don't have to store in acme.json - if fact I will be asking if the middlewares provide more flexibility if you define TLS certs there.
That is THE END of traefik-static.ynl
The rest is ALL dynamic.
Whether in file or in docker labels. Kapeesh???


NOTE: In case I forget, acme.json has a new format. You need to either pull new certs or use a converter Traefik provides.

All the rest of these files are in /share/appdata/config/traefik/dynamic/
But they could easily be in one file (in fact, this file):
  file:
    # Optional instead of directory:
    # filename: /etc/traefik/traefik-dynamic.yaml
First up, we have static-servers.yaml (but KEEP IN MIND you decide how to break these up and I may change them for convenience, etc.
# Traefik Dynamic Configuration
# Services: Static (Non-Docker) Services
# Host Path: /share/appdata/config/traefik/dynamic/static-servers.yaml
# Internal Path: /etc/traefik/dynamic/static-servers.yaml

http:
  services:
    noop:
      loadBalancer:
        servers:
          - url: "http://127.0.0.1"

    gknas:
      loadBalancer:
        servers:
          - url: "http://192.168.1.100:8880"
I have two services so far - noop and gknas (shout out to my NAS!)
Now a piece I largely believe unnecessary, but demonstrative, is the static-routers.yml
# Traefik Dynamic Configuration
# Routers: Static (Non-Docker) Routers
# Host Path: /share/appdata/config/traefik/dynamic/static-routers.yaml
# Internal Path: /etc/traefik/dynamic/static-routers.yaml
http:
  routers:
    gkoerk:
      entryPoints:
        - https
      middlewares:
        - forward-auth
      service: gknas
      rule: "Host(`gkoerk.com`)"
      tls:
        certResolver: namecheap
        domains:
          - main: "*.gkoerk.com"
            sans:
              - gkoerk.com

    qnapunofficial:
      entryPoints:
        - https
      middlewares:
        - forward-auth
      service: noop
      rule: "Host(`qnapunofficial.com`)"
      tls:
        certResolver: namecheap
        domains:
          - main: "*.qnapunofficial.com"
            sans:
              - qnapunofficial.com

    oracleoptimizer:
      entryPoints:
        - https
      middlewares:
        - forward-auth
      service: noop
      rule: "Host(`oracleoptimizer.com`)"
      tls:
        certResolver: namecheap
        domains:
          - main: "*.oracleoptimizer.com"
            sans:
              - oracleoptimizer.com

    gknas:
      entryPoints:
        - https
      middlewares:
        - forward-auth
      service: gknas
      rule: "Host(`gknas.gkoerk.com`)"
      tls:
        certResolver: namecheap
        domains:
          - main: "*.gkoerk.com"
            sans:
              - gkoerk.com
Note that there are entrypoints in traefik (ports/protocols) which are read by rules.
There are services which are like the old backends.
Routers are LIKE ye olde frontends... but more robust. EVERYTHING is defined in a router.
A router rules all.
And then there are middlewares which act on headers in flight.
Or redirect, etc.
The file above was built as I needed a route to my NAS, and because I wanted traefik pulling a wildcard cert from every domain I own.
The first three are absolutely NOT NEEDED in my configuration, but demonstrative only.
Actually - amendment time.
I think if you remove these, you MAY need to specify a certresolver in a router WITH the domains and sans first.
After they are there, they use the host rule or SNI rule to decide which cert satisfies.
So my new static-routers.yaml is:
# Traefik Dynamic Configuration
# Routers: Static (Non-Docker) Routers
# Host Path: /share/appdata/config/traefik/dynamic/static-routers.yaml
# Internal Path: /etc/traefik/dynamic/static-routers.yaml

http:
  routers:
    gknas:
      entryPoints:
        - https
      middlewares:
        - forward-auth
      service: gknas
      rule: "Host(`gknas.gkoerk.com`)"
      tls:
        certResolver: namecheap
        domains:
          - main: "*.gkoerk.com"
            sans:
              - gkoerk.com
Next up we have some middlewares. Again these can all be in one file or otherwise.
basic-user.yaml
# Traefik Dynamic Configuration
# Middleware: Basic Authentication
# Host Path: /share/appdata/config/traefik/dynamic/basic-user.yaml
# Internal Path: /etc/traefik/dynamic/basic-user.yaml

http:
  middlewares:
    basic-user:
      basicAuth:
        users:
          - "admin:<bcrypt_password_from_htpasswd_command>"
Pretty simple?

compress.yaml (unused for now):
# Traefik Dynamic Configuration
# Middleware: Compression
# Host Path: /share/appdata/config/traefik/dynamic/compress.yaml
# Internal Path: /etc/traefik/dynamic/compress.yaml

http:
  middlewares:
    compress:
      compress: {}
https-redirect.yaml:
# Traefik Dynamic Configuration
# Middleware: HTTPS Redirect
# Host Path: /share/appdata/config/traefik/dynamic/https-redirect.yaml
# Internal Path: /etc/traefik/dynamic/https-redirect.yaml

http:
  middlewares:
    https-redirect:
      redirectScheme:
        scheme: https


        forward-auth.yaml:
        # Traefik Dynamic Configuration
        # Middleware: Forward Auth
        # Host Path: /share/appdata/config/traefik/dynamic/forward-auth.yaml
        # Internal Path: /etc/traefik/dynamic/forward-auth.yaml

        http:
          middlewares:
            forward-auth:
              forwardAuth:
                address: "http://192.168.1.100:8080/authorize"
                trustForwardHeader: true
                authResponseHeaders:
                  - X-FORWARDAUTH-NAME
                  - X-FORWARDAUTH-SUB
                  - X-FORWARDAUTH-EMAIL
        Yours will be a bit different on the ResponseHeaders.
        Now some deep confusion...
        One more middleware (a chain) called secured-chain.yaml:
        # Traefik Dynamic Configuration
        # Middleware: Secured Chain (Testing)
        # Host Path: /share/appdata/config/traefik/dynamic/secured-chain.yaml
        # Internal Path: /etc/traefik/dynamic/secured-chain.yaml

        http:
          middlewares:
            secured:
              chain:
                middlewares:
                  - https-redirect
                  - forward-auth
        Does NOT seem to work.
        AT ALL.
        Regardless of the order I specify them in.
        So for me to get http-> https redirection I need one more file:
        global-redirect.yaml:
        # Traefik Dynamic Configuration
        # Routers: Global HTTP -> HTTPS Redirect
        # Host Path: /share/appdata/config/traefik/dynamic/globalredirect.yaml
        # Internal Path: /etc/traefik/dynamic/globalredirect.yaml

        http:
          routers:
            https-redirect:
              entryPoints:
                - http
              middlewares:
                - https-redirect
              rule: "HostRegexp(`{host:[a-z-.]+}`)"
              service: noop
        The rest is in docker.
        Cool to proceed or anyone know about the chain middlewares?
        So that router will be used by almost EVERY service.
        The one BAD part to this cleanness is like XML - it is wordy as crap in the docker labels.
        Technically you need 2 (two) routers per service.
        That is the gospel from Containous so far.
        So I made one VERY generic.

        That goes nowhere (gets short circuited by the redirect).
        But on to docker labels:

        traefik_app (in traefik.yml) labels:
              labels:
                - "traefik.enable=true"
                - "traefik.http.routers.traefik.entrypoints=https"
                - "traefik.http.routers.traefik.rule=Host(`traefik.gkoerk.com`)"
                - "traefik.http.routers.traefik.tls.certresolver=namecheap"
                - "traefik.http.routers.traefik.middlewares=forward-auth@file"
                - "traefik.http.services.traefik.loadbalancer.server.port=8080"
        This is a SECOND router used to get to Traefik.
        In fact - this router is called traefik (.traefik.)
        It only listens on https
        I tried allowing it to listen to both.
        And use: - "traefik.http.routers.traefik.middlewares=secured@file"
        But nope.
        Even in the examples in the forums they show an http and https router.
        I think they have an issue to fix this, however, somehow.   By leaving you on the same router.

        So my plex.yml looks like this (labels on plex_app):
            deploy:
              labels:
                - "traefik.enable=true"
                - "traefik.http.routers.plex.entrypoints=https"
                - "traefik.http.routers.plex.rule=Host(`plex.gkoerk.com`)"
                - "traefik.http.routers.plex.tls.certresolver=namecheap"
                #- "traefik.http.routers.plex.middlewares=forward-auth@file"
                - "traefik.http.services.plex.loadbalancer.server.port=32400"
        Because it has strong auth.
        The TROUBLE - is allowing a single service you WANT exposed on http!
        Since I need a rule that excludes it.
        Here is my use case:
          synclounge:
            image: starbix/synclounge:latest
            ports:
              - 8088:8088
              - 8089:8089
            environment:
              - DOMAIN=lounge.gkoerk.com
            networks:
              - traefik_public
            deploy:
              labels:
                - "traefik.enable=true"
                - "traefik.http.routers.synclounge.entrypoints=http"
                - "traefik.http.routers.synclounge.rule=Host(`synclounge.gkoerk.com`)"
                - "traefik.http.services.synclounge.loadbalancer.server.port=8088"
                #- "traefik.http.routers.slserver.entrypoints=http"
                #- "traefik.http.routers.slserver.rule=Host(`slserver.gkoerk.com`)"
                #- "traefik.http.services.slserver.loadbalancer.server.port=8089"
        That is in my plex.yml
