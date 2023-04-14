# nginxplus-loadbalancing

This demo requires an NGINX+ repository key and cert (the build will fail if the files are not in place). Place the .key and .crt files in ./plus-build of this repo before running docker-compose. If you are not a current NGINX+ customer, you can request a free 30-day trial at https://www.nginx.com/free-trial-request/

In addition to the key/cert you will need:

* docker, docker-compose
* authorization to build containers
* authorization to forward host ports
* curl and ab installed on your host

Clone this repo and use docker-compose to bring up the environment:


    git clone https://github.com/timquinlan/nginxplus-loadbalancing
    cp nginx-repo.crt nginx-repo.key nginxplus-loadbalancing/plus-build
    cd nginxplus-loadbalancing
    docker-compose up -dA

This is a demonstration of NGINX+ least_time load balancing algorithm, active heatlh check, dynamic upstream management and plus API.  All of the configuration for this demo takes place in the **upstream** block.


    upstream oss_upstreams {
        zone oss_upstreams 64k;
        least_time header; 
        server labapp1:80 max_fails=1 fail_timeout=1s slow_start=30s weight=5;
        server labapp2:80 max_fails=1 fail_timeout=1s slow_start=30s weight=5;
        server labapp3:80 max_fails=1 fail_timeout=1s slow_start=30s weight=5;
        keepalive 32;
    }


The demo creates four containers on your host, one reverse proxy listening on port 8080 and three upstreams to proxy too.  You will only need to interact with the reverse proxy server.  To get the Container ID of the reverse proxy

    $ docker ps | grep nginxplus-loadbalancing-revproxy
    eb1002e2defa   nginxplus-loadbalancing-revproxy   "nginx -g 'daemon of…"   9 minutes ago   Up 9 minutes   0.0.0.0:8080->80/tcp   nginxplus-loadbalancing-revproxy-1

Open two additional terminals and connect to the reverse proxy container. Connect the first terminal and tail the error.log:


    $ docker exec -it eb1002e2defa bash
    # tail -f /var/log/nginx/error.log

Connect the second terminal and tail the access log:


    $ docker exec -it eb1002e2defa bash
    # tail -f /var/log/nginx/access.log

In another terminal, make a few http requests with curl and watch the access log:


    $ curl http://localhost:8080/

