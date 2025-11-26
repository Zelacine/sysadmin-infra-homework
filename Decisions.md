Briefly describe key decisions (3-5 points):
- Why you chose these variables and default values.
- How you connected Nginx and PHP-FPM (socket vs tcp), why.
- How you ensured idempotency in Ansible.
- What exactly /healthz checks and how.
- What you would improve given more time.


## 1. Variable choices and default values

**Design rationale:**
- **`web_app_env: dev`** — Defaults to development; can be overridden for staging/production.
- **`web_project_name: php-app`** — Used for consistent resource naming (containers, networks, volumes).
- **`web_project_root: "{{ playbook_dir | dirname }}"** — Dynamically resolves to the project root, making the playbook portable.
- **`web_nginx_conf_host_path`** — Points to the Terraform-managed config file that’s bind-mounted read-only into the container.
- **`web_php_fpm_upstream: "{{ web_php_fpm_container_name }}:9000"`** — Uses TCP (container name:port) because containers are on a shared Docker network.
- **`web_container_user: www-data`** — Standard web server user in Debian/Ubuntu-based images.
- **`web_temp_render_dir`** — Isolated per-project temp directory to avoid conflicts.

**Why `web_*` prefix:** Ansible-lint requires role-scoped variables to be prefixed with the role name to avoid collisions.

## 2. Nginx ↔ PHP-FPM connection: TCP vs Unix socket

**Current implementation: TCP (`php-app-php-fpm:9000`)**

**Why TCP:**
- Containers run in separate containers on a shared Docker network.
- TCP works across containers without shared filesystem access.
- Docker networking provides DNS resolution for container names.
- Simpler setup: no socket file permissions or volume mounts for sockets.

**When Unix sockets would be better:**
- Same host/container: lower latency, no network stack.
- Shared filesystem: can mount a socket directory.
- Security: no network exposure.

**Trade-off:** For containerized setups, TCP is the practical choice. If both services were in one container or shared a volume, a Unix socket would be preferable.


## 3. Idempotency in Ansible

**Mechanisms used:**
1. **Template module** — Only writes when content changes; reports `changed` only on diff.
2. **`changed_when: false`** — Used for read-only checks (container existence, directory creation, nginx -t) to avoid false positives.
3. **Conditional execution** — `when: rendered_* is defined` ensures tasks run only when templates are rendered.
4. **Handler notifications** — Services restart only when configs actually change.
5. **Docker container state** — `docker_container` ensures containers are running without unnecessary restarts.

**Example flow:**
```yaml
- name: Deploy nginx configuration on host
  template:
    src: nginx.conf.j2
    dest: "{{ web_nginx_conf_host_path }}"
  register: rendered_nginx
  notify: restart nginx container  # Only fires if rendered_nginx.changed == true
```

**Second run behavior:** Templates detect no changes → no file writes → no handler triggers → services remain running → playbook reports `changed=0`.
## 4. `/healthz` endpoint implementation

**What it does:**
```nginx
location /healthz {
    default_type application/json;
    return 200 '{"status":"ok","service":"nginx","env":"{{ web_app_env }}"}';
}
```

**How it works:**
- Nginx serves this directly (no PHP-FPM).
- Returns HTTP 200 with JSON: `{"status":"ok","service":"nginx","env":"dev"}`.
- Includes the current `app_env` from the template variable.

**Why `/healthz`:**
- Kubernetes/cloud-native convention for health checks.
- Lightweight: no PHP execution, minimal overhead.
- Useful for load balancer/container orchestrator probes.

**Limitations:**
- Only confirms Nginx is responding, not PHP-FPM availability.
- Doesn’t verify database connectivity or other dependencies.

**Improvement:** Add a PHP-FPM check:
```nginx
location /healthz {
    default_type application/json;
    access_log off;
    fastcgi_pass {{ web_php_fpm_upstream }};
    fastcgi_param SCRIPT_FILENAME /var/www/html/healthz.php;
    include fastcgi_params;
}
```
## 5. Improvements given more time

**1. Health checks:**
- ~~Add PHP-FPM health endpoint (`/healthz` via PHP-FPM or separate `/healthz-php`).~~
  Done added check it http://localhost:8080/healthz-php 
- Implement health/liveness probes.
- Add dependency checks (database, Redis, etc.).

**2. Security:**
- Harden Nginx (security headers).
- Use non-root containers.
- Add secret management (Ansible Vault, HashiCorp Vault).
- Add TLS/HTTPS.

**3. Observability:**
- Possibly metrics (Prometheus exporters).

**4. Configuration management:**
- Use Ansible Vault for sensitive values.
- Environment-specific variable files (`group_vars/prod/`, `group_vars/dev/`).
- Verify variables using `assert` tasks.

**5. Testing:**
- Molecular tests for Ansible role.
- Integration tests (container testing, endpoint testing).
- Terraform plan validation in CI.

**6. Infrastructure as code:**
- Terraform remote state (S3, Terraform Cloud).
- Terraform workspaces for environments.
- Use `terraform_remote_state` data source as needed.

**7. Deployment:**
- Blue-green deployments.
- Rollback mechanisms.
- Automated backups before changes.

**8. Performance:**
- Nginx caching (FastCGI cache, proxy cache).
- PHP-FPM setup (pool configuration, process management).
- CDN integration for static resources.

**9. Documentation:**
- README with architecture diagrams.
- Variable documentation.


