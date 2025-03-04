![Souin logo](https://github.com/darkweak/souin/blob/master/docs/img/logo.svg)

# Souin Table of Contents
1. [Souin reverse-proxy cache](#project-description)
2. [Configuration](#configuration)  
  2.1. [Required configuration](#required-configuration)  
    2.1.1. [Souin as plugin](#souin-as-plugin)  
    2.1.2. [Souin out-of-the-box](#souin-out-of-the-box)  
  2.2. [Optional configuration](#optional-configuration)
3. [APIs](#apis)  
  3.1. [Souin API](#souin-api)  
  3.2. [Security API](#security-api)
4. [Diagrams](#diagrams)  
  4.1. [Sequence diagram](#sequence-diagram)
5. [Cache systems](#cache-systems)
6. [Examples](#examples)  
  6.1. [Træfik container](#træfik-container)
7. [Plugins](#plugins)  
  7.1. [Caddy module](#caddy-module)  
  7.2. [Echo middleware](#echo-middleware)  
  7.2. [Gin middleware](#gin-middleware)  
  7.2. [Træfik plugin](#træfik-plugin)  
  7.2. [Tyk plugin](#tyk-plugin)  
  7.3. [Prestashop plugin](#prestashop-plugin)  
  7.4. [Wordpress plugin](#wordpress-plugin)  
8. [Credits](#credits)

[![Travis CI](https://travis-ci.com/Darkweak/Souin.svg?branch=master)](https://travis-ci.com/Darkweak/Souin)

# Souin HTTP cache

## Project description
Souin is a new HTTP cache system suitable for every reverse-proxy. It can be either placed on top of your current reverse-proxy whether it's Apache, Nginx or as plugin in your favorite reverse-proxy like Træfik, Caddy or Tyk.  
Since it's written in go, it can be deployed on any server and thanks to the docker integration, it will be easy to install on top of a Swarm, or a kubernetes instance.  
It's RFC compatible, supporting Vary, request coalescing, stale cache-control and other specifications related to the [RFC-7234](https://tools.ietf.org/html/rfc7234).  
It also supports the [Cache-Status HTTP response header](https://httpwg.org/http-extensions/draft-ietf-httpbis-cache-header.html) and the YKey group such as Varnish.

## Disclaimer
If you need redis or other custom cache providers, you have to use the fully-featured version. You can read the documentation, on [the fully-featured branch](https://github.com/Darkweak/Souin/tree/full-version) to understand the specific parts.

## Configuration
The configuration file is store at `/anywhere/configuration.yml`. You can supply your own as long as you use one of the minimal configurations below.

### Required configuration
#### Souin as plugin
```yaml
default_cache: # Required
  ttl: 10s # Default TTL
```

#### Souin out-of-the-box
```yaml
default_cache: # Required
  ttl: 10s # Default TTL
reverse_proxy_url: 'http://traefik' # If it's in the same network you can use http://your-service, otherwise just use https://yourdomain.com
```

|  Key                           |  Description                                                       |  Value example                                                                                                            |
|:------------------------------:|:------------------------------------------------------------------:|:-------------------------------------------------------------------------------------------------------------------------:|
| `default_cache.ttl`            | Duration to cache request (in seconds)                             | 10                                                                                                                        |
| `reverse_proxy_url`            | The reverse-proxy's instance URL (Apache, Nginx, Træfik...)        | - `http://yourservice` (Container way)<br/>`http://localhost:81` (Local way)<br/>`http://yourdomain.com:81` (Network way) |

### Optional configuration
```yaml
# /anywhere/configuration.yml
api:
  basepath: /souin-api # Default route basepath for every additional APIs to avoid conflicts with existing routes
  security: # Secure your APIs
    secret: your_secret_key # JWT secret key
    enable: true # Required to enable the endpoints
    users: # Users declaration
      - username: user1
        password: test
  souin: # Souin listing keys and cache management
    security: true # Enable JWT Authentication token
    enable: true # Enable the endpoints
cdn: # If Souin is set after a CDN fill these informations
  api_key: XXXX # Your provider API key if mandatory
  provider: fastly # The provider placed before Souin (e.g. fastly, cloudflare, akamai, varnish)
  strategy: soft # The strategy to purge the CDN cache based on tags (e.g. soft, hard)
  dynamic: true # If true, you'll be able to add custom keys than the ones defined under the surrogate_keys key
default_cache:
  distributed: true # Use Olric distributed storage
  headers: # Default headers concatenated in stored keys
    - Authorization
  olric: # If distributed is set to true, you'll have to define the olric section
    url: 'olric:3320' # Olric server
  regex:
    exclude: 'ARegexHere' # Regex to exclude from cache
  stale: 1000s # Stale duration
  ttl: 1000s # Default TTL
log_level: INFO # Logs verbosity [ DEBUG, INFO, WARN, ERROR, DPANIC, PANIC, FATAL ], case do not matter
ssl_providers: # The {providers}.json to use
  - traefik
urls:
  'https:\/\/domain.com\/first-.+': # First regex route configuration
    ttl: 1000s # Override default TTL
  'https:\/\/domain.com\/second-route': # Second regex route configuration
    ttl: 10s # Override default TTL
    headers: # Override default headers
    - Authorization
  'https?:\/\/mysubdomain\.domain\.com': # Third regex route configuration
    ttl: 50s # Override default TTL
    headers: # Override default headers
    - Authorization
    - 'Content-Type'
ykeys:
  The_First_Test:
    headers:
      Content-Type: '.+'
  The_Second_Test:
    url: 'the/second/.+'
  The_Third_Test:
  The_Fourth_Test:
surrogate_keys:
  The_First_Test:
    headers:
      Content-Type: '.+'
  The_Second_Test:
    url: 'the/second/.+'
  The_Third_Test:
  The_Fourth_Test:
```

|  Key                                              |  Description                                                                                   |  Value example                                                                                                          |
|:-------------------------------------------------:|:----------------------------------------------------------------------------------------------:|:-----------------------------------------------------------------------------------------------------------------------:|
| `api`                                             | The cache-handler API cache management                                                         |                                                                                                                         |
| `api.basepath`                                    | BasePath for all APIs to avoid conflicts                                                       | `/your-non-conflicting-route`<br/><br/>`(default: /souin-api)`                                                          |
| `api.{api}.enable`                                | Enable the new API with related routes                                                         | `true`<br/><br/>`(default: false)`                                                                                      |
| `api.{api}.security`                              | Enable the JWT Authentication token verification                                               | `true`<br/><br/>`(default: false)`                                                                                      |
| `api.security.secret`                             | JWT secret key                                                                                 | `Any_charCanW0rk123`                                                                                                    |
| `api.security.users`                              | Array of authorized users with username x password combo                                       | `- username: admin`<br/><br/>`  password: admin`                                                                        |
| `api.souin.security`                              | Enable JWT validation to access the resource                                                   | `true`<br/><br/>`(default: false)`                                                                                      |
| `cdn`                                             | The CDN management, if you use any cdn to proxy your requests Souin will handle that           |                                                                                                                         |
| `cdn.provider`                                    | The provider placed before Souin                                                               | `akamai`<br/><br/>`fastly`<br/><br/>`souin`                                                                             |
| `cdn.api_key`                                     | The api key used to access to the provider                                                     | `XXXX`                                                                                                                  |
| `cdn.dynamic`                                     | Enable the dynamic keys returned by your backend application                                   | `true`<br/><br/>`(default: false)`                                                                                      |
| `cdn.email`                                       | The api key used to access to the provider if required, depending the provider                 | `XXXX`                                                                                                                  |
| `cdn.hostname`                                    | The hostname if required, depending the provider                                               | `domain.com`                                                                                                            |
| `cdn.network`                                     | The network if required, depending the provider                                                | `your_network`                                                                                                          |
| `cdn.strategy`                                    | The strategy to use to purge the cdn cache, soft will keep the content as a stale resource     | `hard`<br/><br/>`(default: soft)`                                                                                       |
| `cdn.service_id`                                  | The service id if required, depending the provider                                             | `123456_id`                                                                                                             |
| `cdn.zone_id`                                     | The zone id if required, depending the provider                                                | `anywhere_zone`                                                                                                         |
| `default_cache.badger`                            | Configure the Badger cache storage                                                             |                                                                                                                         |
| `default_cache.badger.path`                       | Configure Badger with a file                                                                   | `/anywhere/badger_configuration.json`                                                                                   |
| `default_cache.badger.configuration`              | Configure Badger directly in the Caddyfile or your JSON caddy configuration                    | [See the Badger configuration for the options](https://dgraph.io/docs/badger/get-started/)                              |
| `default_cache.headers`                           | List of headers to include to the cache                                                        | `- Authorization`<br/><br/>`- Content-Type`<br/><br/>`- X-Additional-Header`                                            |
| `default_cache.olric`                             | Configure the Olric cache storage                                                              |                                                                                                                         |
| `default_cache.olric.path`                        | Configure Olric with a file                                                                    | `/anywhere/badger_configuration.json`                                                                                   |
| `default_cache.olric.configuration`               | Configure Olric directly in the Caddyfile or your JSON caddy configuration                     | [See the Badger configuration for the options](https://github.com/buraksezer/olric/blob/master/cmd/olricd/olricd.yaml/) |
| `default_cache.port.{web,tls}`                    | The device's local HTTP/TLS port that Souin should be listening on                             | Respectively `80` and `443`                                                                                             |
| `default_cache.regex.exclude`                     | The regex used to prevent paths being cached                                                   | `^[A-z]+.*$`                                                                                                            |
| `default_cache.stale`                             | The stale duration                                                                             | `25m`                                                                                                                   |
| `default_cache.ttl`                               | The TTL duration                                                                               | `120s`                                                                                                                  |
| `log_level`                                       | The log level                                                                                  | `One of DEBUG, INFO, WARN, ERROR, DPANIC, PANIC, FATAL it's case insensitive`                                           |
| `ssl_providers`                                   | List of your providers handling certificates                                                   | `- traefik`<br/><br/>`- nginx`<br/><br/>`- apache`                                                                      |
| `urls.{your url or regex}`                        | List of your custom configuration depending each URL or regex                                  | 'https:\/\/yourdomain.com'                                                                                              |
| `urls.{your url or regex}.ttl`                    | Override the default TTL if defined                                                            | `90s`<br/><br/>`10m`                                                                                                    |
| `urls.{your url or regex}.headers`                | Override the default headers if defined                                                        | `- Authorization`<br/><br/>`- 'Content-Type'`                                                                           |
| `surrogate_keys.{key name}.headers`               | Headers that should match to be part of the surrogate key group                                | `Authorization: ey.+`<br/><br/>`Content-Type: json`                                                                     |
| `surrogate_keys.{key name}.headers.{header name}` | Header name that should be present a match the regex to be part of the surrogate key group     | `Content-Type: json`                                                                                                    |
| `surrogate_keys.{key name}.url`                   | Url that should match to be part of the surrogate key group                                    | `.+`                                                                                                                    |
| `ykeys.{key name}.headers`                        | (DEPRECATED) Headers that should match to be part of the ykey group                            | `Authorization: ey.+`<br/><br/>`Content-Type: json`                                                                     |
| `ykeys.{key name}.headers.{header name}`          | (DEPRECATED) Header name that should be present a match the regex to be part of the ykey group | `Content-Type: json`                                                                                                    |
| `ykeys.{key name}.url`                            | (DEPRECATED) Url that should match to be part of the ykey group                                | `.+`                                                                                                                    |
|  Key                                              |  Description                                                                                   |  Value example                                                                                                          |

## APIs
All endpoints are accessible through the `api.basepath` configuration line or by default through `/souin-api` to avoid named route conflicts. Be sure to define an unused route to not break your existing application.

### Souin API
Souin API allow users to manage the cache.  
The base path for the souin API is `/souin`.  
The Souin API supports the invalidation by surrogate keys such as Fastly which will replace the Varnish system. You can read the doc [about this system](https://github.com/darkweak/souin/blob/master/cache/surrogate/README.md).
This system is able to invalidate by tags your cloud provider cache. Actually it supports Akamai and Fastly but in a near future some other providers would be implemented like Cloudflare or Varnish.

| Method  | Endpoint          | Description                                                                              |
|:-------:|:-----------------:|:-----------------------------------------------------------------------------------------|
| `GET`   | `/`               | List stored keys cache                                                                   |
| `PURGE` | `/{id or regexp}` | Purge selected item(s) depending. The parameter can be either a specific key or a regexp |
| `PURGE` | `/?ykey={key}`    | Purge selected item(s) corresponding to the target ykey such as Varnish (deprecated)     |

### Security API
Security API allows users to protect other APIs with JWT authentication.  
The base path for the security API is `/authentication`.

| Method | Endpoint   | Body                                       | Headers                                                                         | Description                                                                                                            |
|:------:|:----------:|:------------------------------------------:|:-------------------------------------------------------------------------------:|:----------------------------------------------------------------------------------------------------------------------:|
| `POST` | `/login`   | `{"username":"admin", "password":"admin"}` | `['Content-Type' => 'json']`                                                    | Try to login, it returns a response which contains the cookie name `souin-authorization-token` with the JWT if succeed |
| `POST` | `/refresh` | `-`                                        | `['Content-Type' => 'json', 'Cookie' => 'souin-authorization-token=the-token']` | Refreshes the token, replaces the old with a new one                                                                   |

## Diagrams

### Sequence diagram
See the sequence diagram for the minimal version below
![Sequence diagram](https://github.com/darkweak/souin/blob/master/docs/plantUML/sequenceDiagram.svg?sanitize=true)

## Cache systems
Supported providers
 - [Badger](https://github.com/dgraph-io/badger)
 - [Olric](https://github.com/buraksezer/olric)

 The cache system sits on top of three providers at the moment. It provides an in-memory, redis and Olric cache systems because setting, getting, updating and deleting keys in these providers is as easy as it gets.  
 In order to do that, Redis and Olric providers need to be either on the same network as the Souin instance when using docker-compose or over the internet, then it will use by default in-memory to avoid network latency as much as possible. 
 Souin will return at first the in-memory response when it gives a non-empty response, then the olric one followed by the redis one with same condition, or fallback to the reverse proxy otherwise.
 Since v1.4.2, Souin supports [Olric](https://github.com/buraksezer/olric) to handle distributed cache.

### Cache invalidation
The cache invalidation is built for CRUD requests, if you're doing a GET HTTP request, it will serve the cached response when it exists, otherwise the reverse-proxy response will be served.  
If you're doing a POST, PUT, PATCH or DELETE HTTP request, the related cache GET request, and the list endpoint will be dropped.  
It also supports invalidation via [Souin API](#souin-api) to invalidate the cache programmatically.

## Examples

### Træfik container
[Træfik](https://traefik.io) is a modern reverse-proxy which helps you to manage full container architecture projects.

```yaml
# your-traefik-instance/docker-compose.yml
version: '3.7'

x-networks: &networks
  networks:
    - your_network

services:
  traefik:
    image: traefik:v2.0
    command: --providers.docker
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /anywhere/traefik.json:/acme.json
    <<: *networks

  # your other services here...

networks:
  your_network:
    external: true
```

```yaml
# your-souin-instance/docker-compose.yml
version: '3.7'

x-networks: &networks
  networks:
    - your_network

services:
  souin:
    image: darkweak/souin:latest
    ports:
      - 80:80
      - 443:443
    environment:
      GOPATH: /app
    volumes:
      - /anywhere/traefik.json:/ssl/traefik.json
      - /anywhere/configuration.yml:/configuration/configuration.yml
    <<: *networks

networks:
  your_network:
    external: true
```

## Plugins

### Caddy module
To use Souin as caddy module, you can refer to the [Caddy module integration folder](https://github.com/darkweak/souin/tree/master/plugins/caddy) to discover how to configure it.  
The related Caddyfile can be found [here](https://github.com/darkweak/souin/tree/master/plugins/caddy/Caddyfile).  
Then you just have to run the following command:
```bash
xcaddy build --with github.com/darkweak/souin/plugins/caddy
```

There is the fully configuration below
```caddy
{
    order cache before rewrite
    log {
        level debug
    }
    cache {
        api {
            basepath /some-basepath
            souin {
                enable true
                security true
            }
        }
        badger {
            path the_path_to_a_file.json
            configuration {
                # Your badger configuration here
            }
        }
        cdn {
            api_key XXXX
            dynamic true
            email darkweak@protonmail.com
            hostname domain.com
            network your_network
            provider fastly
            strategy soft
            service_id 123456_id
            zone_id anywhere_zone
        }
        headers Content-Type Authorization
        log_level debug
        olric {
            url url_to_your_cluster:3320
            path the_path_to_a_file.yaml
            configuration {
                # Your badger configuration here
            }
        }
        regex {
            exclude /test2.*
        }
        stale 200s
        ttl 1000s
    }
}

:4443
respond "Hello World!"

@match path /test1*
@match2 path /test2*
@matchdefault path /default
@souin-api path /souin-api*

cache @match {
    ttl 5s
}

cache @match2 {
    ttl 50s
    headers Authorization
}

cache @matchdefault {
    ttl 5s
}

cache @souin-api {}
```

### Echo middleware
To use Souin as echo middleware, you can refer to the [Echo plugin integration folder](https://github.com/darkweak/souin/tree/master/plugins/echo) to discover how to configure it.  
You just have to define a new echo router and tell to the instance to use the process method like below:
```go
import (
	"net/http"

	souin_echo "github.com/darkweak/souin/plugins/echo"
	"github.com/labstack/echo/v4"
)

func main(){

    // ...
	e := echo.New()
	s := souin_echo.New(souin_echo.DefaultConfiguration)
	e.Use(s.Process)
    // ...

}
```

### Gin middleware
To use Souin as gin middleware, you can refer to the [Gin plugin integration folder](https://github.com/darkweak/souin/tree/master/plugins/gin) to discover how to configure it.  
You just have to define a new gin router and tell to the instance to use the process method like below:
```go
import (
	"net/http"

	souin_gin "github.com/darkweak/souin/plugins/gin"
	"github.com/gin-gonic/gin"
)

func main(){

    // ...
	r := gin.New()
	s := souin_gin.New(souin_gin.DefaultConfiguration)
	r.Use(s.Process())
    // ...

}
```

### Træfik plugin
To use Souin as Træfik plugin, you can refer to the [pilot documentation](https://pilot.traefik.io/plugins/6123af58d00e6cd1260b290e/souin) and the [Træfik plugin integration folder](https://github.com/darkweak/souin/tree/master/plugins/traefik) to discover how to configure it.  
You have to declare the `experimental` block in your traefik static configuration file. Keep in mind Træfik run their own interpreter and they often break any dependances (such as the yaml.v3 support).
```yaml
# anywhere/traefik.yml
experimental:
  plugins:
    souin:
      moduleName: github.com/darkweak/souin
      version: v1.5.11-beta6
```
After that you can declare either the whole configuration at once in the middleware block or by service. See the examples below.
```yaml
# anywhere/dynamic-configuration
http:
  routers:
    whoami:
      middlewares:
        - http-cache
      service: whoami
      rule: Host(`domain.com`)
  middlewares:
    http-cache:
      plugin:
        souin-plugin:
          api:
            souin:
              enable: true
          default_cache:
            headers:
              - Authorization
              - Content-Type
            regex:
              exclude: '/test_exclude.*'
            ttl: 5s
          log_level: debug
          urls:
            'domain.com/testing':
              ttl: 5s
              headers:
                - Authorization
            'mysubdomain.domain.com':
              ttl: 50s
              headers:
                - Authorization
                - 'Content-Type'
          ykeys:
            The_First_Test:
              headers:
                Content-Type: '.+'
            The_Second_Test:
              url: 'the/second/.+'
            The_Third_Test:
            The_Fourth_Test:
          surrogate_keys:
            The_First_Test:
              headers:
                Content-Type: '.+'
            The_Second_Test:
              url: 'the/second/.+'
            The_Third_Test:
            The_Fourth_Test:
```

```yaml
# anywhere/docker-compose.yml
services:
#...
  whoami:
    image: traefik/whoami
    labels:
      # other labels...
      - traefik.http.routers.whoami.middlewares=http-cache
      - traefik.http.middlewares.http-cache.plugin.souin-plugin.api.souin.enable=true
      - traefik.http.middlewares.http-cache.plugin.souin-plugin.default_cache.headers=Authorization,Content-Type
      - traefik.http.middlewares.http-cache.plugin.souin-plugin.default_cache.ttl=10s
      - traefik.http.middlewares.http-cache.plugin.souin-plugin.log_level=debug
```

### Tyk plugin
To use Souin as a Tyk plugin, you can refer to the [Tyk plugin integration folder](https://github.com/darkweak/souin/tree/master/plugins/tyk) to discover how to configure it.  
You have to define the use of Souin as `post` and `response` custom middleware. You can compile your own Souin integration using the `Makefile` and the `docker-compose` inside the [tyk integration directory](https://github.com/darkweak/souin/tree/master/plugins/tyk) and place your generated `souin-plugin.so` file inside your `middleware` directory.
```json
{
  "name":"httpbin.org",
  "api_id":"3",
  "org_id":"3",
  "use_keyless": true,
  "version_data": {
    "not_versioned": true,
    "versions": {
      "Default": {
        "name": "Default",
        "use_extended_paths": true
      }
    }
  },
  "custom_middleware": {
    "pre": [],
    "post": [
      {
        "name": "SouinRequestHandler",
        "path": "/opt/tyk-gateway/middleware/souin-plugin.so"
      }
    ],
    "post_key_auth": [],
    "auth_check": {
      "name": "",
      "path": "",
      "require_session": false
    },
    "response": [
      {
        "name": "SouinResponseHandler",
        "path": "/opt/tyk-gateway/middleware/souin-plugin.so"
      }
    ],
    "driver": "goplugin",
    "id_extractor": {
      "extract_from": "",
      "extract_with": "",
      "extractor_config": {}
    }
  },
  "proxy":{
    "listen_path":"/httpbin/",
    "target_url":"http://httpbin.org/",
    "strip_listen_path":true
  },
  "active":true,
  "souin": {
    "api": {
      "souin": {
        "enable": true
      }
    },
    "default_cache": {
      "ttl": "5s"
    }
  }
}
```

### Prestashop plugin
A repository called [prestashop-souin](https://github.com/lucmichalski/prestashop-souin) has been started by [lucmichalski](https://github.com/lucmichalski). You can manage your Souin instance through the admin panel UI.

### Wordpress plugin
A repository called [wordpress-souin](https://github.com/Darkweak/wordpress-souin) to be able to manage your Souin instance through the admin panel UI.


## Credits

Thanks to these users for contributing or helping this project in any way  
* [Oxodao](https://github.com/oxodao)
* [Deuchnord](https://github.com/deuchnord)
* [Sata51](https://github.com/sata51)
* [Pierre Diancourt](https://github.com/pierrediancourt)
* [Burak Sezer](https://github.com/buraksezer)
* [Luc Michalski](https://github.com/lucmichalski)
* [Jenaye](https://github.com/jenaye)
* [Brennan Kinney](https://github.com/polarathene)
* [Agneev](https://github.com/agneevX)
* [Eidenschink](https://github.com/eidenschink/)
* [Massimiliano Cannarozzo](https://github.com/maxcanna/)
