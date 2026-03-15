# Ansible
![Ansible[200]](./images/welcome.png)
## Quick start
- Устанавливаем ansible на машину (Ansible Master)
```
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```
- Создаём инветарный файл в котором описываются хосты (сервера), которыми будет управлять ansible
```
[web_servers]
webserv ansible_host=155.0.222.227  ansible_user=user  ansible_private_key=/home/user/.ssh/webserv
```
