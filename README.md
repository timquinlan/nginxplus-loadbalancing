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

This is a demonstration of NGINX+ least_time load balancing algorithm, active heatlh check, dynamic upstream management and plus API.  It creates four containers on your host, one reverse proxy listening on port 8080 and three upstreams to proxy too.  You will only need to interact with the reverse proxy server.  To get the Container ID of the reverse proxy

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


Next us ab to send 200 requests all at once to the proxy:


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



