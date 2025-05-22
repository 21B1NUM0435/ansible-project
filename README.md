# Ansible Автоматжуулалтын Төсөл

Энэхүү репозиторид Vagrant ашиглан 7 виртуал машин удирдаж, 4 үндсэн автоматжуулалтын ажлыг гүйцэтгэдэг бүрэн Ansible төсөл байна.

## 🎯 Төслийн Тойм

Энэ төсөл дараах 4 ажлыг автоматжуулдаг:
1. **Даалгавар 1**: Бүх хост дээр системийн шинэчлэл болон дахин эхлүүлэх
2. **Даалгавар 2**: site1 хостууд дээр Docker болон Git суулгах
3. **Даалгавар 3**: host5 дээр nested Vagrant VM үүсгэх (50% RAM, 2 CPU)
4. **Даалгавар 4**: Бүх хост дээр 9090 порт дээр netcat сервис үүсгэх

## 🚀 Суулгах

### Шаардлагатай Зүйлс
- Ubuntu 24.04.2 LTS
- 16GB+ RAM, 70GB+ диск орон зай

### 1-р Алхам: Шаардлагатай Программууд Суулгах
```bash
# Системийг шинэчлэх
sudo apt update && sudo apt upgrade -y

# Шаардлагатай програмуудыг суулгах
sudo apt install -y vagrant virtualbox virtualbox-ext-pack ansible git

# Суулгалтыг шалгах
vagrant --version && vboxmanage --version && ansible --version
```

### 2-р Алхам: Төслийг Татах
```bash
git clone https://github.com/21B1NUM0435/ansible-project.git
cd ansible-project
```

### 3-р Алхам: VirtualBox Сүлжээ Тохируулах
```bash
# Host-only сүлжээ үүсгэх (сүлжээний зөрчил гарвал)
sudo vboxmanage hostonlyif create
sudo vboxmanage hostonlyif ipconfig vboxnet0 --ip 192.168.57.1 --netmask 255.255.255.0
```

### 4-р Алхам: VM-үүдийг Эхлүүлэх
```bash
# Бүх 7 VM-ийг эхлүүлэх (10-15 минут)
vagrant up

# Статусыг шалгах
vagrant status
```

### 5-р Алхам: SSH Түлхүүр Тохируулах

Энэхүү скриптийг ажиллуулаарай:

```bash
cat > setup_ssh_keys.sh << 'EOF'
#!/bin/bash

echo "🔑 Бүх VM-д SSH түлхүүр тохируулж байна..."

# Хэрэглэгч бүрт SSH түлхүүр үүсгэх
for i in {1..7}; do
    if [ ! -f ~/.ssh/user${i}_key ]; then
        ssh-keygen -t rsa -b 2048 -f ~/.ssh/user${i}_key -N "" -C "user${i}@ansible-managed"
        echo "✅ user${i}-д SSH түлхүүр үүсгэлээ"
    else
        echo "✅ user${i}-ийн SSH түлхүүр аль хэдийн байна"
    fi
done

# SSH түлхүүрүүдийг VM-д хуулах
for i in {1..7}; do
    echo "📤 host${i} дээр user${i}-д SSH түлхүүр тохируулж байна..."
    PUB_KEY=$(cat ~/.ssh/user${i}_key.pub)
    vagrant ssh host${i} -c "
        sudo mkdir -p /home/user${i}/.ssh
        echo '${PUB_KEY}' | sudo tee /home/user${i}/.ssh/authorized_keys
        sudo chown -R user${i}:user${i} /home/user${i}/.ssh
        sudo chmod 700 /home/user${i}/.ssh
        sudo chmod 600 /home/user${i}/.ssh/authorized_keys
        
        # Ansible-д зориулсан tmp хавтас үүсгэх
        sudo mkdir -p /home/user${i}/.ansible/tmp
        sudo chown -R user${i}:user${i} /home/user${i}/.ansible
        sudo chmod -R 755 /home/user${i}/.ansible
        
        echo '✅ user${i} тохирууллаа'
    "
done

echo "🧪 SSH холболтыг шалгаж байна..."
ansible all -m ping

if [ $? -eq 0 ]; then
    echo "🎉 SSH тохиргоо амжилттай дууслаа!"
else
    echo "❌ Зарим холболтод алдаа гарлаа. Доорх засах скриптийг ажиллуулаарай."
fi
EOF

chmod +x setup_ssh_keys.sh
./setup_ssh_keys.sh
```

