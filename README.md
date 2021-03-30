## Nginx PrivateTmp
By default for security reasons `PrivateTmp` in `nginx.service` config file is set to `True`. So you can't define a cache path in another directory. You should set `PrivateTmp=false` and reload systemctl deamon to config desire path for cache storage.

***
  

## Hot Cache in Memory
There are two main types of RAM disk which can be used in Linux and each have their own benefits and weaknesses:
* ramfs
* tmpfs

You can read the the details with this [link](https://www.jamescoyle.net/knowledge/951-the-difference-between-a-tmpfs-and-ramfs-ram-disk). In this scenario we use `tmpfs` which is a newer technology.

Run `free -hm` command to see how much memory you have already.

Create a folder to use as a mount point for your RAM disk.

`mkdir /tmp/nginx-ram-cache`
 
Open `/etc/fstab` file with your favorite editor and add this line:

`tmpfs      /tmp/nginx-ram-cache      tmpfs defaults,noatime,nosuid,nodev,noexec,mode=1777,size=200m 0 0`

Save and exit the `/etc/fstab` file and run `mount -a` command to reload the config file and mount the RAM disk permanently. use `df -h` to check if it's ok.

Now let's go to the nginx part. Open `/etc/nginx/nginx.conf` file with your favorite editor and in the `http` section put this line to declare a cache path for nginx:

`proxy_cache_path /tmp/nginx-ram-cache use_temp_path=off keys_zone=ram_cache:10m max_size=190m inactive=5m ;`

* `use_temp_path=off` Directive instructs NGINX to write them to the same directories where they will be cached.

* `keys_zone=ram_cache:10m` Defines the name and size of the shared memory zone that is used to store metadata about cached items.

* `max_size=190m` If the cache size exceeds the limit set by the max_size parameter to the proxy_cache_path directive, the cache manager removes the data that was accessed least recently.

* `inactive=5m` Specifies how long an item can remain in the cache without being accessed.

Now open desired config file, it's usually in `/etc/nginx/sites-enabled` directory and put this sample config. here we cache `.ts` video files for a little time in cache:

```
server {
        server_name mydomian.com;
        listen 80;

        add_header X-Cache-Status $upstream_cache_status;

        location / {
           proxy_pass http://backend:8000;
           access_log   /var/log/nginx/mydomian.com-access.log main ;
           error_log    /var/log/nginx/mydomian.com-error.log info;
                }
        location ~* (.ts)$ {
           access_log   /var/log/nginx/mydomian.com-access.log  main;
           error_log    /var/log/nginx/mydomian.com-error.log info ;
           proxy_pass http://backend:8000;
           proxy_cache_key $uri;
           proxy_cache_methods GET;
           proxy_cache ram_cache ;
           proxy_cache_valid 200 302 206 4m;
                           }
}
```

Now for check our configs run `nginx -t` to check it. If you had a typo or miss configurations, It will return error. If you see **test is ok** in the return, you can reload or restart nginx service.

***
## Cold Cache in Disk

In nginx we can gave multiple cache storage for multiple purposes. For example we can use RAM memory, HDD or SSD or all of them. In the second part of scenario we use hard disk (SSD or HDD) to configure second cache storage for nginx.

Make sure you configured and partitioned the new disk that you about to use for cache storage.

Now we define second cache path in `nginx.conf` file inside `http` section:

`proxy_cache_path /tmp/nginx-ssd-cache use_temp_path=off keys_zone=ssd_cache:100m max_size=500G inactive=168h loader_threshold=300 loader_files=200;`

* `use_temp_path=off` Directive instructs NGINX to write them to the same directories where they will be cached.

* `keys_zone=ssd_cache_temp:100m` Defines the name and size of the shared memory zone that is used to store metadata about cached items.

* `max_size=500G` If the cache size exceeds the limit set by the max_size parameter to the proxy_cache_path directive, the cache manager removes the data that was accessed least recently.

* `inactive=168h` Specifies how long an item can remain in the cache without being accessed.

* `loader_threshold=300` Duration of an iteration, in milliseconds (by default, 200)

* `loader_files=200` Maximum number of items loaded during one iteration (by default, 100)

```
server {
    listen 80;
    server_name mydomain.com ;

    add_header X-Cache-Status $upstream_cache_status;
    
    location / {
            proxy_pass http://backend:8000;
            access_log   /var/log/nginx/mydomain.com-access.log main ;
            error_log    /var/log/nginx/mydomain.com-error.log info;
                }
    location ~* (.ts)$ {
            access_log   /var/log/nginx/mydomain.com-access.log main;
            error_log    /var/log/nginx/mydomain.com-error.log info ;
            proxy_pass http://backend:8000;
            proxy_cache_key $uri;
            proxy_cache_methods GET;
            proxy_cache ssd_cache ;
            proxy_cache_valid 200 302 206 240h;
	    proxy_cache_min_uses 3;
                           }
}
```

Now for check our configs run `nginx -t` to check it. If you had a typo or miss configurations, It will return error. If you see **test is ok** in the return, you can reload or restart nginx service.

* `proxy_cache_key` Defines a key for caching.

* `proxy_cache_methods` Allowed client request methods.

* `proxy_cache_valid` Sets caching time for different response codes.

* `proxy_cache_min_uses` Sets the number of requests after which the response will be cached.


***

## Custom log print
The `add_header X-Cache-Status $upstream_cache_status;` will show if the served content is from cache or not. It will be `HIT` or `MISS`. You can see this variable in console of web browser or in the http header of response.

Another way is to config nginx access log to print if cache is miss or hit.

add this line in `nginx.conf` inside `http` section:

`log_format main '$remote_addr - $remote_user [$time_local]' '"$request" $status $upstream_cache_status $body_bytes_sent' '"$http_referer" "$http_user_agent"';`

Now when you add `main` in front of `access_log` it will print based on that format:

`access_log   /var/log/nginx/mydomain.com-access.log main ;`
