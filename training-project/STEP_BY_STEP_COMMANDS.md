# 07_systemd Lab - Усі команди для самостійного виконання

> **Документація з усіма командами, які потрібні для виконання лабораторної роботи Теми 7: systemd.**

---

## 🛠 Крок 0: Налаштування прямого SSH-доступу до VM

Виконуєте з **хоста**, з директорії `training-project/`:

```bash
# Знаходимо шлях до приватного ключа Vagrant
VAGRANT_KEY=$(vagrant ssh-config | grep IdentityFile | awk '{print $2}')
echo "Ключ знаходиться: $VAGRANT_KEY"

# Копіюємо ваш публічний ключ на VM (якщо у вас немає ~/.ssh/id_ed25519, генеруємо його спочатку)
ssh-copy-id -i ~/.ssh/id_ed25519.pub -o "IdentityFile=$VAGRANT_KEY" vagrant@192.168.56.10

# Перевіряємо, що все працює
ssh vagrant@192.168.56.10 "echo 'SSH-доступ налаштовано!'"
```

**Очікуваний результат:** `SSH-доступ налаштовано!` без запиту пароля.

---

## ✅ Крок 1: Перевірка стартового стану

Перевіряємо, що все з Теми 6 на місці:

```bash
# На хості - підключаємось до VM
cd /Users/antonygritsai/Desktop/lab_devops/1/devops-course/training-project
vagrant ssh

# На VM - виконуємо ці команди:
id training
ls -la /opt/training-app/
ls -la /opt/training-app/training-project/
sudo -u training /opt/training-app/training-project/venv/bin/pip show flask
```

**Очікуваний результат:** користувач `training` існує, у `/opt/training-app/training-project/` є `app.py`, `.env`, `venv/`, Flask встановлено.

---

## 🔴 Крок 2: Ручний запуск — відтворення проблеми

На **VM** запускаємо Flask у фоні:

```bash
# На VM:
sudo -u training bash -c 'cd /opt/training-app/training-project && nohup venv/bin/python app.py > /tmp/flask.log 2>&1 &'
sleep 1

# Перевіряємо, що додаток відповідає:
curl http://localhost:5000/health
```

**Очікуваний результат:** `{"status":"ok"}`

Тепер імітуємо перезавантаження:

```bash
# На VM - перезавантажуємо:
sudo reboot

# На хості - чекаємо 30 секунд та перевіряємо:
sleep 30
vagrant ssh -c "curl http://localhost:5000/health"
```

**Очікуваний результат:** помилка `curl: (7) Failed to connect to localhost port 5000` — процес не запустився після перезавантаження.

---

## 📄 Крок 3: Створення unit-файлу

На **хості**, у каталозі `training-project/`:

```bash
# Створюємо unit-файл
touch ansible/training-app.service
```

Відкрийте файл `ansible/training-app.service` у редакторі (VS Code, Vim, nano) і вставте вміст:

```ini
[Unit]
Description=Training Flask Application
After=network.target

[Service]
Type=simple
User=training
WorkingDirectory=/opt/training-app/training-project
EnvironmentFile=/opt/training-app/training-project/.env
ExecStart=/opt/training-app/training-project/venv/bin/python app.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Перевіримо вміст:

```bash
cat ansible/training-app.service
```

**Очікуваний результат:** файл містить три секції `[Unit]`, `[Service]`, `[Install]`.

---

## 📤 Крок 4: Ручне розгортання unit-файлу на VM

На **хості**, скопіюємо файл на VM через SCP:

```bash
# Знаходимо port SSH для Vagrant:
VAGRANT_PORT=2222  # Зазвичай це 2222

# Копіюємо файл:
scp -P $VAGRANT_PORT -o "IdentityFile=$(vagrant ssh-config | grep IdentityFile | awk '{print $2}')" \
  -o StrictHostKeyChecking=no ansible/training-app.service vagrant@127.0.0.1:/tmp/training-app.service
```

На **VM**:

```bash
# Переміщуємо unit-файл у системний каталог:
sudo mv /tmp/training-app.service /etc/systemd/system/training-app.service