On the access.log terminal notice that the traffic is proxied to the upstream servers ("ua=" in this example) in a more-or-less round robin pattern.  This is because the [least_time](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#least_time) load balancing  algorithm specifies: 
>A group should use a load balancing method where a request is passed to the server with the least average response time and least number of active connections, taking into account weights of servers. If there are several such servers, they are tried in turn using a weighted round-robin balancing method.
In this example, since there is no additional load on the upstream servers they are all weighted the same:


    192.168.32.1 - - [14/Apr/2023:15:36:46 +0000] "GET / HTTP/1.1" 200 7234 "-" "" "curl/7.86.0" "-" "localhost" sn="_" rt=0.003 ua="192.168.32.3:80" us="200" uct="0.001" urt="0.003" uht="0.002"  uln="7215" cs=- 9f409cda740d258be7434dc3c51e27ea
    192.168.32.1 - - [14/Apr/2023:15:36:53 +0000] "GET / HTTP/1.1" 200 7228 "-" "" "curl/7.86.0" "-" "localhost" sn="_" rt=0.002 ua="192.168.32.2:80" us="200" uct="0.000" urt="0.001" uht="0.001"  uln="7215" cs=- aeac1ba6e3d9cdc5f568411df5a4e7fa
    192.168.32.1 - - [14/Apr/2023:15:36:55 +0000] "GET / HTTP/1.1" 200 7234 "-" "" "curl/7.86.0" "-" "localhost" sn="_" rt=0.001 ua="192.168.32.4:80" us="200" uct="0.000" urt="0.001" uht="0.001"  uln="7215" cs=- 29edf35240d7c9a72a20af19f863bd7c
    192.168.32.1 - - [14/Apr/2023:15:36:57 +0000] "GET / HTTP/1.1" 200 7234 "-" "" "curl/7.86.0" "-" "localhost" sn="_" rt=0.001 ua="192.168.32.3:80" us="200" uct="0.000" urt="0.001" uht="0.001"  uln="7215" cs=- 466fa5d30ff8d0c29be6a0a54b39ac11
    192.168.32.1 - - [14/Apr/2023:15:36:59 +0000] "GET / HTTP/1.1" 200 7234 "-" "" "curl/7.86.0" "-" "localhost" sn="_" rt=0.000 ua="192.168.32.2:80" us="200" uct="0.000" urt="0.001" uht="0.000"  uln="7215" cs=- 20a6bb1fd9fc24747aa6d6114e17544b
    192.168.32.1 - - [14/Apr/2023:15:37:00 +0000] "GET / HTTP/1.1" 200 7228 "-" "" "curl/7.86.0" "-" "localhost" sn="_" rt=0.004 ua="192.168.32.4:80" us="200" uct="0.000" urt="0.004" uht="0.004"  uln="7215" cs=- de4bbb1dfd68a293d6550945e717a871


Next use ab to send 200 requests all at once to the proxy:


    $ ab -c200 -n200 http://localhost:8080/ 

Notice now the traffic is proxied in a much less orderly pattern. Since we are sending so much more traffic, each upstream may have a different response time.  The logging is configured with some of NGINX+'s additional metrics as such:
    'uct="$upstream_connect_time" '
    'urt="$upstream_response_time" '
    'uht="$upstream_header_time"  '
    'uln="$upstream_response_length" '

When we look at the access log, notice the fields for uct, urt, uht and uln.  Least_time can act on either the time to get the first header OR the time to the last byte of the proxy_pass.  The lower the Upstream Response Time OR Upstream Header Time are, the more likely that server is to get traffic proxied to it.  


    192.168.32.1 - - [14/Apr/2023:15:41:42 +0000] "GET / HTTP/1.0" 200 7215 "-" "" "ApacheBench/2.3" "-" "localhost" sn="_" rt=0.019 ua="192.168.32.3:80" us="200" uct="0.014" urt="0.019" uht="0.019"  uln="7215" cs=- 383801db607a6cc95e99fb7c28129f4a
    192.168.32.1 - - [14/Apr/2023:15:41:42 +0000] "GET / HTTP/1.0" 200 7215 "-" "" "ApacheBench/2.3" "-" "localhost" sn="_" rt=0.022 ua="192.168.32.2:80" us="200" uct="0.002" urt="0.022" uht="0.016"  uln="7215" cs=- 6f04939523eda8b3ddccabdd08b393ea
    192.168.32.1 - - [14/Apr/2023:15:41:42 +0000] "GET / HTTP/1.0" 200 7215 "-" "" "ApacheBench/2.3" "-" "localhost" sn="_" rt=0.022 ua="192.168.32.2:80" us="200" uct="0.002" urt="0.022" uht="0.022"  uln="7215" cs=- 65412d85bccdf3067e6bc5073cecc74b
    192.168.32.1 - - [14/Apr/2023:15:41:42 +0000] "GET / HTTP/1.0" 200 7215 "-" "" "ApacheBench/2.3" "-" "localhost" sn="_" rt=0.022 ua="192.168.32.2:80" us="200" uct="0.002" urt="0.022" uht="0.022"  uln="7215" cs=- 3a298403ea069eacb27a7adaee2e3d1c
    192.168.32.1 - - [14/Apr/2023:15:41:42 +0000] "GET / HTTP/1.0" 200 7215 "-" "" "ApacheBench/2.3" "-" "localhost" sn="_" rt=0.003 ua="192.168.32.4:80" us="200" uct="0.001" urt="0.003" uht="0.003"  uln="7215" cs=- cbe1456a34cae509acf28141d7617ec5
    192.168.32.1 - - [14/Apr/2023:15:41:42 +0000] "GET / HTTP/1.0" 200 7215 "-" "" "ApacheBench/2.3" "-" "localhost" sn="_" rt=0.022 ua="192.168.32.2:80" us="200" uct="0.002" urt="0.022" uht="0.022"  uln="7215" cs=- 6caa7a634691d2120cadbfeecd88eb5a
    192.168.32.1 - - [14/Apr/2023:15:41:42 +0000] "GET / HTTP/1.0" 200 7215 "-" "" "ApacheBench/2.3" "-" "localhost" sn="_" rt=0.003 ua="192.168.32.2:80" us="200" uct="0.001" urt="0.003" uht="0.003"  uln="7215" cs=- 682eb186bd6f9ac9ebaea9b07d83ba89
    192.168.32.1 - - [14/Apr/2023:15:41:42 +0000] "GET / HTTP/1.0" 200 7215 "-" "" "ApacheBench/2.3" "-" "localhost" sn="_" rt=0.003 ua="192.168.32.4:80" us="200" uct="0.001" urt="0.003" uht="0.003"  uln="7215" cs=- 331a12a083d48768e10f19a7fa320432
    192.168.32.1 - - [14/Apr/2023:15:41:42 +0000] "GET / HTTP/1.0" 200 7215 "-" "" "ApacheBench/2.3" "-" "localhost" sn="_" rt=0.003 ua="192.168.32.4:80" us="200" uct="0.001" urt="0.003" uht="0.003"  uln="7215" cs=- 913c2deb25d45435457a6df94de7ac57
    192.168.32.1 - - [14/Apr/2023:15:41:42 +0000] "GET / HTTP/1.0" 200 7215 "-" "" "ApacheBench/2.3" "-" "localhost" sn="_" rt=0.003 ua="192.168.32.4:80" us="200" uct="0.001" urt="0.003" uht="0.003"  uln="7215" cs=- fda9a716331b8d5bd4da85bd23da2c12
    192.168.32.1 - - [14/Apr/2023:15:41:42 +0000] "GET / HTTP/1.0" 200 7215 "-" "" "ApacheBench/2.3" "-" "localhost" sn="_" rt=0.013 ua="192.168.32.3:80" us="200" uct="0.001" urt="0.013" uht="0.002"  uln="7215" cs=- 52697239766324fb55987fdfe5b18486
    192.168.32.1 - - [14/Apr/2023:15:41:42 +0000] "GET / HTTP/1.0" 200 7215 "-" "" "ApacheBench/2.3" "-" "localhost" sn="_" rt=0.019 ua="192.168.32.4:80" us="200" uct="0.001" urt="0.019" uht="0.008"  uln="7215" cs=- 53764a38dacdf6abc8465897d595e171


To get a more indepth look at each upstream server's stats, use the NGINX+ API: 


    $ curl http://localhost:8080/api/8/http/upstreams | jq

    {
      "oss_upstreams": {
        "peers": [
          {
            "id": 0,
            "server": "192.168.32.4:80",
            "name": "labapp1:80",
            "backup": false,
            "weight": 5,
            "state": "up",
            "active": 0,
            "requests": 139,
            "header_time": 9,
            "response_time": 9,
            "responses": {
              "1xx": 0,
              "2xx": 139,
              "3xx": 0,
              "4xx": 0,
              "5xx": 0,
              "codes": {
                "200": 139
              },
              "total": 139
            },
            "sent": 13892,
            "received": 1028878,
            "fails": 0,
            "unavail": 0,
            "health_checks": {
              "checks": 546,
              "fails": 1,
              "unhealthy": 1,
              "last_passed": true
            },
            "downtime": 5045,
            "selected": "2023-04-14T15:41:42Z"
          },
          {
            "id": 1,
            "server": "192.168.32.3:80",
            "name": "labapp2:80",
            "backup": false,
            "weight": 5,
            "state": "up",
            "active": 0,
            "requests": 135,
            "header_time": 9,
            "response_time": 9,
            "responses": {
              "1xx": 0,
              "2xx": 135,
              "3xx": 0,
              "4xx": 0,
              "5xx": 0,
              "codes": {
                "200": 135
              },
              "total": 135
            },
            "sent": 13492,
            "received": 999270,
            "fails": 0,
            "unavail": 0,
            "health_checks": {
              "checks": 546,
              "fails": 0,
              "unhealthy": 0,
              "last_passed": true
            },
            "downtime": 0,
            "selected": "2023-04-14T15:41:42Z"
          },
          {
            "id": 2,
            "server": "192.168.32.2:80",
            "name": "labapp3:80",
            "backup": false,
            "weight": 5,
            "state": "up",
            "active": 0,
            "requests": 132,
            "header_time": 10,
            "response_time": 11,
            "responses": {
              "1xx": 0,
              "2xx": 132,
              "3xx": 0,
              "4xx": 0,
              "5xx": 0,
              "codes": {
                "200": 132
              },
              "total": 132
            },
            "sent": 13192,
            "received": 977064,
            "fails": 0,
            "unavail": 0,
            "health_checks": {
              "checks": 546,
              "fails": 0,
              "unhealthy": 0,
              "last_passed": true
            },
            "downtime": 0,
            "selected": "2023-04-14T15:41:42Z"
          }
        ],
        "keepalive": 0,
        "zombies": 0,
        "zone": "oss_upstreams"
      }
    }


NGINX+ is configured to actively check the health of each upstream server by constantly making http calls to each.  If one fails it is marked unhealthy and proxies are suspended to that upstream.  Force a failure of the first upstream server by pausing it's container:


    $ docker ps | grep nginxplus-loadbalancing-labapp1
    d528a775f55f   nginxplus-loadbalancing-labapp1    "/docker-entrypoint.…"   50 minutes ago   Up 50 minutes   80/tcp                 nginxplus-loadbalancing-labapp1-1
    $ docker pause d528a775f55f

Notice the terminal were you are tailing the error.log, once it's last health check request fails you will see: 
   

    4/14 16:10:17 [error] 7#7: upstream timed out (110: Connection timed out) while reading response header from upstream, health check "" of peer 192.168.32.4:80 in upstream "oss_upstreams"
2023/04/14 16:10:17 [warn] 7#7: peer is unhealthy while reading response header from upstream, health check "" of peer 192.168.32.4:80 in upstream "oss_upstreams"

We can use the API to verify that labapp1 is marked unhealthy:
    $ curl http://localhost:8080/api/8/http/upstreams | jq | egrep "name|state"
        "name": "labapp1:80",
        "state": "unhealthy",
        "name": "labapp2:80",
        "state": "up",
        "name": "labapp3:80",
        "state": "up",


If you now send traffic to the reverse proxy, you will notice that traffic is only proxied to labapp2 and labapp3.  


If you unpause labapp1's container you will see a "peer is health" message in the error log almost immediately:


    2023/04/14 16:15:24 [notice] 7#7: peer is healthy while checking status code, health check "" of peer 192.168.32.4:80 in upstream "oss_upstreams"


You can check the API to verify that labapp1 is marked healthy:


    $ curl http://localhost:8080/api/8/http/upstreams | jq | egrep "name|state"
        "name": "labapp1:80",
        "state": "up",
        "name": "labapp2:80",
        "state": "up",
        "name": "labapp3:80",
        "state": "up",


If you now send traffic to the reverse proxy, you will notice that traffic is only proxied to all three labapp servers.




