# Inventory
### For scanning
```
$cat hosts_scanning
localhost ansible_connection=local

[nginx]
192.168.122.xxx ansible_ssh_user=xxx ansible_ssh_pass=xxx ansible_sudo_pass=xxx ansible_port=xxx
```



# Ansible Playbook

### See the variable values in vars/main.yml for scanning.
```
ls -ltrh <code_path>/roles/nginx-hygiene-scanning/vars/main.yml
```
```
cat <code_path>/roles/nginx-hygiene-scanning/vars/main.yml 
---
# vars file for nginx-hygiene-scanning
nginxcsv: "files/nginx-hygiene-{{ inventory_hostname }}.csv"
n_conf_path: "/etc/nginx/nginx.conf"
nginx_path: "/etc/nginx/"
nginx_name: "nginx"
nginx_module: "modsecurity"
url: "http://localhost:80"
nginx_extra_module: "mod_include|mod_info|mod_status|mod_dav|mod_userdir"
```

### Want to skip policies then check defaults/main.yml for scanning.
```
ls -ltrh <code_path>/roles/nginx-hygiene-scanning/defaults/main.yml
```
```
cat <code_path>/roles/nginx-hygiene-scanning/defaults/main.yml
rule_2_1_1: true
rule_2_1_2: true
rule_2_1_3: true
rule_2_1_4: true
rule_2_1_5: true
rule_2_1_6: true
rule_2_1_7: true
rule_2_1_8: true
rule_2_1_9: true
rule_2_1_10: true
rule_2_1_11: true
```
### Run Scanning Playbook
```
ansible-playbook -i hosts_scanning scanning-playbook.yml
```

##### Report Path by default
```
ls -ltrh /var/lib/awx/report/v3/<hostname>.html
```

#### Report Format
```
| Policies                                                                   | Security Level |
|----------------------------------------------------------------------------|----------------|
| 2.1.1 Web server type and version disclosure                               | Level1         |
| 2.1.2 Trace method enable on the remote server                             | Level1         |
| 2.1.3 Check if XSS Protection enabled or not to avoid Cross-site Scripting | Level1         |
| 2.1.4 Check Configure SSL and Cipher Suites                                | Level1         |
| 2.1.5 Privileged user account                                              | Level1         |
| 2.1.6 Verifiy if Certain Modules are enabled in nginx modules              | Level1         |
| 2.1.7 Check Nginx Logging Parameters & format                              | Level1         |
| 2.1.8 Check if Unwanted HTTP methods are disabled                          | Level1         |
| 2.1.9 Check if selected module is enabled                                  | Level1         |
| 2.1.10 Check Server version leakage in nginx                               | Level1         |
| 2.1.11 Server name comes in response header in nginx                       | Level1         |
```