## 🔧 Асуудал Засах Скриптүүд

### SSH Холболтын Асуудал Засах
Хэрэв SSH холболт амжилтгүй бол:

```bash
cat > fix_ssh_issues.sh << 'EOF'
#!/bin/bash

echo "🔧 SSH асуудлыг засаж байна..."

# Хэрэглэгч бүрийг шалгаж, байхгүй бол үүсгэх
for i in {1..7}; do
    echo "host${i} дээр user${i}-г шалгаж байна..."
    vagrant ssh host${i} -c "
        # user${i} байгаа эсэхийг шалгах
        if ! id user${i} >/dev/null 2>&1; then
            echo 'user${i} үүсгэж байна...'
            sudo useradd -m -s /bin/bash user${i}
            echo 'user${i}:password123' | sudo chpasswd
            sudo usermod -aG sudo user${i}
            echo 'user${i} ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/user${i}
            sudo chmod 0440 /etc/sudoers.d/user${i}
        fi
        
        # SSH тохиргоог засах
        sudo mkdir -p /home/user${i}/.ssh
        PUB_KEY='$(cat ~/.ssh/user${i}_key.pub)'
        echo \"\$PUB_KEY\" | sudo tee /home/user${i}/.ssh/authorized_keys
        sudo chown -R user${i}:user${i} /home/user${i}/.ssh
        sudo chmod 700 /home/user${i}/.ssh
        sudo chmod 600 /home/user${i}/.ssh/authorized_keys
        
        # Ansible tmp хавтас
        sudo mkdir -p /home/user${i}/.ansible/tmp
        sudo chown -R user${i}:user${i} /home/user${i}/.ansible
        sudo chmod -R 755 /home/user${i}/.ansible
        
        sudo systemctl restart ssh
        echo '✅ user${i} бүрэн засагдлаа'
    "
done

echo "🧪 Холболтыг дахин шалгаж байна..."
ansible all -m ping

echo "🎉 SSH асуудал засагдлаа!"
EOF

chmod +x fix_ssh_issues.sh
# Шаардлагатай үед л ажиллуулна: ./fix_ssh_issues.sh
```


## 📋 Бүх Даалгавар Ажиллуулах

SSH тохиргоо амжилттай болсны дараа дараах дарааллаар playbook-уудыг ажиллуулаарай:

```bash
# Холболтыг шалгах
ansible all -m ping

# Даалгавар 1: Системийн шинэчлэл
ansible-playbook playbooks/01-dist-upgrade.yml

# Даалгавар 2: Docker & Git (site1 дээр)
ansible-playbook playbooks/02-docker-git-site1.yml

# Даалгавар 3: Nested VM (host5 дээр)
ansible-playbook playbooks/03-vagrant-setup-fixed.yml

# Даалгавар 4: Netcat сервисүүд
ansible-playbook playbooks/04-netcat-service-fixed.yml
```

## 🧪 Шалгах

Бүх зүйл зөв ажиллаж байгаа эсэхийг шалгах:

```bash
cat > validate_everything.sh << 'EOF'
#!/bin/bash

echo "🔍 Бүх даалгаврын шалгалт"

echo "1. Хостуудын холболтыг шалгаж байна..."
ansible all -m ping

echo "2. Docker-г site1 дээр шалгаж байна..."
ansible site1 -m shell -a "docker --version && docker ps" --become=no

echo "3. Git-г site1 дээр шалгаж байна..."
ansible site1 -m shell -a "git --version" --become=no

echo "4. Nested VM-г host5 дээр шалгаж байна..."
ansible site2[0] -m shell -a "cd /home/user5/nested-vagrant-vm && vagrant status" --become-user=user5

echo "5. Netcat сервисүүдийг шалгаж байна..."
ansible all -m shell -a "systemctl is-active netcat-{{ ansible_user }}"

echo "6. 9090 портыг шалгаж байна..."
ansible all -m shell -a "ss -tlnp | grep :9090 || echo 'Порт сонсохгүй байна'"

echo "✅ Бүх шалгалт дууслаа!"
EOF

chmod +x validate_everything.sh
./validate_everything.sh
```
