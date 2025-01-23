[![asciicast](https://asciinema.org/a/O8OcDyXpuJXhKkdS23IEXF1fO.svg)](https://asciinema.org/a/O8OcDyXpuJXhKkdS23IEXF1fO)

```bash
ansible-playbook -i hosts.yml playbook.yml --ask-pass
```

r5900-2 remote f5os device:

```bash
(venv-2.16) [Ubuntu 20.04.6 LTS] ssh root@r5900-2

[root@appliance-1(r5900-2):Active] ~ # python3 --version
Python 3.6.8
[root@appliance-1(r5900-2):Active] ~ # su admin
Welcome to the Management CLI
r5900-2# show sys version
system version os-version 1.8.0-16036
system version service-version 1.8.0-16036
system version product F5OS-A
```

ansible controller:

```bash
(venv-2.16) [Ubuntu 20.04.6 LTS] python --version

Python 3.11.11

(venv-2.16) [Ubuntu 20.04.6 LTS] ansible --version

ansible [core 2.16.6]
  config file = /home/tonym/projects/ansible_f5os_tmos_linux/ansible.cfg
  configured module search path = ['/home/tonym/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /home/tonym/ansible-f5/ansible_2.16/venv-2.16/lib/python3.11/site-packages/ansible
  ansible collection location = /home/tonym/.ansible/collections:/usr/share/ansible/collections
  executable location = /home/tonym/ansible-f5/ansible_2.16/venv-2.16/bin/ansible
  python version = 3.11.11 (main, Dec  4 2024, 08:55:08) [GCC 9.4.0] (/home/tonym/ansible-f5/ansible_2.16/venv-2.16/bin/python3.11)
  jinja version = 3.1.5
  libyaml = True  
```

ansible core support matrix:
https://docs.ansible.com/ansible/latest/reference_appendices/release_and_maintenance.html#ansible-core-support-matrix