# Перевіряємо:
ls -la /etc/systemd/system/training-app.service
```

**Очікуваний результат:** файл з властивостями `-rw-r--r-- 1 root root`.

---

## 🔄 Крок 5: Реєстрація сервісу в systemd

На **VM**:

```bash
# Повідомляємо systemd про новий unit-файл:
sudo systemctl daemon-reload

# Перевіряємо статус:
sudo systemctl status training-app
```

**Очікуваний результат:** `Loaded: loaded` і `Active: inactive (dead)`.

---

## ▶️ Крок 6: Запуск сервісу та активація автозапуску

На **VM**:

```bash
# Активуємо автозапуск:
sudo systemctl enable training-app

# Запускаємо сервіс:
sudo systemctl start training-app

# Перевіряємо статус:
sudo systemctl status training-app

# Перевіряємо, що додаток відповідає:
curl http://localhost:5000/health
```

**Очікуваний результат:** `Loaded: loaded ... enabled`, `Active: active (running)` і `{"status":"ok"}`.

---

## 🔄 Крок 7: Перевірка автозапуску після перезавантаження

На **VM**:

```bash
# Перезавантажуємо:
sudo reboot
```

На **хості** - чекаємо 30 секунд та перевіряємо:

```bash
sleep 30
vagrant ssh -c "sudo systemctl status training-app"
vagrant ssh -c "curl http://localhost:5000/health"
```

**Очікуваний результат:** `Active: active (running)` і `{"status":"ok"}` без ручного втручання.

---

## 📋 Крок 8: Перегляд логів через journalctl

На **VM**:

```bash
# Усі логи сервісу:
sudo journalctl -u training-app

# Останні 30 рядків:
sudo journalctl -u training-app -n 30

# Логи в реальному часі (зупинити Ctrl+C):
sudo journalctl -u training-app -f
```

У **другому терміналі на VM** - робимо запити:

```bash
curl -s http://localhost:5000/health
curl -s http://localhost:5000/
```

**Очікуваний результат:** у першому терміналі з'являться логи Flask.

---

## 💥 Крок 9: Симуляція збою та автоматичне відновлення

На **VM**, у першому терміналі запустіть моніторинг логів:

```bash
sudo journalctl -u training-app -f
```

У **другому терміналі на VM** - зупиняємо процес:

```bash
# Примусово вбиваємо процес:
sudo kill -9 $(sudo systemctl show -p MainPID training-app --value)
```

Спостерігайте за першим терміналом — з'явиться фраза `Scheduled restart job`.

Перевіримо, що сервіс відновився:

```bash
curl http://localhost:5000/health
```

**Очікуваний результат:** логи показують перезапуск, `{"status":"ok"}` відповідь.

---

## 🔧 Крок 10: Інтеграція з Ansible playbook

На **хості**, у каталозі `training-project/`, відкрийте файл `ansible/playbook.yml` і додайте після задачі "Встановити Python-залежності додатку":

```yaml
    - name: Встановити Python-залежності додатку
      pip:
        requirements: "{{ app_dir }}/requirements.txt"
        virtualenv: "{{ app_dir }}/venv"
        virtualenv_command: /usr/bin/python3 -m venv
      become_user: "{{ app_user }}"

    - name: Скопіювати unit-файл сервісу
      copy:
        src: training-app.service
        dest: /etc/systemd/system/training-app.service
        owner: root
        group: root
        mode: '0644'
      notify:
        - Reload systemd
        - Restart training-app

    - name: Активувати автозапуск сервісу
      systemd:
        name: training-app
        enabled: yes

    - name: Запустити сервіс
      systemd:
        name: training-app
        state: started

  handlers:
    - name: Reload systemd
      systemd:
        daemon_reload: yes

    - name: Restart training-app
      systemd:
        name: training-app
        state: restarted
