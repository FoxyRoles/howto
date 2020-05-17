# Ansible HOWTO

## Lists

### Format shorten package names using another variable
```yaml
---
# playbook
vars:
  php_version: 7.2
  php_modules:
    - apc
    - opcache

---
# task
- apt:
  pkg: "{{ php_modules | map('regex_replace', '(.*)', 'php%s-\\1' % php_version) | list }}"
  state: present
# Result:
# pkg: "php7.2-apc php7.2-opcache"
```

## Dicts

### Loop dict as key => value pairs
```yaml
---
# playbook
vars:
  php_ini_settings:
    expose_php: Off
    max_execution_time: 120

---
# task
- lineinfile:
    dest: "/etc/php/7.2/fpm/php.ini"
    regexp: "^;?{{ item.key }}"
    line: "{{ item.key }} = {{ item.value }}"
  when: "php_ini_settings is defined"
  with_dict:
    - "{{ php_ini_settings }}"
```

## Password

### Generate password when not set or saved
```yaml
---
# playbook
vars:
  local_passdb: ~/ansible/passwords
  mysql_users:
    - name: john
      host: localhost
      database: john

---
# role default vars
mysql_local_passdb: "{{ local_passdb | default('~/.ansible_mysql_passwords') }}"
mysql_pass_len: 24
mysql_pass_complexity: "ascii_letters,digits" # ascii_letters,digits,hexdigits,punctuation
mysql_root_password: "{{ lookup('password', [mysql_local_passdb, '/', inventory_hostname, '/', 'mysql.root', ' chars={{ mysql_pass_complexity }} length={{ mysql_pass_len }}']|join) }}"

---
# task
- name: create users
  mysql_user:
    login_unix_socket: /run/mysqld/mysqld.sock
    name: "{{ item.name }}"
    password: "{{ item.password | default(lookup('password', [mysql_local_passdb, '/', inventory_hostname, '/', 'percona.', item.name, ' chars={{ mysql_pass_complexity }} length={{ mysql_pass_len }}']|join)) }}"
    host: "{{ item.host }}"
    priv: "{{ item.database }}.*:ALL"
    state: present
  with_items: "{{ mysql_users }}"
  when: mysql_users is defined
  no_log: true
```

## Templates

### Send custom variables to template
```yaml
---
# playbook
vars:
  php_version: 7.2
  php_pools:
    web:
      port: 9001

---
# task
- template:
    src: template.file.j2
    dest: "/etc/php/7.2/fpm/pool.d/web.conf"
  vars:
    pool: "{{ item.key }}"
    user: "{{ item.value.user | default('www-data') }}"
    port: "{{ item.value.port | default('90' + (php_version | regex_replace('\\.', ''))) }}"
  with_dict:
    - "{{ php_pools }}"
```

## Misc

### Generate file only once
```yaml
---
# task
- name: Ensure Diffie Hellman Ephemeral Parameters file is generated
  command: openssl dhparam -out /etc/nginx/ssl/dhparam4096.pem 4096
  args:
    creates: /etc/nginx/ssl/dhparam4096.pem
```
