# Лабораторная работа: Исследование поведения сетевых и прикладных протоколов в Mininet

## 1. Цель работы
Создать изолированную сеть в Mininet на Ubuntu, изучить поведение TCP, UDP и HTTP при разных нагрузках, собрать метрики и подготовить визуальные результаты.

---

## 2. Подготовка среды

### 2.1 Установка зависимостей
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3-pip python3-setuptools git build-essential                     mininet openvswitch-switch iperf3 tcpdump apache2-utils                     net-tools iproute2 vim
```

Проверяем версию Mininet:
```bash
sudo mn --version
```

✅ *Ожидается вывод вроде:*  
`2.3.0d6`

---

## 3. Создание структуры эксперимента

Создаём директорию для результатов:
```bash
mkdir -p ~/experiment_results/{img,logs}
```

---

## 4. Создание топологии

Открываем файл:
```bash
nano ~/mininet_safe_topology.py
```

Вставляем код:

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

"""
Безопасная минимальная топология Mininet:
h1 (Client) —— s1 —— h2 (Server)
                │
                └—— h3 (Monitor)
"""

from mininet.net import Mininet
from mininet.node import OVSSwitch
from mininet.link import TCLink
from mininet.cli import CLI

def run_topology():
    print("*** Создаётся сеть без контроллера...")
    net = Mininet(controller=None, switch=OVSSwitch, link=TCLink)

    print("*** Добавление узлов")
    h1 = net.addHost('h1', ip='10.0.0.1/24')
    h2 = net.addHost('h2', ip='10.0.0.2/24')
    h3 = net.addHost('h3', ip='10.0.0.3/24')
    
    # Коммутатор в режиме standalone — форвардит пакеты без контроллера
    s1 = net.addSwitch('s1', failMode='standalone')

    print("*** Создание линков")
    net.addLink(h1, s1)
    net.addLink(h2, s1)
    net.addLink(h3, s1)

    print("*** Запуск сети")
    net.start()

    print("*** Поднимаем интерфейсы всех хостов")
    for h in (h1, h2, h3):
        for intf in h.intfList():
            h.cmd(f'ifconfig {intf} up')

    print("*** Запуск простого HTTP-сервера на h2 с чистым текстом")
    h2.cmd("""nohup python3 -c '
import socket
s = socket.socket()
s.bind(("0.0.0.0", 8000))
s.listen(1)
while True:
    c, addr = s.accept()
    data = c.recv(1024)
    c.sendall(b"HTTP/1.1 200 OK\\nContent-Type: text/plain\\n\\nHello from victim\\n")
    c.close()
' >/tmp/http_server.log 2>&1 &""")

    print("*** Сеть готова: h1 (клиент), h2 (сервер), h3 (монитор)")
    print("*** Для выхода из CLI набери: exit")
    CLI(net)

    print("*** Остановка сети")
    net.stop()

if __name__ == '__main__':
    run_topology()

```

Сохраняем (`Ctrl+O`, `Enter`, `Ctrl+X`), делаем файл исполняемым:
```bash
chmod +x ~/mininet_safe_topology.py
```

---

## 5. Запуск Mininet

```bash
sudo python3 ~/mininet_safe_topology.py
```

📸 *Вставь сюда скриншот запуска сети:*  
`![](img/start_network.png)`

---

## 6. Проверка связности

```text
mininet> pingall
```

📸 *Скриншот успешного pingall (0% packet loss)*  
`![](img/pingall.png)`

---

## 7. Проверка HTTP

```text
mininet> h1 curl -sS http://10.0.0.2:8000/
```

✅ Ожидаемый ответ:
```
Hello from victim
```

📸 *Скриншот вывода curl:*  
`![](img/http_baseline.png)`

---

## 8. TCP-тест (5 Мбит/с, 10 с)

```text
mininet> h2 iperf3 -s -D
mininet> h1 iperf3 -c 10.0.0.2 -b 5M -t 10 > /home/$USER/experiment_results/logs/iperf3_5M.txt
```

📸 *Скриншот iperf3 результата:*  
`![](img/iperf_5M.png)`

---

## 9. TCP-тест (10 Мбит/с, 10 с)

```text
mininet> h1 iperf3 -c 10.0.0.2 -b 10M -t 10 > /home/$USER/experiment_results/logs/iperf3_10M.txt
```

📸 *Скриншот iperf3 10M теста:*  
`![](img/iperf_10M.png)`

---

## 10. UDP-тест

```text
mininet> h1 iperf3 -c 10.0.0.2 -u -b 5M -t 10 > /home/$USER/experiment_results/logs/iperf3_udp.txt
```

📸 *Скриншот результата UDP:*  
`![](img/iperf_udp.png)`

---

## 11. HTTP-нагрузка (ApacheBench)

```text
mininet> h1 ab -n 100 -c 5 http://10.0.0.2:8000/ > /home/$USER/experiment_results/logs/ab_100x5.txt
```

📸 *Скриншот вывода ab:*  
`![](img/ab_output.png)`

---

## 12. Захват пакетов

```text
mininet> h3 tcpdump -r /home/$USER/experiment_results/trace.pcap -n -tt | head -n 5
```

📸 *Скриншот tcpdump:*  
`![](img/tcpdump_head.png)`

---

## 13. Завершение работы

```text
mininet> exit
```
Очистка сети:
```bash
sudo mn -c
```

---

## 14. Проверка результатов

```bash
ls ~/experiment_results/
```

📸 *Скриншот содержимого папки experiment_results:*  
`![](img/results_folder.png)`

---

## 15. Вывод

- Все тесты проведены в изолированной среде Mininet.  
- Потери пакетов минимальны, HTTP и TCP соединения стабильны.  
- Получены данные для анализа пропускной способности и поведения протоколов.

---

## 16. Анализ логов (Python)

Создайте файл `analyze_results.py`:

```python
import matplotlib.pyplot as plt

# Пример чтения логов iperf3
with open('~/experiment_results/logs/iperf3_5M.txt') as f:
    data = f.read()

# Простейший парсер пропускной способности
for line in data.splitlines():
    if "receiver" in line:
        print(line)

# Визуализация — пример
speeds = [4.9, 9.6, 4.9]
labels = ['TCP 5M', 'TCP 10M', 'UDP 5M']
plt.bar(labels, speeds)
plt.ylabel('Mbit/s')
plt.title('Результаты iperf3')
plt.savefig('~/experiment_results/img/iperf_summary.png')
plt.show()
```

Запуск:
```bash
python3 analyze_results.py
```

📸 *Скриншот графика из analyze_results.py:*  
`![](img/iperf_summary.png)`

---

*Файл создан автоматически — шаблон лабораторного отчёта Mininet (2025)*
