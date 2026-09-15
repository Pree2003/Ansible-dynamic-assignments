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

<img width="554" height="375" alt="image" src="https://github.com/user-attachments/assets/6ac08f5a-2815-49c0-9d86-bffc8f41b212" />

<img width="554" height="505" alt="image" src="https://github.com/user-attachments/assets/7fc27496-73cf-47d9-b270-493f31aba0a0" />

<img width="553" height="349" alt="image" src="https://github.com/user-attachments/assets/62e048ee-ef6a-42cc-946b-24b932654ddc" />

<img width="472" height="242" alt="image" src="https://github.com/user-attachments/assets/19a5b876-b6d7-400e-97f4-8d18dbb23d2e" />

<img width="554" height="497" alt="image" src="https://github.com/user-attachments/assets/f1a10afc-e902-4eef-ae6c-e3a7188d41dd" />

<img width="554" height="386" alt="image" src="https://github.com/user-attachments/assets/7c111324-b924-443f-a05e-c86d45092f4a" />

<img width="554" height="483" alt="image" src="https://github.com/user-attachments/assets/e4a466ee-5199-4f8a-95a1-b5e688f691a9" />

<img width="554" height="172" alt="image" src="https://github.com/user-attachments/assets/ec6721aa-9827-4229-ad0e-40bf1d85a902" />

<img width="489" height="132" alt="image" src="https://github.com/user-attachments/assets/a7731719-a03b-4075-9353-d34405a4883a" />

<img width="479" height="170" alt="image" src="https://github.com/user-attachments/assets/b3369693-e85b-440a-b6b0-1160f8cf069c" />

<img width="554" height="83" alt="image" src="https://github.com/user-attachments/assets/1e4522ac-3636-4ec8-a065-e999699e3bfb" />

<img width="553" height="223" alt="image" src="https://github.com/user-attachments/assets/2a2bec38-b460-49b6-8b96-a77130353dd3" />

<img width="553" height="211" alt="image" src="https://github.com/user-attachments/assets/64b51931-a170-42a3-8515-78f6ab54ba8c" />

<img width="553" height="162" alt="image" src="https://github.com/user-attachments/assets/7efe6d0a-c463-4456-a35e-91030c5134f5" />

<img width="553" height="230" alt="image" src="https://github.com/user-attachments/assets/35d18007-ffce-42cf-9589-393d3987ee84" />

<img width="514" height="534" alt="image" src="https://github.com/user-attachments/assets/3953f3d6-32b3-4414-8e65-04bd128feddb" />

<img width="554" height="426" alt="image" src="https://github.com/user-attachments/assets/b6f7a23a-fc0c-4c29-8514-6c2cc2690f8d" />

<img width="554" height="266" alt="image" src="https://github.com/user-attachments/assets/23652f7f-40ec-46b1-a365-ac1359f50abb" />

<img width="554" height="280" alt="image" src="https://github.com/user-attachments/assets/0adb0e23-e926-465f-aae1-ff770e6516ce" />

<img width="554" height="213" alt="image" src="https://github.com/user-attachments/assets/c174b6dc-26ce-4cc6-9046-e5d3f42ef6ed" />

<img width="554" height="283" alt="image" src="https://github.com/user-attachments/assets/50fa770f-931f-4768-a8fc-b5a1b05020d1" />

<img width="554" height="437" alt="image" src="https://github.com/user-attachments/assets/34f5cd98-00ec-4e3e-8daa-83a214d11373" />

<img width="518" height="249" alt="image" src="https://github.com/user-attachments/assets/97c7698e-5673-4c47-a5b4-a73d0b10c6d6" />

<img width="554" height="216" alt="image" src="https://github.com/user-attachments/assets/a8bf6d00-a6ed-43f7-a3e2-e8a6e8d96528" />

<img width="554" height="143" alt="image" src="https://github.com/user-attachments/assets/41b2d416-3d3a-4499-af28-d9a3f296d281" />

<img width="493" height="125" alt="image" src="https://github.com/user-attachments/assets/ddba0944-fcb7-4db3-8cc6-191a61fb3d92" />

<img width="554" height="135" alt="image" src="https://github.com/user-attachments/assets/e6fed450-969f-4747-b81f-970096f451dc" />

<img width="554" height="238" alt="image" src="https://github.com/user-attachments/assets/0e8deb28-57d6-418e-b32b-f85bd4347ae1" />

<img width="554" height="239" alt="image" src="https://github.com/user-attachments/assets/b0b53582-19bd-4cb6-b036-cd72b2e18dc0" />

<img width="554" height="268" alt="image" src="https://github.com/user-attachments/assets/cc74cd9a-c1cd-4cd8-bb90-3691afb8bc31" />

