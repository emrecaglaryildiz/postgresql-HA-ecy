# postgresql-HA-ecy
Postgresql + Etcd + Patroni == HA Postgresql 

ansible-playbook -i inventory postgresql-ha-ecy.yml

ssh root@192.168.31.100
patronictl -c /etc/patroni/patroni.yml list

ok :)
