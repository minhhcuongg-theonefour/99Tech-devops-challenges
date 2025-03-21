# Problem 3: Diagnose Me Doctor

## Troubleshoot storage issue for NGINX Load Balancer on VM

### First let's check the disk usage of the VM

```bash
# Check disk usage
df -h

# List the largest directories/files starting from the root (/) to see what’s consuming space.
du -h / | sort -rh | head -n 10

# Check for held deleted files
lsof | grep deleted

# Brief check NGINX cache and the size of if
ls -lh /var/log/cache/
du -sh /var/nginx/cache

# Brief check NGINX log the size of it
ls -lh /var/log/nginx/
du -sh /var/log/nginx/
```

### Common causes/scenario and solutions

**1. Caching Zones Issue**

- **Big cache size**: Setting a large `max_size` in `proxy_cache_path` lets NGINX save lots of data on disk.
- **Long cache time**: Keeping cached items too long (high `inactive` in `proxy_cache_path`) makes the cache grow bigger.

**Investigate steps:**
```bash
# Confirm if the cache file exists and check the size
du -sh /var/nginx/cache

# Check config file at /etc/nginx/nginx.conf, review http{} block
cat /etc/nginx/nginx.conf
# or
awk '/http *{/{flag=1; print; next} flag{print} /}/{flag=0}' /etc/nginx/nginx.conf
# to print out the http{} block for a quick check

```

**Impact:**
- Disk Space Exhaustion
- NGINX Service Disruption
- Performance Degradation
- Upstream Service Strain

**Recovery Steps:**

```bash
#  Clear cache
rm -rf /var/nginx/cache/*
```


- **Limit cache size**: Use the ``max_size`` parameter in the ``proxy_cache_path`` directive to limit the maximum capacity that the cache can use.
- **Adjust inactive time**: Use the ``inactive`` parameter in ``proxy_cache_path`` to set a timeout before infrequently accessed items are removed from the cache.

```bash
#  Access to config file, normally the proxy_cache_path lied inside the http{} block
nano /etc/nginx/nginx.conf
```

```bash
proxy_cache_path /var/nginx/cache     # storage location for cached files from proxied requests
            	 keys_zone=CACHE:60m  # shared memory zone named CACHE with a size of 60MB to store cache keys
				 levels=1:2           # organizes the cache directory into a two-level structure
				 inactive=3h          # cached items are removed if not accessed within 3 hours
				 max_size=20g;        # limits the cache size on disk to 20GB
proxy_cache CACHE;                    # enables caching, using the CACHE zone defined above.
```


```bash
# After saving config, check syntax and reload NGINX
nginx -t && systemctl reload nginx
```

```bash
# Runs du -sh every 60 seconds to show the cache size in real-time
watch -n 60 "du -sh /var/nginx/cache"
   ```

**2. Log Accumulation Issue**

- **Detailed logging**: Setting a high-detail log level (like `debug` for `error_log`) makes log files grow fast.
- **Long log storage**: Without good log rotation or cleanup, old logs pile up and use lots of space.
- **Logging every request**: Recording all requests in the access log, especially on busy sites, creates big log files quickly.

**Investigate steps:**
```bash
# Check NGINX log directory size
du -sh /var/log/nginx

# Identify specific log file sizes
ls -lh /var/log/nginx/

# Check log format at /etc/nginx/nginx.conf, review server{} block if there's any custom log declared
cat /etc/nginx/nginx.conf
# or
awk '/server *{/{flag=1; print; next} flag{print} /}/{flag=0}' /etc/nginx/nginx.conf
# to print out the server{} block for a quick check

```

Check if there's a log declaration:

```bash
server {
	   	access_log /var/log/nginx/access.log geoproxy;
        error_log /var/log/nginx/error.log warn;
		# ...
}
```

**Impact:**
- **Out of space**: Large logs fill the disk (upto 99%).
- **Log loss**: New logs can’t be written when disk is full.
- **System slowdown**: Higher I/O, lower performance.
- **System errors**: Affects filesystem and other processes.

**Recovery Steps:**

   ```bash
   #  Truncate large log files, faster and safer than using rm -rf
   > /var/log/nginx/access.log
   > /var/log/nginx/error.log

```
```bash
# Config log rotation to rotate daily, keep n days (depends on business cases), and compress old files
nano /etc/logrotate.d/nginx
```

```bash
/var/log/nginx/*.log {
        daily
        missingok
        rotate n
        compress
        delaycompress
        notifempty
        create 0640 www-data adm
        sharedscripts
        prerotate
                if [ -d /etc/logrotate.d/httpd-prerotate ]; then \
                        run-parts /etc/logrotate.d/httpd-prerotate; \
                fi \
        endscript
        postrotate
                invoke-rc.d nginx rotate >/dev/null 2>&1
        endscript
}

```
```bash
# After saving config, check syntax and reload NGINX
nginx -t && systemctl reload nginx

# Test the logrotate
logrotate -f /etc/logrotate.d/nginx
```


**Long-term Solutions:**
- **Adjust logging level**: Set the logging level appropriate for your debugging and monitoring needs. Typically, info, warn, or error are sufficient for production environments.
- **Set up log rotation**: Use system tools like ``logrotate`` to automatically rotate, compress, and delete old log files on a schedule.
- **Only log important requests**: Use the if parameter in the ``access_log`` directive to only log requests that meet certain conditions.
- **Send logs to a centralized system (Syslog)**: Configure Nginx to send logs to a remote Syslog server for centralized storage and management - allows to view logs together in one place without having to jump from server to server, freeing up space on the Nginx server

Consider these steps below to send logs to a remote Syslog server
```bash
# Open NGINX config file
nano /etc/nginx/nginx.conf
```

```bash
# Edit in http{} block
http {
    # Send access logs to remote Syslog server
    access_log syslog:server=10.0.1.42,tag=nginx,severity=info geoproxy;

    # Send error logs to remote Syslog server
    error_log syslog:server=10.0.1.42 debug;

    # Existing proxy_cache_path or server blocks remain unchanged
    proxy_cache_path /var/nginx/cache keys_zone=CACHE:60m levels=1:2 inactive=1h max_size=10g;
    server {
        listen 80;
        ...
    }
}
```

```bash
# After saving config, check syntax and reload NGINX
nginx -t && systemctl reload nginx

```