<img width="553" height="267" alt="image" src="https://github.com/user-attachments/assets/1521fd7d-a004-4a6b-83b1-bf3ab670d99d" />

<img width="553" height="191" alt="image" src="https://github.com/user-attachments/assets/f0ae8520-23af-4c1b-b62c-fd8585380982" />

<img width="554" height="406" alt="image" src="https://github.com/user-attachments/assets/2ec2c9ae-7f72-41a9-b1f1-28ef71c58d12" />

<img width="554" height="261" alt="image" src="https://github.com/user-attachments/assets/d1becfed-9198-42fb-b249-968e17214fa0" />

<img width="554" height="171" alt="image" src="https://github.com/user-attachments/assets/545a6e01-f2be-473f-8868-d25f24e4db90" />

<img width="554" height="271" alt="image" src="https://github.com/user-attachments/assets/860d3eea-5622-4927-aaee-b704a813f214" />

<img width="553" height="73" alt="image" src="https://github.com/user-attachments/assets/1f68d73f-0097-4f6c-8503-e69d0c7c318f" />

<img width="553" height="276" alt="image" src="https://github.com/user-attachments/assets/0e7da5f4-efc1-42b7-bd33-64d6297286a7" />

<img width="554" height="230" alt="image" src="https://github.com/user-attachments/assets/693d4543-d15a-41f0-96bd-2657df90939b" />

<img width="554" height="153" alt="image" src="https://github.com/user-attachments/assets/8fee878c-c5bb-42a9-8eb3-361a72b6acd9" />

<img width="554" height="266" alt="image" src="https://github.com/user-attachments/assets/b1810bfd-76d6-4370-926e-207860574b21" />

<img width="554" height="266" alt="image" src="https://github.com/user-attachments/assets/986c26eb-237b-4bb0-8373-09d54e094b14" />

<img width="553" height="266" alt="image" src="https://github.com/user-attachments/assets/9abd6e9d-d411-4037-ba24-43adb3f75b99" />

<img width="553" height="267" alt="image" src="https://github.com/user-attachments/assets/104564a9-1c0b-47d2-8c5e-ef7b73d8e610" />

<img width="553" height="65" alt="image" src="https://github.com/user-attachments/assets/5ceabccd-736e-4ae7-8ec9-c5a1709d077a" />

<img width="554" height="270" alt="image" src="https://github.com/user-attachments/assets/726770ad-9408-4a02-8aec-09990b037e65" />

<img width="517" height="366" alt="image" src="https://github.com/user-attachments/assets/cb6e902c-23ab-4998-91a1-62dcbe1ddeb5" />

<img width="526" height="195" alt="image" src="https://github.com/user-attachments/assets/fcfc9854-fd3b-413d-a7e8-598f1d00fd22" />

<img width="553" height="256" alt="image" src="https://github.com/user-attachments/assets/074fdf70-f361-4a46-b601-71432ae7a5f3" />

<img width="553" height="278" alt="image" src="https://github.com/user-attachments/assets/8c36af6d-e4a9-453f-b383-f40488daf28c" />

<img width="553" height="266" alt="image" src="https://github.com/user-attachments/assets/54c90c7c-a595-4d82-bcff-f26b45a9facd" />

<img width="533" height="213" alt="image" src="https://github.com/user-attachments/assets/68e1bf99-5e05-4cbe-b93b-8926a6796abd" />

<img width="554" height="144" alt="image" src="https://github.com/user-attachments/assets/eb10b815-e082-457d-bb6b-e423507eec84" />

<img width="553" height="271" alt="image" src="https://github.com/user-attachments/assets/70407c68-b949-4c22-bbdf-1e26e0c0948a" />

<img width="521" height="544" alt="image" src="https://github.com/user-attachments/assets/aa559138-dc16-4933-88c7-55126d4b0914" />

<img width="554" height="424" alt="image" src="https://github.com/user-attachments/assets/44287d25-7912-436b-86f3-2dc067e12d52" />

<img width="554" height="231" alt="image" src="https://github.com/user-attachments/assets/f8a55014-5d80-4834-b925-dd6416f971ab" />

<img width="554" height="139" alt="image" src="https://github.com/user-attachments/assets/6543cb16-94ea-492e-a6b0-53ebe41871c5" />

<img width="553" height="205" alt="image" src="https://github.com/user-attachments/assets/cb2f134f-5725-48fe-ba43-c622ff205f73" />

<img width="554" height="156" alt="image" src="https://github.com/user-attachments/assets/d79e75e5-f376-437c-95d3-f5b4b93dcf83" />