```

Перевіряємо синтаксис:

```bash
cd /Users/antonygritsai/Desktop/lab_devops/1/devops-course/training-project/ansible
ansible-playbook playbook.yml --syntax-check
```

**Очікуваний результат:** `playbook: playbook.yml` без помилок.

---

## 🚀 Крок 11: Перевірка повного playbook — від нуля

На **хості**:

```bash
cd /Users/antonygritsai/Desktop/lab_devops/1/devops-course/training-project

# Знищуємо стару VM:
vagrant destroy -f

# Створюємо нову VM:
vagrant up
```

Чекаємо 2-3 хвилини. Потім запускаємо playbook:

```bash
cd ansible
ansible-playbook playbook.yml
```

**Очікуваний результат:** `PLAY RECAP ... failed=0 ... changed=11`

Перевіряємо, що все працює на новій VM:

```bash
cd ..
vagrant ssh -c "sudo systemctl status training-app"
vagrant ssh -c "curl http://localhost:5000/health"
vagrant ssh -c "sudo systemctl is-enabled training-app"
```

**Очікуваний результат:** `enabled`, `active (running)`, `{"status":"ok"}`.

---

## 💾 Крок 12: Збереження змін у Git

На **хості**:

```bash
cd /Users/antonygritsai/Desktop/lab_devops/1/devops-course/training-project

# Додаємо файли до Git:
git add ansible/training-app.service ansible/playbook.yml

# Коммітимо зміни:
git commit -m "Add systemd service unit and ansible tasks for training-app"

# Пушимо на GitHub (якщо це репозиторій з remote):
git push
```

**Очікуваний результат:** зміни на GitHub.

---

## 📸 Контрольні точки для скріншотів

Робіть скріншоти в таких місцях для звіту:

1. **Крок 2** — помилка після перезавантаження: `curl: (7) Failed to connect`
2. **Крок 5** — статус до запуску: `inactive (dead)`
3. **Крок 6** — статус після запуску: `active (running)` + `enabled`
4. **Крок 7** — автозапуск після перезавантаження: `active (running)` 
5. **Крок 8** — логи з `journalctl -u training-app -n 30`
6. **Крок 9** — логи перезапуску з фразою `Scheduled restart`
7. **Крок 11** — вихід Ansible playbook: `PLAY RECAP ... failed=0`
8. **Крок 11** — статус на новій VM: `enabled` + `active (running)`

---

## ✅ Контрольний список завершення

- [ ] Крок 0: SSH-доступ налаштовано без пароля
- [ ] Крок 1: Всі файли з Теми 6 на місці
- [ ] Крок 2: Відтворена проблема (процес не запускається після reboot)
- [ ] Крок 3: Unit-файл створено з правильним вмістом
- [ ] Крок 4: Unit-файл розгорнуто на VM
- [ ] Крок 5: systemd зареєстрував unit-файл
- [ ] Крок 6: Сервіс запущений та автозапуск активований
- [ ] Крок 7: Автозапуск працює після перезавантаження
- [ ] Крок 8: Логи видно через journalctl
- [ ] Крок 9: Сервіс автоматично перезапускається після kill -9
- [ ] Крок 10: Ansible playbook розширено правильно
- [ ] Крок 11: Повний цикл від нуля працює без помилок
- [ ] Крок 12: Зміни закомічено в Git

---

## 🎓 Додаткові корисні команди

```bash
# Переглянути unit-файл на VM:
sudo cat /etc/systemd/system/training-app.service

# Зупинити сервіс (правильно, без перезапуску):
sudo systemctl stop training-app

# Перезапустити сервіс:
sudo systemctl restart training-app

# Отримати PID процесу:
sudo systemctl show -p MainPID training-app --value

# Переглянути логи з часом:
sudo journalctl -u training-app --since "1 hour ago"

# Переглянути помилки в логах:
sudo journalctl -u training-app --no-pager | grep -i error

# Перевірити, чи сервіс автозапускається:
sudo systemctl is-enabled training-app

# Отримати детальну інформацію про сервіс:
sudo systemctl show training-app
```

---

Це все! Тепер ви можете самостійно повторити всю лабораторну роботу. 🚀
