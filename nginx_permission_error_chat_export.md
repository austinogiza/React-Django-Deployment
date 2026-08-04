# Chat Export: Nginx Permission Denied Error Troubleshooting

## User Prompt

```text
2026/08/04 12:20:54 [crit] 21083#21083: *32 stat() "/home/sanntomafashion/sanntoma-fe/dist/index.html" failed (13: Permission denied), client: 34.107.93.3, server: 72.61.147.207, request: "GET / HTTP/1.1", host: "www.sanntomafashion.com"
2026/08/04 12:20:54 [crit] 21083#21083: *32 stat() "/home/sanntomafashion/sanntoma-fe/dist/index.html" failed (13: Permission denied), client: 34.107.93.3, server: 72.61.147.207, request: "GET / HTTP/1.1", host: "www.sanntomafashion.com"
2026/08/04 12:20:54 [crit] 21083#21083: *32 stat() "/home/sanntomafashion/sanntoma-fe/dist/index.html" failed (13: Permission denied), client: 34.107.93.3, server: 72.61.147.207, request: "GET / HTTP/1.1", host: "www.sanntomafashion.com"
2026/08/04 12:20:54 [crit] 21083#21083: *32 stat() "/home/sanntomafashion/sanntoma-fe/dist/index.html" failed (13: Permission denied), client: 34.107.93.3, server: 72.61.147.207, request: "GET / HTTP/1.1", host: "www.sanntomafashion.com"
2026/08/04 12:20:54 [crit] 21083#21083: *32 stat() "/home/sanntomafashion/sanntoma-fe/dist/index.html" failed (13: Permission denied), client: 34.107.93.3, server: 72.61.147.207, request: "GET / HTTP/1.1", host: "www.sanntomafashion.com"
2026/08/04 12:20:54 [crit] 21083#21083: *32 stat() "/home/sanntomafashion/sanntoma-fe/dist/index.html" failed (13: Permission denied), client: 34.107.93.3, server: 72.61.147.207, request: "GET / HTTP/1.1", host: "www.sanntomafashion.com"
2026/08/04 12:20:54 [crit] 21083#21083: *32 stat() "/home/sanntomafashion/sanntoma-fe/dist/index.html" failed (13: Permission denied), client: 34.107.93.3, server: 72.61.147.207, request: "GET / HTTP/1.1", host: "www.sanntomafashion.com"
2026/08/04 12:20:54 [crit] 21083#21083: *32 stat() "/home/sanntomafashion/sanntoma-fe/dist/index.html" failed (13: Permission denied), client: 34.107.93.3, server: 72.61.147.207, request: "GET / HTTP/1.1", host: "www.sanntomafashion.com"
2026/08/04 12:20:54 [crit] 21083#21083: *32 stat() "/home/sanntomafashion/sanntoma-fe/dist/index.html" failed (13: Permission denied), client: 34.107.93.3, server: 72.61.147.207, request: "GET / HTTP/1.1", host: "www.sanntomafashion.com"
2026/08/04 12:20:54 [error] 21083#21083: *32 rewrite or internal redirection cycle while internally redirecting to "/index.html", client: 34.107.93.3, server: 72.61.147.207, request: "GET / HTTP/1.1", host: "www.sanntomafashion.com"
2026/08/04 12:20:56 [crit] 21083#21083: *14 connect() to unix:/home/sanntomafashion/sanntoma-backend/sanntoma.sock failed (13: Permission denied) while connecting to upstream, client: 154.160.0.104, server: 72.61.147.207, request: "GET /api/categories/ HTTP/1.1", upstream: "http://unix:/home/sanntomafashion/sanntoma-backend/sanntoma.sock:/api/categories/", host: "api.sanntomafashion.com", referrer: "https://sanntoma-fe.vercel.app/"
```

---

## Assistant Response

This error indicates that the Nginx worker process (usually running as `www-data` or `nginx`) does not have execute (`+x`) permissions on the parent directory path `/home/sanntomafashion/` or read permissions on the web root and Gunicorn socket files.

By default, `/home/username` directories are restricted to `700` (`rwx------`), blocking outside services like Nginx from traversing into the folder.

### Step-by-Step Resolution

#### Step 1: Grant execution permissions on the parent home directory
*Allows Nginx to traverse through `/home/sanntomafashion`*

Run the following to give external users execution (traversal) access to the home directory:

```bash
sudo chmod 755 /home/sanntomafashion
```

---

#### Step 2: Fix permissions for the Frontend static files
*Resolves the `13: Permission denied` on `index.html`*

Ensure the frontend build directory has proper read and execution permissions:

```bash
sudo chmod -R 755 /home/sanntomafashion/sanntoma-fe/dist
```

*(Note: Granting `+x` on parent subdirectories may also be needed if they were restricted: `chmod +x /home/sanntomafashion/sanntoma-fe`)*

---

#### Step 3: Fix permissions for the Unix Socket
*Resolves `13: Permission denied` on `sanntoma.sock`*

Give Nginx permissions to read and write to the Gunicorn socket file and its parent folder:

```bash
sudo chmod 755 /home/sanntomafashion/sanntoma-backend
sudo chmod 666 /home/sanntomafashion/sanntoma-backend/sanntoma.sock
```

Alternatively, assign the socket's group ownership to `www-data` inside your Gunicorn/systemd config (`umask = 007`, `group = www-data`).

---

#### Step 4: Verify Nginx worker user ownership
*Confirm which user runs Nginx*

Check which user runs the Nginx worker:

```bash
ps aux | grep nginx
```

If the worker runs as `www-data`, add `www-data` to the `sanntomafashion` user group as a cleaner security practice:

```bash
sudo usermod -aG sanntomafashion www-data
sudo chmod 750 /home/sanntomafashion
```

---

#### Step 5: Restart Nginx and Gunicorn
*Apply permission updates*

Restart both services to re-establish the connection:

```bash
sudo systemctl restart gunicorn
sudo systemctl restart nginx
```

---

*Once permissions are updated, the internal rewrite loop on `/index.html` will automatically resolve, as Nginx will successfully serve `index.html` instead of failing the file check and cycling back through `try_files`.*