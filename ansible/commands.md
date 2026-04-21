Ansible - шпаргалка

## Ping всех хостов

```bash 
ansible all -m ping
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