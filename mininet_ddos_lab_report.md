# Лабораторный отчёт
**Тема:** Исследование поведения сетевых и прикладных протоколов в Mininet при контролируемой нагрузке и оценка защитных мер
**Студент:** _<ФИО>_
**Группа:** _<Группа>_
**Преподаватель:** _<ФИО>_
**Дата:** _<дата>_

---

# 1. Аннотация
В работе моделируется изолированная лабораторная сеть в Mininet, проводятся контролируемые стресс‑тесты TCP/UDP/HTTP и исследуется влияние защитных механизмов (фильтрация, rate‑limiting, firewall, IDS). Все эксперименты выполнялись в изолированной виртуальной машине; **в работу не включены инструкции для проведения вредоносных действий против реальных сетей**. Цель — показать влияние нагрузки на сервис, методы защиты и сравнить метрики «до» и «после» включения защит.

---

# 2. Оборудование и ПО
- Виртуальная машина Ubuntu (изолированная, без внешнего воздействия на реальную сеть)
- Mininet (рекомендуемая версия: 2.3.x)
- Python 3
- iperf3, tcpdump, tshark, apache2-utils (ab), top/ss/vmstat
- (опционально) Suricata или Snort для IDS в режиме мониторинга

---

# 3. Топология сети
```
h1 (Client) --- s1 --- h2 (Server)
                       \
                        \- h3 (Monitor / IDS)
```

Файл топологии: `~/mininet_lab_topo.py` (запуск `sudo python3 ~/mininet_lab_topo.py`)  
**Скриншот запуска сети:**  
`![](img/start_network.png)`

---

# 4. Подготовка директорий и логов
```bash
mkdir -p ~/experiment_results/{img,logs}
```
**Скриншот содержимого папки:**  
`![](img/results_folder.png)`

---

# 5. Базовые проверки (baseline)

### 5.1 Проверка связности
В Mininet CLI:
```
mininet> pingall
```
**Ожидаем:** 0% packet loss.  
`![](img/pingall.png)`

### 5.2 Запуск HTTP‑сервера
На h2:
```
mininet> h2 python3 -m http.server 8000 --bind 10.0.0.2 &
```
Проверка с h1:
```
mininet> h1 curl -sS http://10.0.0.2:8000/ -o /home/experiment_results/logs/http_baseline.txt
```
**Скриншот curl:**  
`![](img/http_baseline.png)`

### 5.3 Базовый TCP‑тест (iperf3)
На h2 (сервер):
```
mininet> h2 iperf3 -s -D
```
На h1 (клиент):
```
mininet> h1 iperf3 -c 10.0.0.2 -b 5M -t 10 > /home/experiment_results/logs/iperf3_baseline_5M.txt
```
**Скриншот iperf3 baseline:**  
`![](img/iperf_5M.png)`

### 5.4 Базовый UDP‑тест (контролируемый)
```
mininet> h1 iperf3 -c 10.0.0.2 -u -b 5M -t 10 > /home/experiment_results/logs/iperf3_udp_baseline.txt
```
**Скриншот UDP baseline:**  
`![](img/iperf_udp.png)`

---

# 6. Контролируемая нагрузка (имитация аномальной активности)
> Все стресс‑тесты выполняются в изолированном окружении.

### 6.1 TCP — повышенная нагрузка
```
mininet> h1 iperf3 -c 10.0.0.2 -b 20M -t 30 > /home/experiment_results/logs/iperf3_20M.txt
mininet> h1 iperf3 -c 10.0.0.2 -b 50M -t 30 > /home/experiment_results/logs/iperf3_50M.txt
```
**Скриншот:**  
`![](img/iperf_20M.png)`

### 6.2 UDP — повышенная нагрузка (контролируемая)
```
mininet> h1 iperf3 -c 10.0.0.2 -u -b 15M -t 30 > /home/experiment_results/logs/iperf3_udp_15M.txt
```

### 6.3 HTTP‑нагрузка (ApacheBench)
```
mininet> h1 ab -n 2000 -c 100 http://10.0.0.2:8000/ > /home/experiment_results/logs/ab_2000x100.txt
```
**Скриншот ab:**  
`![](img/ab_output.png)`

### 6.4 Эмуляция ухудшения канала (tc / TCLink)
```
mininet> h2 tc qdisc add dev h2-eth0 root netem delay 100ms loss 2%
```
или в Python‑топологии:
```python
net.addLink(h1, s1, cls=TCLink, bw=10, delay='100ms', loss=2)
```

### 6.5 Сбор данных во время нагрузок
```
mininet> h3 tcpdump -i h3-eth0 -w /home/experiment_results/trace_during.pcap -c 20000 &
mininet> h2 ss -ant > /home/experiment_results/logs/ss_during.txt
```
**Скриншот tcpdump head:**  
`![](img/tcpdump_head.png)`

---

# 7. Включение защитных мер
### 7.1 Системные параметры (tcp)
```bash
sudo sysctl -w net.ipv4.tcp_syncookies=1
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=2048
sudo sysctl -w net.ipv4.tcp_fin_timeout=30
```

### 7.2 Базовая фильтрация и rate limiting (iptables)
```bash
sudo iptables -N RATE_LIMIT_NEW
sudo iptables -A INPUT -p tcp --syn -m conntrack --ctstate NEW -j RATE_LIMIT_NEW
sudo iptables -A RATE_LIMIT_NEW -m limit --limit 20/min --limit-burst 40 -j ACCEPT
sudo iptables -A RATE_LIMIT_NEW -j DROP
sudo iptables -A INPUT -p tcp --syn -m connlimit --connlimit-above 50 -j DROP
```

### 7.3 Ограничение запросов на веб‑сервере (nginx пример)
```
limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;
location / {
    limit_req zone=one burst=20 nodelay;
}
```

### 7.4 IDS/мониторинг (Suricata)
```
sudo suricata -c /etc/suricata/suricata.yaml -i h3-eth0
```
**Скриншот включённых защит/логов:**  
`![](img/defense_config.png)`

---

# 8. Повторное тестирование (после включения защит)
**Скриншоты после защит:**  
`![](img/iperf_after.png)`  
`![](img/ab_after.png)`

---

# 9. Сбор метрик
Пример CSV: `summary.csv`
```
test_id,scenario,metric,timestamp,value,unit,notes
1,baseline,throughput,2025-10-25T10:12:00,4.9,Mbps,"iperf3 receiver baseline"
2,stress_20M_before,throughput,2025-10-25T10:25:00,18.6,Mbps,"iperf3 under load before defenses"
3,stress_20M_after,throughput,2025-10-25T11:10:00,15.2,Mbps,"iperf3 under load after defenses"
```

---

# 10. Анализ результатов
- Сравнение `throughput` и `http_latency` «до» и «после»  
- Анализ CPU/Memory и состояния соединений  
- Проверка логов IDS (Suricata) на события

**Пример визуализации:**  
`![](img/throughput_compare.png)`  
`![](img/latency_compare.png)`

---

# 11. Выводы
- Контролируемые стресс‑тесты показывают рост latency и CPU при нагрузке.  
- После включения защит снижается число новых соединений и spike‑эффекты, возможно небольшое увеличение времени отклика.  
- Рекомендуется комбинировать меры на уровне приложения, сетевого уровня и мониторинга.

---

# 12. Этическая заметка
Все тесты проводились в изолированной среде. Запуск нагрузочных тестов против чужих ресурсов запрещён. Все инструкции безопасны для лабораторной работы.

