# postgresql-HA-ecy
Postgresql + Etcd + Patroni == HA Postgresql 

--------------------------------------------------
--------------------------------------------------
Ansible ile Postgresql sunucusunu birkaç dosya ve komutla kolayca kurun hemde HA !

Easily install Postgresql server with Ansible with a few files and commands and HA !
--------------------------------------------------
--------------------------------------------------
ansible-playbook -i inventory postgresql-ha-ecy.yml

ssh root@192.168.31.100
patronictl -c /etc/patroni/patroni.yml list

ok :)
