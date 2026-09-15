# Ansible-dynamic-assignments
The objective of this assignment was to extend an existing Ansible configuration management project by introducing a configurable load-balancing layer for the UAT environment.
# Ansible Load Balancer Configuration

## 1. Business / Technical Problem

In a production-like environment, applications may need to distribute incoming traffic across multiple web servers instead of relying on a single server.

The objective of this assignment was to extend an existing Ansible configuration management project by introducing a configurable load-balancing layer for the UAT environment.

The solution needed to support both **Nginx** and **Apache** as load balancers, with the ability to select which load balancer was active through environment-specific variables.

## 2. Architecture

The UAT environment consists of:
                    Users / Clients
                           |
                           v
                +----------------------+
                |   Load Balancer      |
                |   172.31.10.238      |
                |                      |
                | Nginx OR Apache      |
                +----------+-----------+
                           |
                +----------+----------+
                |                     |
                v                     v
       +----------------+    +----------------+
       | UAT Web Server |    | UAT Web Server |
       | 172.31.43.209  |    | 172.31.42.66   |
       | Apache         |    | Apache         |
       +----------------+    +----------------+

The load balancer was controlled through environment variables in:

env-vars/uat.yml

For Nginx:
yaml
enable_nginx_lb: true
enable_apache_lb: false
load_balancer_is_required: true

For Apache:

yaml
enable_nginx_lb: false
enable_apache_lb: true
load_balancer_is_required: true

## 3. Technologies Used

* Ansible
* Ansible Roles
* Nginx
* Apache HTTP Server
* Ubuntu
* Red Hat Enterprise Linux
* AWS EC2
* Git
* GitHub
* YAML
* SSH
* HTTP
* Reverse Proxy
* Environment-specific configuration


## 4. What I Implemented

### 4.1 Load Balancer Playbook

Created:
text
static-assignments/loadbalancers.yml

The playbook uses conditional role execution:

```yaml
- hosts: lb
  become: true
  roles:
    - { role: nginx, when: enable_nginx_lb and load_balancer_is_required }
    - { role: apache, when: enable_apache_lb and load_balancer_is_required }

This allows the environment configuration to determine which load balancer is deployed.

### 4.2 Environment-Specific Configuration

Updated:
text
env-vars/uat.yml

The configuration supports switching between Nginx and Apache without changing the main playbook.

For the Nginx test:

yaml
enable_nginx_lb: true
enable_apache_lb: false
load_balancer_is_required: true

For the Apache test:
yaml
enable_nginx_lb: false
enable_apache_lb: true
load_balancer_is_required: true


### 4.3 Nginx Load Balancer

Configured Nginx as a reverse proxy for the two UAT web servers.

The upstream configuration was:

nginx
upstream uat_webservers {
    server 172.31.43.209;
    server 172.31.42.66;
}


Traffic was forwarded to the upstream using:
proxy_pass http://uat_webservers;


Additional proxy headers were configured to preserve request information.

The default Nginx virtual host was also removed so that the default Nginx welcome page would not take precedence over the load-balancer configuration.

### 4.4 Apache Load Balancer

The same environment configuration was tested with Apache enabled instead of Nginx.

Nginx was disabled and Apache was allowed to use port 80.

The Apache configuration was verified with:

apache2ctl configtest


The result was:

Syntax OK

Apache was then successfully started and verified as active.


## 5. Challenges Encountered

### Challenge 1: Ansible privilege issue

During the initial Nginx deployment, Ansible encountered a permission problem while running package management tasks.

The issue was resolved by enabling privilege escalation in the load-balancer playbook:

become: true

This allowed package installation and service management tasks to execute with the required privileges.

### Challenge 2: Nginx default virtual host

After configuring the Nginx upstream, the load balancer was initially still capable of serving the default Nginx page.

Inspection of the Nginx configuration showed that the default virtual host was still listening on port 80.

The solution was to enable:

yaml
nginx_remove_default_vhost: true

After rerunning the playbook, the default virtual host was removed.


### Challenge 3: Apache could not start

When switching the UAT environment from Nginx to Apache, Apache initially failed to start.

The first step was to validate the Apache configuration:

apache2ctl configtest

The result was:
Syntax OK

This showed that the Apache configuration syntax was valid.

The next troubleshooting step checked which process was listening on port 80:

ss -ltnp | grep ':80'

The output showed that Nginx was still listening on port 80.

Nginx was stopped and disabled:

systemctl stop nginx
systemctl disable nginx

The Ansible playbook was then executed again.

Apache started successfully.

## 6. Troubleshooting Approach

The troubleshooting process followed a structured approach:

1. Identify the failed Ansible task.
2. Validate the affected service configuration.
3. Check whether another service was using the required port.
4. Identify the process listening on port 80.
5. Stop and disable the conflicting service.
6. Rerun the Ansible playbook.
7. Verify the service status.

This demonstrated the importance of checking the actual system state rather than assuming that an Ansible failure means the configuration itself is invalid.

## 7. Testing and Verification

### Nginx Load Balancer

The Nginx service was verified with:

systemctl is-active nginx

Result:
active

The Nginx configuration was also inspected using:

nginx -T


The upstream configuration showed both UAT web servers:
upstream uat_webservers {
    server 172.31.43.209;
    server 172.31.42.66;
}

The load balancer was tested with:

curl -I http://172.31.10.238

The response confirmed that traffic was reaching the backend application servers through Nginx.

### Apache Load Balancer

Apache configuration syntax was validated:

apache2ctl configtest

Result:
Syntax OK

Apache was then verified using:

systemctl is-active apache2

Result:
active

The complete Ansible playbook also completed successfully:

172.31.10.238 : ok=17 changed=1 failed=0
172.31.42.66  : ok=9  changed=3 failed=0
172.31.43.209 : ok=9  changed=3 failed=0

The final result had:

failed=0

## 8. Git and GitHub Workflow

The work was developed on the `roles-feature` branch.

The Nginx load-balancer implementation was committed as:

112d692 Configure Nginx load balancer for UAT


The changes were pushed to GitHub and merged into the `main` branch through a Pull Request.

The Apache load-balancer test was then committed as:

de9a5b2 Test Apache load balancer for UAT

The commit was pushed to GitHub and merged into `main`.

The project therefore has a complete Git history showing the implementation and testing process.


## 9. Final Outcome

The load-balancer assignment was successfully completed.

The final solution supports:

* Nginx as a load balancer
* Apache as a load balancer
* Environment-based load-balancer selection
* Conditional deployment using Ansible variables
* Two UAT backend web servers
* Nginx reverse-proxy configuration
* Automated service configuration through Ansible
* Successful testing of both Nginx and Apache

The UAT environment can switch between Nginx and Apache by changing the appropriate environment variables rather than modifying the main load-balancer playbook.

## 10. Lessons Learned

This assignment strengthened my practical understanding of:

* Ansible roles and conditional execution
* Environment-specific configuration
* Reverse proxy and load-balancing concepts
* Nginx configuration
* Apache service management
* Linux service troubleshooting
* Port conflicts
* Ansible privilege escalation
* Git branching and Pull Requests
* Testing infrastructure changes after deployment

One of the most valuable lessons was that successful automation requires both **configuration management and troubleshooting skills**. When Apache failed to start, checking the service configuration and then checking which process was using port 80 helped identify the actual cause quickly.


## 11. Project Status

Status: Completed*

The Nginx and Apache load-balancer configurations were successfully tested, committed, pushed to GitHub, and merged into the `main` branch.

