# Five Ways to Automate F5OS with Ansible: A Practical Guide 

This article explores five methods for automating F5OS using Ansible, complete with real-world examples you can adapt to your own environment.

## Understanding F5OS Architecture

Our example system is an F5OS appliance named `r5900-2`. 

When you connect to an F5OS device terminal as the root user, you enter a Linux Bash shell:

```console
[root@appliance-1(r5900-2):Active] ~ # uname -r
3.10.0-1160.71.1.F5.1.1.1.el7_8.x86_64
[root@appliance-1(r5900-2):Active] ~ # cat /etc/centos-release 
CentOS Linux release 7.8.2003 (Core)
[root@appliance-1(r5900-2):Active] ~ #
```

Root access is rarely needed to manage an F5OS appliance. Most tasks can be accomplished via the [F5OS CLI](https://clouddocs.f5.com/api/rseries-api/F5OS-A-1.8.0-cli.html).

If you 'substitute user' to the 'admin' user (via the `su` command) or SSH directly as admin@r5900-2, you enter the F5OS CLI. Here you configure low-level network components such as Link Aggregation Groups and VLANs, monitor the F5OS platform, and manage BIG-IP tenants.

```console
[root@appliance-1(r5900-2):Active] ~ # su admin
Welcome to the Management CLI
admin connected from 172.18.7.92 using ssh on r5900-2 
r5900-2# show system version
system version os-version 1.8.0-16036
system version service-version 1.8.0-16036 
system version product F5OS-A
```

The F5OS CLI is built on top of the [F5OS API](https://clouddocs.f5.com/api/rseries-api/rseries-api-index.html). With this same admin account, you can target the F5OS API to configure anything at the F5OS layer. A simple test with 'curl' can confirm you are ready to automate via the F5OS API:

```console
$ curl -k -X GET \
f/data/openconfi>   "https://r5900-2:8888/restconf/data/openconfig-system:system/f5-system-version:version" \
H "Content-Type:>   -H "Content-Type: application/yang-data+json" \
>   -H "Accept: application/yang-data+json" \
>   -u admin:'YOUR_PASSWORD'
{
  "f5-system-version:version": {
    "os-version": "1.8.0-16036",
    "service-version": "1.8.0-16036",
    "product": "F5OS-A"
  }
}
```

When you create a BIG-IP tenant, it behaves like a traditional BIG-IP instance. The tenant has its own isolated management IP address and [iControl REST API](https://clouddocs.f5.com/api/icontrol-rest/) endpoint. Your [existing BIG-IP Ansible playbooks](https://clouddocs.f5.com/products/orchestration/ansible/devel/f5_bigip/f5_bigip.html)
will continue to work with minimal changes.  

However, automation workflows for BIG-IP are not compatible with F5OS. Let's explore how we can automate just about anything on F5OS using Ansible.

## Method 1: F5OS Ansible Collection Modules

```yaml
# Method 1: Using F5OS collection modules
- name: Get F5OS system information using F5OS Ansible module
  f5os_device_info:
    gather_subset:
      - system-info
  register: f5os_facts
```

**Mechanics:** This method uses the [official F5OS Ansible collection](https://clouddocs.f5.com/products/orchestration/ansible/devel/f5os/F5OS-index.html) to communicate with the F5OS REST API over HTTPS. The modules handle authentication, request formatting, and response parsing.

**Pros:** 
- Purpose-built for F5OS operations
- Input validation and error handling 
- Consistent interface across F5OS versions
- Simplified syntax for complex operations

**Cons:**
- Limited to functionality provided by existing modules 
- May need to wait for new modules as F5OS features evolve

## Method 2: Direct REST API Calls

```yaml
# Method 2: Using generic Ansible uri module for REST API
- name: Get F5OS version information using REST API direct call
  uri:
    url: "{{ api_base_url }}/data/openconfig-system:system/f5-system-version:version"
    method: GET
    force_basic_auth: true 
    user: "{{ ansible_user }}"
    password: "{{ ansible_httpapi_password }}"
    headers:
      Content-Type: "application/yang-data+json"
      Accept: "application/yang-data+json"  
    validate_certs: "{{ ansible_httpapi_validate_certs }}"
    status_code: 200
    return_content: true
  register: f5os_version
```

**Mechanics:** This approach uses [Ansible's uri module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html) to make direct REST API calls to F5OS. It provides maximum flexibility but requires detailed knowledge of the F5OS API.

**Pros:**
- Access to all F5OS API endpoints 
- No dependency on F5OS collection modules
- Useful for testing new F5OS features

**Cons:** 
- Requires understanding of F5OS API structure
- More verbose configuration
- Need to handle response parsing manually

## Method 3: Linux System Access 

```yaml
# Method 3: Using generic Ansible SSH for Linux system access 
- name: Gather Linux system information via SSH
  setup:
    gather_subset:
      - hardware  
  delegate_to: r5900-2-linux
  vars:
    ansible_user: root
    ansible_connection: ssh
  register: linux_facts
```

**Mechanics:** This method establishes an SSH connection to access the underlying Linux operating system for system-level operations and diagnostics.

**Pros:**
- Direct access to system resources
- Useful for troubleshooting 
- Leverage existing Linux automation skills

**Cons:**
- Requires root access
- Risk of affecting system stability

## Method 4: F5OS CLI via SSH

```yaml 
# Method 4: Using generic Ansible shell module for F5OS CLI
- name: Get F5OS version information via CLI 
  shell: |
    sshpass -p "{{ f5os_cli_password }}" ssh -t -o StrictHostKeyChecking=no admin@{{ hostvars['r5900-2-cli']['ansible_host'] }} << 'EOF'
    show sys version
    EOF
  delegate_to: localhost
  register: cli_version
```

**Mechanics:** This method uses SSH to connect directly to the F5OS CLI, similar to how you interact with the device manually. 

**Pros:**
- Familiar CLI interface
- Suitable for operational commands
- Easy to test and debug

**Cons:**
- Output parsing can be complex 
- Less idempotent than API methods
- Requires sshpass installation

## Method 5: Bash Scripts with f5sh

```yaml
# Method 5: Using Ansible copy and shell modules to run f5sh commands from Bash scripts
- name: Transfer f5sh_example.sh script to F5OS device
  copy:
    content: |
      #!/bin/bash
      f5sh show sys version
    dest: "/root/f5sh_example.sh"
    mode: '0755'
  delegate_to: r5900-2-linux
  vars:
    ansible_user: root
    ansible_connection: ssh
  register: script_transfer

- name: Execute f5sh_example.sh script on F5OS device
  shell: "/root/f5sh_example.sh"
  delegate_to: r5900-2-linux
  vars:
    ansible_user: root
    ansible_connection: ssh
  register: script_output 
```

**Mechanics:** This method combines Linux shell access with the [`f5sh` command prefix introduced in F5OS 1.8.0](https://techdocs.f5.com/en-us/f5os-a-1-8-0/relnote-f5os-a-1-8-0/title-new-in.html), allowing F5OS CLI commands to be run from Bash scripts.

**Pros:** 
- Combine shell scripting with F5OS commands

**Cons:**
- Requires root access

## Inventory and Connection Methods

Each connection method requires specific inventory settings defined in `hosts.yml`:

### API Connections 
```yaml
# hosts.yml 
f5os:
  hosts:
    r5900-2-api:
      ansible_host: r5900-2
```

Used with `httpapi` connection in the playbook:
```yaml
# playbook.yml
- name: F5OS Management Tasks
  connection: httpapi
  hosts: r5900-2-api
  vars:
    ansible_user: "admin" 
    ansible_httpapi_password: "{{ f5os_api_password }}"
    ansible_network_os: "f5networks.f5os.f5os"
    ansible_httpapi_use_ssl: true
    ansible_httpapi_port: 8888
```

You may send API calls to either port 8888 or port 443. The URI path will change slightly depending on which TCP port you choose to use:
- For port 443: Initial path will be `/api`
- For port 8888: Initial path will be `/restconf`
 
You can then configure distinct network access control polices for traffic to:
- Management interface user interface (443)
- Management interface REST API endpoint (8888)

### CLI Access
```yaml
# hosts.yml
f5os_cli:  
  hosts:
    r5900-2-cli:
      ansible_host: r5900-2
      ansible_user: admin
      ansible_connection: ssh
      ansible_become: no 
      ansible_ssh_common_args: '-o StrictHostKeyChecking=no'
```

Enables CLI commands through SSH:
```yaml 
# playbook.yml  
- name: Get F5OS version information via CLI
  shell: |
    sshpass -p "{{ f5os_cli_password }}" ssh -t -o StrictHostKeyChecking=no admin@{{ hostvars['r5900-2-cli']['ansible_host'] }} << 'EOF'  
    show sys version
    EOF
  delegate_to: localhost
```

### Linux Shell Access
```yaml
# hosts.yml
linux_hosts:
  hosts: 
    r5900-2-linux:
      ansible_host: r5900-2
      ansible_user: root
      ansible_connection: ssh
      ansible_become: no
      ansible_python_interpreter: /usr/bin/python3
      ansible_ssh_common_args: '-o StrictHostKeyChecking=no'  
```

Used for system information gathering and script execution:
```yaml
# playbook.yml
- name: Gather Linux system information via SSH
  setup:
    gather_subset:
      - hardware
  delegate_to: r5900-2-linux 
  vars:
    ansible_user: root
    ansible_connection: ssh
```

### Notes on `hosts.yml` Inventory Entries

The inventory defines three connections to the same device:
- `r5900-2-api`: HTTPS API access 
- `r5900-2-cli`: SSH CLI access with admin user
- `r5900-2-linux`: SSH root access

SSH configuration: 
- `StrictHostKeyChecking=no`: Skips host key verification
- `ansible_become: no`: No privilege escalation 
- Custom Python path for Linux shell access

Run playbooks with `--ask-pass` to prompt for the root password when accessing the Linux shell:

```bash
ansible-playbook -i hosts.yml playbook.yml --ask-pass
```
See the complete `hosts.yml` and `playbook.yml` examples below that you can use as templates for your own environments. The latest versions are hosted on [GitHub](https://github.com/tmarfil/ansible_f5os_tmos_linux).

```yaml
# hosts.yml - Ansible inventory configuration
# This inventory file defines multiple ways to connect to the same F5OS device

all:
  children:
    # F5OS API group - For REST API interactions via HTTPS
    # This method is used for F5OS platform configuration and monitoring via REST
    # User credentials configured as vars in playbook.yml
    f5os:
      hosts:
        r5900-2-api:
          ansible_host: r5900-2

    # F5OS CLI group - For F5OS CLI access via SSH
    # This method is used to run F5OS CLI commands via SSH using admin account
    # User credentials prompted for at runtime
    f5os_cli:
      hosts:
        r5900-2-cli:
          ansible_host: r5900-2
          ansible_user: admin
          ansible_connection: ssh
          ansible_become: no
          ansible_ssh_common_args: '-o StrictHostKeyChecking=no'

    # Linux group - For direct Linux shell access via SSH
    # This method is used to access the underlying Linux OS using root account
    # User credentials prompted for at runtime using --ask-pass
    linux_hosts:
      hosts:
        r5900-2-linux:
          ansible_host: r5900-2
          ansible_user: root
          ansible_connection: ssh
          ansible_become: no
          ansible_python_interpreter: /usr/bin/python3
          ansible_ssh_common_args: '-o StrictHostKeyChecking=no'
```

```yaml
# playbook.yml
#
# This playbook demonstrates multiple methods to interact with F5OS devices using Ansible:
# 1. Using F5OS-specific Ansible modules from the F5 Ansible collection
#    - Connects via HTTPS to F5OS REST API
#    - Uses F5OS maintained collection modules
#    - Best for F5OS platform operations with verified modules
#
# 2. Using Ansible uri module to make direct REST API calls
#    - Connects via HTTPS to F5OS REST API
#    - Uses Ansible built-in uri module
#    - Best for custom REST API operations or when collection modules unavailable
#
# 3. Using SSH connection to access the underlying Linux OS
#    - Connects via SSH as root to Linux shell
#    - Uses Ansible built-in setup modules
#    - Best for system-level operations and diagnostics
#
# 4. Using SSH to connect as admin and run F5OS CLI commands
#    - Connects via SSH as admin to F5OS CLI
#    - Uses Ansible built-in shell module with sshpass
#    - Best for running operational commands and gathering CLI output
#
# 5. Using SSH connection to access the underlying Linux OS
#    - Connects via SSH as root to Linux shell
#    - Uses Ansible built-in SSH modules
#    - Uses Ansible built-in copy module to transfer Bash script
#    - Uses Ansible built-in shell module to execute script and display output
#    - Best for automated script-based management via f5sh commands
#
# Requirements:
# - sshpass must be installed on the Ansible control node for F5OS CLI access
#   Ubuntu/Debian: sudo apt-get install sshpass
#   RHEL/CentOS: sudo yum install sshpass
#
# References:
# - F5OS Collection: https://clouddocs.f5.com/products/orchestration/ansible/devel/f5os/F5OS-index.html
# - Ansible URI Module: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html
# - F5OS-A/F5 rSeries - API: https://clouddocs.f5.com/api/rseries-api/rseries-api-index.html
# - Ansible SSH Connection: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ssh_connection.html
# - F5OS-A/F5 rSeries - CLI: https://clouddocs.f5.com/api/rseries-api/rseries-cli-index.html
#
# Debug Notes:
# - Each connection method has debug lines (commented out) that can be enabled
# - Uncomment Debug Info lines when troubleshooting connection issues
# - Use -vvvv verbosity for additional connection debugging
# - CLI task can be modified to show stderr/stdout with no_log: false
---
- name: F5OS Management Tasks
  connection: httpapi
  hosts: r5900-2-api
  gather_facts: false
  collections:
    - f5networks.f5os
  any_errors_fatal: true

  vars_prompt:
    - name: f5os_api_password
      prompt: "ENTER F5OS REST API password (admin user):"
      private: yes
    - name: f5os_cli_password
      prompt: "ENTER F5OS CLI SSH password (admin user):"
      private: yes

  vars:
    ansible_user: "admin"
    ansible_httpapi_password: "{{ f5os_api_password }}"
    ansible_network_os: "f5networks.f5os.f5os"
    ansible_httpapi_use_ssl: true
    ansible_httpapi_use_proxy: false
    ansible_httpapi_validate_certs: "no"
    ansible_httpapi_port: 8888
    ansible_command_timeout: 1800
    persistent_log_messages: true
    api_base_url: "https://{{ ansible_host }}:{{ ansible_httpapi_port }}/restconf"

  tasks:
    # Method 1: Using F5OS collection modules
    - name: Get F5OS system information using F5OS Ansible module
      f5os_device_info:
        gather_subset:
          - system-info
      register: f5os_facts

    # Method 2: Using generic Ansible uri module for REST API
    - name: Get F5OS version information using REST API direct call
      uri:
        url: "{{ api_base_url }}/data/openconfig-system:system/f5-system-version:version"
        method: GET
        force_basic_auth: true
        user: "{{ ansible_user }}"
        password: "{{ ansible_httpapi_password }}"
        headers:
          Content-Type: "application/yang-data+json"
          Accept: "application/yang-data+json"
        validate_certs: "{{ ansible_httpapi_validate_certs }}"
        status_code: 200
        return_content: true
      register: f5os_version

    # Method 3: Using generic Ansible SSH for Linux system access
    - name: Gather Linux system information via SSH connection
      setup:
        gather_subset:
          - hardware
      delegate_to: r5900-2-linux
      vars:
        ansible_user: root
        ansible_connection: ssh
      register: linux_facts

    # Method 4: Using generic Ansible shell module for F5OS CLI
    - name: Get F5OS version information via CLI
      shell: |
        sshpass -p "{{ f5os_cli_password }}" ssh -t -o StrictHostKeyChecking=no admin@{{ hostvars['r5900-2-cli']['ansible_host'] }} << 'EOF'
        show sys version
        EOF
      delegate_to: localhost
      register: cli_version
      changed_when: false
      # no_log: false      # Uncomment to show CLI output for debugging
      # failed_when: false # Uncomment to prevent CLI failures for debugging

    # Method 5: Using Ansible copy and shell modules to run f5sh commands from Bash scripts
    - name: Transfer f5sh_example.sh script to F5OS device
      copy:
        content: |
          #!/bin/bash
          f5sh show sys version
        dest: "/root/f5sh_example.sh"
        mode: '0755'
      delegate_to: r5900-2-linux
      vars:
        ansible_user: root
        ansible_connection: ssh
      register: script_transfer

    - name: Execute f5sh_example.sh script on F5OS device
      shell: "/root/f5sh_example.sh"
      delegate_to: r5900-2-linux
      vars:
        ansible_user: root
        ansible_connection: ssh
      register: script_output

    # Display combined information from all five methods
    - name: Show combined information from all sources
      vars:
        system_info:
          "F5OS Version Info (via F5 maintained F5OS Ansible module collection, communicating over HTTPS REST API, using: admin account)":
            "Platform Type": "{{ f5os_facts.system_info.platform_type }}"
            "Software Version": "{{ f5os_facts.system_info.running_software.os_version }}"
            # "Debug Info": "{{ f5os_facts | default('f5os_facts not defined') }}"
          "F5OS Version Info (via built-in Ansible uri module, communicating over HTTPS REST API, using: admin account)":
            "OS Version": "{{ f5os_version.json['f5-system-version:version']['os-version'] }}"
            "Product": "{{ f5os_version.json['f5-system-version:version']['product'] }}"
            "Service Version": "{{ f5os_version.json['f5-system-version:version']['service-version'] }}"
            # "Debug Info": "{{ f5os_version | default('f5os_version not defined') }}"
          "Linux System Info (via built-in Ansible ssh module, communicating over SSH, using: root account and bash shell)":
            "Distribution": "{{ linux_facts.ansible_facts.ansible_distribution }}"
            "Kernel Version": "{{ linux_facts.ansible_facts.ansible_kernel }}"
            "Python Version": "{{ linux_facts.ansible_facts.ansible_python_version }}"
            "Total Memory": "{{ linux_facts.ansible_facts.ansible_memtotal_mb }} MB"
            # "Debug Info": "{{ linux_facts | default('linux_facts not defined') }}"
          "F5OS Version Info (via built-in Ansible ssh module, communicating over SSH, using: admin account and f5_confd_cli shell)":
            "OS Version": "{{ cli_version.stdout_lines[0].split()[-1] if cli_version is defined and cli_version.rc == 0 else 'Error getting CLI version' }}"
            "Service Version": "{{ cli_version.stdout_lines[1].split()[-1] if cli_version is defined and cli_version.rc == 0 else 'Error getting CLI version' }}"
            "Product": "{{ cli_version.stdout_lines[2].split()[-1] if cli_version is defined and cli_version.rc == 0 else 'Error getting CLI version' }}"
            # "CLI Return Code": "{{ cli_version.rc | default('rc not defined') }}"
            # "CLI STDOUT": "{{ cli_version.stdout_lines | default('stdout not defined') }}"
            # "CLI STDERR": "{{ cli_version.stderr_lines | default('stderr not defined') }}"
            # "Debug Info": "{{ cli_version | default('cli_version not defined') }}"
          "F5OS Version Info (via built-in Ansible script copy and execution modules, using: root account, bash shell, and f5sh commands)":
            "OS Version": "{{ script_output.stdout_lines[0].split()[-1] if script_output is defined and script_output.rc == 0 else 'Error executing script' }}"
            "Product": "{{ script_output.stdout_lines[2].split()[-1] if script_output is defined and script_output.rc == 0 else 'Error executing script' }}"
            "Service Version": "{{ script_output.stdout_lines[1].split()[-1] if script_output is defined and script_output.rc == 0 else 'Error executing script' }}"
            # "Script Return Code": "{{ script_output.rc | default('rc not defined') }}"
            # "Script STDOUT": "{{ script_output.stdout_lines | default('stdout not defined') }}"
            # "Script STDERR": "{{ script_output.stderr_lines | default('stderr not defined') }}"
            # "Debug Info": "{{ script_output | default('script_output not defined') }}"
      debug:
        var: system_info
(venv-2.16) 7GZL063:
```

## Terminal Demo
[![asciicast](https://asciinema.org/a/DmjfpnHKnu2o2dPZ0kFBbe6fm.svg)](https://asciinema.org/a/DmjfpnHKnu2o2dPZ0kFBbe6fm)

## Notes on Software Version Compatibility

If using the F5 maintained F5OS collection, you'll need a compatible version of Ansible:
https://github.com/F5Networks/f5-ansible-f5os

If using an alternate method that depends on compatibility with the Python3 version on the remote host, confirm the Python3 version on your F5OS appliance:

```bash
[root@appliance-1(r5900-2):Active] ~ # python3 --version
Python 3.6.8
```

...against the [ansible-core support matrix](https://docs.ansible.com/ansible/latest/reference_appendices/release_and_maintenance.html#ansible-core-support-matrix).

## Next Steps: Responsible Authentication

This tutorial focused on simple examples using default local accounts (root, admin) and local authentication.

In production, authentication is more complex, requiring:

- SSH key-based authentication instead of passwords 
- API token authentication instead basic auth
- Role-based access control with dedicated automation accounts  
- Secure credential management (e.g. Ansible Vault)
- Regular credential rotation and audit logging
- Remote authentication (LDAP, TACACS+, RADIUS)
- Upgrading from self-signed and unvalidated certificates to Certificate Authority signed and validated certificates
