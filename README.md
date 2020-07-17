# Docker private registry implementation

This little repo configures and runs local docker private registry with a simple UI on Synology server. 
It is built off two existing docker containers, [registry](https://hub.docker.com/_/registry)
and  [quiq/docker-registry-ui](https://github.com/Quiq/docker-registry-ui) and basically provides tested out
configuration.

These two services use the following directories:

| Directory | Function |
| --------- | -------- |
| `./auth` | Designated for authentication information |
| `./certs` | Should be used for storing TLS certificates when TLS is set up |
| `./data/repo` | The actual registry data / manifests. This should be regularly backed up |
| `./data/ui` | UI repo event database. Probably less important to back up |

The components of this package will use the following arbitrarily chosen ports on your Synology:
* 49999 - The port where the repo API is exposed
* 49998 - The port of the repo UI

If required, these ports can be arbitrarily altered by a simple search-replace in these instructions, 
`./docker-compose.yml` and `./config-ui.yml`.

## Installation

Installation steps below assumes the following URLs:
* `registry.mywebsite.com` - a public URL for registry repo (only if you want to expose the repo outside of your LAN)
* `registry.home` - a local (private) URL for registry repo
* `registry-ui.home` - a local (private) URL for registry UI

There is no public UI for registry, as [quiq/docker-registry-ui](https://github.com/Quiq/docker-registry-ui) does
not currently provide authentication. 

HTTPS-related steps below are only useful if you are opening your registry repo externally. 
These external URL steps can be skipped. Actually, if you do not know what you are doing, prefer skipping them. 
If you decide to open your repo externally, consider security risks and their mitigation, particularly implementing 
an authentication method other than basic authentication. Never open your repo without at least implementing HTTPS/TLS.

1. **Pre-installation:** You can use the registry directly on ports 49999 (registry repo) and 49998 (UI). 
   In that case you only need to open ports using the first step. The rest of steps is there to allow for more
   convenient usage without port numbers:
    1. Open port 49998, 49999 in the Synology server firewall. If you are not implementing TLS, make sure that
       ports are only open for your local IP subnet.
    1. If you only want to open up the registry repo and UI locally on your LAN, use the local DNS server and create 
       A-records for local URLs (UI and repo). 
       For example, if your home domain is called `.home` and your Synology has a DNS of `myds.home`, you can create 
       the following:
       * For registry repo:
           * Source:
               * Protocol: HTTP
               * Hostname: `registry.home`
               * Port: 80
           * Destination:
               * Protocol: HTTP
               * Hostname: 127.0.0.1 (Note: Using server IP address or DNS alias will probably not work)
               * Port: 49999
       * For registry UI:
           * Source:
               * Protocol: HTTP
               * Hostname: `registry-ui.home`
               * Port: 80
           * Destination:
               * Protocol: HTTP
               * Hostname: 127.0.0.1 (Note: Using server IP address or DNS alias will probably not work)
               * Port: 49998
    1. If you are opening repo externally, make sure you have an public DNS alias (an example of 
       `registry.mywebsite.com` is used here) and appropriate certificates. Then open the port 443 on your router 
       and set up the reverse proxy in the following way:
       * One for registry repo:
           * Source:
               * Protocol: HTTPS
               * Hostname: `registry.mywebsite.com`
               * Port: 443
           * Destination:
               * Protocol: HTTP
               * Hostname: 127.0.0.1 (Note: Using server IP address or DNS alias will probably not work)
               * Port: 49999
       * One for registry UI:
           * Source:
               * Protocol: HTTP
               * Hostname: `registry-ui.home`
               * Port: 80
           * Destination:
               * Protocol: HTTP
               * Hostname: 127.0.0.1 (Note: Using server IP address or DNS alias will probably not work)
               * Port: 49998
* SSH to the server
    * Gain root access using `sudo -i`
    * Go to docker directory in your system (e.g. `cd /volume1/docker`)
    * Clone this repo (into directory called `registry`)
    * Create necessary directories:
        ``` 
        cd registry
        mkdir repo/certs
        mkdir repo/auth
        ```
    * Insert your certificate and private key into `./repo/certs` directory (`cert.pem` and `privkey.pem`)
    * Create your `./repo/auth/htpasswd` file, for example by using 
      [this tool](https://hostingcanada.org/htpasswd-generator/). Add necessary users. User / password combinations
      in this file will be able to use the repo API when logged in.
    * Change the following configuration
        * `config-ui.yml`
            * `registry_url` - the full URL of your registry, e.g. `https://registry.mywebsite.com`
            * `registry_username`, `registry_username` - according to what you set up in `htpasswd`
            * `event_listener_token` - a random token matching the respective information in `config-repo.yml`
              (see below)
        * `config-repo.yml`
            * `http.host` - your host name, e.g. `registry.mywebsite.com`
            * `http.secret` - replace with a short random string, for production use strong and secure generator,
              like [random.org](https://www.random.org/strings/)
            * `notifications.endpoints[0].headers.Authorization` change the bearer token to match
              the `event_listener_token` from `config-ui.yml`

**Notes:**
1. All of the above configuration values are set to CHANGEME. Searching your repo using `git grep CHANGEME`
should reveal any forgotten values.
1. This configuration creates registry repo API on port 49999 and UI on port 49998.
It contains basic TLS setup for the repo. UI is currently set up as plain HTTP with no authentication, so it is not
suitable for public exposure.

## Starting, stopping and removing the UI

* Start the registry and the UI using `./start-registry`
* Stop the registry and remove all docker objects using `./stop-registry`

Stop script does not remove any data in `auth`, `certs` or `data` directory.
New version can be deployed, which will automatically use the pre-existing data.
