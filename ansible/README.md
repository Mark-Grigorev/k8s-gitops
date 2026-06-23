Ansible - шпаргалка

## Подключение
Все команды запускаются *из корня репо* - там лежит `ansible.cfg`, который сам указывает на инвентарь(`ansible/inventory/hosts.yml`) и роли.

## Файл подключения
Данные для SSH-подколючения к серверу хранятся в:
```
ansible/inventory/host_vars/k8s_node.yml
```
Этот файл **в `.gitignore`** (содержит IP и путь к ключу), поэтому в репозитории его нет. Нужно создать его из шаблона:
```bash
cp ansible/inventory/host_vars/k8s_node_example.yml \
   ansible/inventory/host_vars/k8s_node.yml
```
И заполнить под себя, либо по ssh, либо по паролю(оба варианта одновременно не использовать).
SSH:
```yaml
ansible_host: 1.2.3.4
ansible_user: root
ansible_ssh_private_key_file: ~/.ssh/id_ed25519
```
PASS:
```
ansible_host: 1.2.3.4
ansible_user: root
ansible_password: any_pass
```
При использовании варианта с user + pass, нужно будет `sshpass`.
`sudo apt install sshpass -y`

## Ping всех хостов

```bash 
ansible all -m ping
```

Успешное подключение:
```
k8s_node | SUCCESS => {
        "ping": "pong"
}
```

## Ping конкретной группы
```bash
ansible k8s_cluster -m ping
```

## Ping конкретного хоста
```bash
ansible k8s_node -m ping
```

# Запуск плейбука 

## Полный прогон
```bash
ansible-playbook ansible/playbooks/setup.yml
```

## Dry-run(ничего не меняет, только показывает что изменится)
```bash
ansible-playbook ansible/playbooks/setup.yml --check
```

## С подробным выводом
```bash
ansible-playbook ansible/playbooks/setup.yml -v   # verbose
ansible-playbook ansible/playbooks/setup.yml -vvv # очень подробно
```

## Прогон только конкретного тега
```bash
ansible-playbook ansible/playbooks/setup.yml --tags containerd      
ansible-playbook ansible/playbooks/setup.yml --tabs kubeadm,cilium  
```

## Пропустить тег
```bash
ansible-playbook ansible/playbooks/setup.yml --skip-tags cilium
```

## Только конкретный хост из инвентаря
```bash
ansible-playbook ansible/playbooks/setup.yml --limit k8s_node
```

## Отладка                                                                                                                                                                                                     
```bash                                                                                                        
# Проверить синтаксис плейбука                                                                                 
ansible-playbook ansible/playbooks/setup.yml --syntax-check                                                    
                                                                                                               
# Список задач без выполнения                                                                                  
ansible-playbook ansible/playbooks/setup.yml --list-tasks                                                      
                                                                                                               
# Список затронутых хостов                                                                                     
ansible-playbook ansible/playbooks/setup.yml --list-hosts                                                      
                                                                                                               
# Начать с конкретной задачи (по имени)                                                                        
ansible-playbook ansible/playbooks/setup.yml --start-at-task "Install kubeadm"                                 
```                   