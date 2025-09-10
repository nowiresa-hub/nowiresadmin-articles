🔧 Кейс: Виртуализация ERP на сервере HP + VLAN + удалённый доступ через iLO

Работал с клиентом — нужно было развернуть ERP на старом сервере HP, обеспечить изоляцию трафика, и всё это удалённо из Москвы.
У сервера был только iLO-доступ, причём через ActiveX (в 2025 году, да 😄).

📌 Что сделал:

🖥 Установка гипервизора
Сервер — под Debian 11, устанавливаю KVM + всё необходимое:
apt install qemu-kvm libvirt-daemon-system virtinst bridge-utils

Проверяю, видит ли система аппаратную виртуализацию:
egrep -c '(vmx|svm)' /proc/cpuinfo

🧱 Настройка сети и VLAN
Сервер подключён к MikroTik. Нужно было завести ERP в отдельную VLAN (19):

🔹 На MikroTik:
/interface vlan
add name=vlan19 interface=ether2 vlan-id=19

/interface bridge
add name=bridge-vm-19

/interface bridge port
add interface=vlan19 bridge=bridge-vm-19

🔹 На Debian создаю подинтерфейс:
cat >> /etc/network/interfaces << EOF
auto eno1.19
iface eno1.19 inet static
  address 10.96.196.10
  netmask 255.255.255.0
EOF
ip link add link eno1 name eno1.19 type vlan id 19
ifup eno1.19

🔹 Создаю bridge для виртуалок:
cat >> /etc/network/interfaces << EOF
auto br-vm-19
iface br-vm-19 inet manual
  bridge_ports eno1.19
  bridge_stp off
  bridge_fd 0
EOF
ifup br-vm-19

🧰 Развёртывание ERP
ISO-образ загружен на сервер. Разворачиваю виртуалку через virt-install:
virt-install \
  --name erp-vm \
  --ram 4096 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/erp.qcow2,size=20 \
  --os-variant debian11 \
  --network bridge=br-vm-19,model=virtio \
  --cdrom /tmp/erp-install.iso \
  --graphics none
После установки виртуалка получает IP из подсети 10.96.196.0/24.

🔐 Доступ через iLO
Так как на этом HP-шнике только iLO 3 и работает он адекватно только через Internet Explorer с ActiveX, пришлось вспомнить 2012 год.

👉 Через него загрузил образ ISO и следил за установкой.
Да, это всё еще работает. Но с костылями.

📦 ERP теперь работает в изолированной виртуальной среде. Трафик между сервисами — разделён, есть возможность администрирования даже при падении сети — через iLO.

🧌 Все VLAN и IP-сети в кейсе — вымышленные

🧠 Использованные технологии:
#kvm #virtinstall #linux #vlan #mikrotik #bridge #ilo #erp #virtualization 

add case: ERP на HP + VLAN + iLO
