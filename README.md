# postgresql-HA-ecy
Postgresql + Etcd + Patroni == HA Postgresql 

postgresql-ha-cluster/
├── inventory
├── playbook.yml
└── templates/
    ├── etcd.conf.j2
    ├── patroni.yml.j2
    └── patroni.service.j2

ansible-playbook -i inventory postgresql-ha-ecy.yml

ssh root@192.168.31.100
patronictl -c /etc/patroni/patroni.yml list

ok :)
