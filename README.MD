# nginx-deploy

OS-independent (ArchLinux / Debian / EL, all systemd) Ansible role that:

1. installs the `nginx` OS package through the local package manager;
2. deploys a **modular, declarative** reverse-proxy config templated from
   structured variables (not file-copy);
3. is **declarative**: if a vhost is removed from `nginx_vhosts`, the role
   removes its file from `conf.d/vhosts/` and reloads (orphan cleanup).

The role does not download nginx; it installs the OS package. It is inactive
while the `nginx` list variable is empty.

## Requirements

* Ansible 3.0.0+
* systemd
* `community.general` collection
* `nginx` available as an OS package (pacman / apt / dnf)

## On-disk layout produced

```
/etc/nginx/nginx.conf                      (template, user parametrized per-OS)
/etc/nginx/conf.d/443.conf                 (static)
/etc/nginx/conf.d/80.conf                  (static)
/etc/nginx/conf.d/defaults.conf            (static)
/etc/nginx/conf.d/header_checks.conf       (static)
/etc/nginx/conf.d/private.conf             (static)
/etc/nginx/conf.d/php_insert.conf          (static, used by locations/php.conf)
/etc/nginx/conf.d/tls.conf                 (template: SSL params + HSTS + CSP)
/etc/nginx/conf.d/<domain>_tls.conf        (template, per nginx_tls_certs)
/etc/nginx/conf.d/locations/50x.conf       (static)
/etc/nginx/conf.d/locations/main.conf      (static)
/etc/nginx/conf.d/locations/policy.conf    (static)
/etc/nginx/conf.d/locations/php.conf       (static)
/etc/nginx/conf.d/vhosts/noTLSreturn.conf  (template, http->https redirect)
/etc/nginx/conf.d/vhosts/<server_name>.conf(template, per nginx_vhosts)
```

## OS-specific defaults (`vars/<os_family>.yml`)

```yaml
nginx_package: 'nginx'
nginx_service: 'nginx'
nginx_conf_dir: '/etc/nginx'
nginx_user: 'http'        # Archlinux; Debian: www-data; RedHat: nginx
```

## Variables

```yaml
nginx: []                 # install/service control; empty = role inactive
nginx_tls_params: {}       # optional per-host override of tls.conf
nginx_tls_certs: []        # -> conf.d/<domain>_tls.conf
nginx_vhosts: []           # -> conf.d/vhosts/<server_name>.conf
nginx_http_extra: []       # raw directives injected into http {}
```

### `nginx_http_extra`

Directives that cannot live inside a `server {}` block — shared rate-limit and
connection zones, `map`, `geo`, `split_clients`. Rendered verbatim into the
`http {}` context of `nginx.conf`, one list item per line.

```yaml
nginx_http_extra:
  - 'limit_req_zone $binary_remote_addr zone=api_ip:10m rate=300r/m;'
  - 'limit_conn_zone $http_authorization zone=api_tok_conn:10m;'
```

Zones are global, so keep names unique across all vhosts on the host. The
zones themselves do nothing until a vhost references them via `limit_req` /
`limit_conn` (per-vhost `extra:` or a location).

### Install/service block

```yaml
nginx:
  - enable: true           # systemctl enable
    started: true          # systemctl start
    package_state: present # present | latest
```

### TLS params override

`nginx_tls_params` is merged over the role defaults
(`nginx_tls_params_defaults`); set only the keys you want to change
(`hsts`, `csp_report_only`, `ssl_ciphers`, `ssl_stapling_verify`, ...).

### Certificates

```yaml
nginx_tls_certs:
  - domain: mail-ok.ru     # cert dir /opt/certificates/mail-ok.ru/
```

### Vhosts

```yaml
nginx_vhosts:
  - server_name: asterisk.mail-ok.ru
    tls_domain: mail-ok.ru
    includes: [50x, main]            # locations/* snippets to include
    include_private: false           # optional: include conf.d/private.conf
    client_max_body_size: 1024M
    client_body_buffer_size: 128k    # optional
    proxy_timeouts: {connect: 300s, send: 300s, read: 300s}
    proxy_buffering: off
    proxy_request_buffering: off     # optional
    root: /srv/http/site             # optional
    upstreams:
      - { name: bx, servers: ['10.110.100.1:8077'], keepalive: 32 }
    locations:
      - { path: '/', proxy_pass: 'http://bx', websocket: true, allow: all }
      - { path: '^~ /records/', proxy_pass: 'http://bx2/', allow: all,
          timeouts: {connect: 7d, send: 7d, read: 7d} }
    extra: []                        # raw server-level directive lines
```

Each proxy `location` emits `proxy_pass`, `proxy_http_version 1.1` and the
standard six forwarded headers (`Host`, `X-Real-IP`, `X-Forwarded-For`,
`X-Forwarded-Proto`, `X-Forwarded-Host`, `X-Forwarded-Port`). When
`websocket: true` it also emits `Upgrade`/`Connection "upgrade"`. Per-location
`timeouts` and `allow` are optional; `extra` injects raw directive lines.

## Declarative cleanup

After templating the configured vhosts, the role:

1. builds the managed filename set
   (`noTLSreturn.conf` plus `<server_name>.conf` for each `nginx_vhosts`
   entry);
2. `find`s every `*.conf` in `conf.d/vhosts/`;
3. removes any file whose basename is not in the managed set and notifies
   `reload nginx`.

Removing a vhost from `nginx_vhosts` therefore deletes its file on the next
run. `noTLSreturn.conf` is always preserved.

## Validation and reload

* `nginx.conf` is templated with `validate: nginx -t -c %s`.
* The `reload nginx` handler runs `nginx -t` first, then a systemd reload,
  so vhost/snippet changes are validated before nginx reloads.

## Playbook example

```yaml
---
- name: Deploy nginx
  become: true
  gather_facts: true
  remote_user: root
  roles:
    - nginx-deploy
  hosts: gw.mail-ok.ru
```
