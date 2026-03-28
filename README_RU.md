# 🌡️ Zabbix Basic CPU Temperature Monitoring

![Zabbix Version](https://img.shields.io/badge/Zabbix-7.0%2B-blue) ![Agent](https://img.shields.io/badge/Zabbix_Agent-2-green) ![License](https://img.shields.io/github/license/didimozg/zabbix-temperature-cpu-and-core)

English documentation: [README.md](./README.md).

Легковесный и оптимизированный шаблон для мониторинга температуры процессора в Zabbix.
Решение использует нативные возможности `lm-sensors` и минимум внешних скриптов (One-liners).

### ⭐️ Особенности
* **Автообнаружение (LLD):** Автоматически находит все физические ядра CPU.
* **Детальная статистика:** Температура каждого ядра + Min/Max/Avg по всему процессору.
* **Оптимизация:** Сбор метрик происходит через однострочные команды `awk` внутри конфига агента (без кучи лишних файлов).
* **Zabbix 7.4 Ready:** Исправлены проблемы с UUIDv4 для корректного импорта в новые версии Zabbix.

---

## 👥 Credits / Авторы

> **Original Idea:** [B1T0 (Tobias Scheithauer)](https://github.com/B1T0/zabbix-basic-cpu-temperature) — *Идея и базовая реализация.*
>
> **Adaptation & Fixes:** [didimozg](https://github.com/didimozg) — *Адаптация под Zabbix 7.x, исправление UUID, перевод на UserParameters.*

---

## 🛠️ Установка

### 1. Подготовка системы (Linux)
На сервере, который нужно мониторить, установите пакет для работы с сенсорами:

```bash
sudo apt update
sudo apt install lm-sensors gawk -y

```

Запустите детектор сенсоров (отвечайте **YES** на все вопросы):

```bash
sudo sensors-detect

```

### 2. Скрипт обнаружения (Discovery)

Zabbix требует JSON для обнаружения объектов. Создадим маленький скрипт-генератор.

**Файл:** `/etc/zabbix/zabbix_agent2.d/get_cores.sh`

```bash
#!/bin/bash
# JSON Generator for Zabbix CPU Core Discovery
first=1
echo "{\"data\":["
sensors | grep "^Core" | while read -r line; do
    # Парсим ID ядра (Core 0, Core 1...)
    core_id=$(echo "$line" | awk '{print $2}' | tr -d ':')
    
    if [ "$first" -eq 0 ]; then echo ","; fi
    echo -n "{\"{#CORE}\":\"$core_id\"}"
    first=0
done
echo "]}"

```

Не забудьте дать права на выполнение:

```bash
sudo chmod +x /etc/zabbix/zabbix_agent2.d/get_cores.sh

```

### 3. Конфигурация Агента

Добавим пользовательские параметры. Вся магия парсинга происходит здесь.

**Файл:** `/etc/zabbix/zabbix_agent2.d/userparameter_cputemp.conf`

```ini
# --- CPU Temperature Monitoring ---

# Discovery script
UserParameter=basicCPUTemp.discovery,/etc/zabbix/zabbix_agent2.d/get_cores.sh

# Metrics (One-liners using awk)
UserParameter=basicCPUTemp.min,LC_ALL=C sensors 2>/dev/null | awk '/^Core [0-9]+:/ { s=$0; sub(/^.*: */, "", s); if (match(s, /[-+]?[0-9]+(\.[0-9]+)?/)) { t=substr(s,RSTART,RLENGTH)+0; if (c==0||t<min) min=t; c++ } } END{ if (c>0) printf("%.1f", min); else print "-1" }'

UserParameter=basicCPUTemp.max,LC_ALL=C sensors 2>/dev/null | awk '/^Core [0-9]+:/ { s=$0; sub(/^.*: */, "", s); if (match(s, /[-+]?[0-9]+(\.[0-9]+)?/)) { t=substr(s,RSTART,RLENGTH)+0; if (c==0||t>max) max=t; c++ } } END{ if (c>0) printf("%.1f", max); else print "-1" }'

UserParameter=basicCPUTemp.avg,LC_ALL=C sensors 2>/dev/null | awk '/^Core [0-9]+:/ { s=$0; sub(/^.*: */, "", s); if (match(s, /[-+]?[0-9]+(\.[0-9]+)?/)) { sum+=substr(s,RSTART,RLENGTH)+0; c++ } } END{ if (c>0) printf("%.1f", sum/c); else print "-1" }'

# Specific core temp
UserParameter=basicCPUTemp.core[*],LC_ALL=C sensors 2>/dev/null | grep "^Core $1:" | awk '{print $3}' | sed 's/[^0-9.]//g'

```

Перезапустите агент:

```bash
sudo systemctl restart zabbix-agent2

```

### 4. Импорт шаблона в Zabbix

1. Скачайте файл `template_cpu_temp_v7.yaml` из репозитория.
2. В веб-интерфейсе перейдите: **Data collection** → **Templates** → **Import**.
3. Загрузите файл.
4. Прикрепите шаблон *Template basicCPUTemp* к нужному хосту.

---

### ❓ Troubleshooting (Проверка)

Вы можете проверить работу метрик прямо с сервера Zabbix или прокси:

```bash
# Проверка Discovery (должен вернуть JSON)
zabbix_get -s <TARGET_IP> -k basicCPUTemp.discovery

# Проверка средней температуры
zabbix_get -s <TARGET_IP> -k basicCPUTemp.avg

```


### 📦 Файл шаблона: `template_cpu_temp_v7.yaml`

```yaml
zabbix_export:
  version: '7.4'
  template_groups:
    - uuid: '7df96b18c230490a9a0a9e2307226338'
      name: Templates
  templates:
    - uuid: 'b9ff215cd8774e7780fc596eb582605b'
      template: 'Template basicCPUTemp'
      name: 'Template basicCPUTemp'
      description: |
        Template for monitoring CPU Temperature using lm-sensors.
        Uses optimized awk one-liners in agent config.
        
        Original idea: https://github.com/B1T0/zabbix-basic-cpu-temperature
        Adaptation & Fixes: https://github.com/didimozg
      groups:
        - name: Templates
      items:
        - uuid: 'fec5da56886b40c48740a057d2be9c54'
          name: 'CPU Temperature Avg'
          key: basicCPUTemp.avg
          delay: 10s
          history: 90d
          value_type: FLOAT
          units: °C
          tags:
            - tag: Application
              value: Temperature
        - uuid: '627b89d7b1954d2ea649b23b8e83576a'
          name: 'CPU Temperature Max'
          key: basicCPUTemp.max
          delay: 10s
          history: 90d
          value_type: FLOAT
          units: °C
          tags:
            - tag: Application
              value: Temperature
          triggers:
            - uuid: 'f5075d392fd7436c8b89ae7cb74ab143'
              expression: 'last(/Template basicCPUTemp/basicCPUTemp.max)>90.0'
              recovery_mode: RECOVERY_EXPRESSION
              recovery_expression: 'avg(/Template basicCPUTemp/basicCPUTemp.max,60s)<75'
              name: 'Temperature high'
              priority: AVERAGE
            - uuid: '8347731ec5fb4fa3a1201fc0983b8cde'
              expression: 'last(/Template basicCPUTemp/basicCPUTemp.max)>95'
              recovery_mode: NONE
              name: 'Temperature too high'
              priority: DISASTER
              manual_close: 'YES'
        - uuid: '3b42e6a5c4c7418386961bb570d63b63'
          name: 'CPU Temperature Min'
          key: basicCPUTemp.min
          delay: 10s
          history: 90d
          value_type: FLOAT
          units: °C
          tags:
            - tag: Application
              value: Temperature
      discovery_rules:
        - uuid: 'aab958191be14454bbdecf00a6416b4d'
          name: 'CPU Cores Discovery'
          key: basicCPUTemp.discovery
          delay: 1h
          item_prototypes:
            - uuid: 'ab914fed6b4f477781dd24f4be9cf1b8'
              name: 'CPU Core {#CORE} Temp'
              key: 'basicCPUTemp.core[{#CORE}]'
              value_type: FLOAT
              units: °C
          trigger_prototypes:
            - uuid: '79a29e4682024220b224151475756303'
              expression: 'last(/Template basicCPUTemp/basicCPUTemp.core[{#CORE}])>85'
              recovery_mode: RECOVERY_EXPRESSION
              recovery_expression: 'avg(/Template basicCPUTemp/basicCPUTemp.core[{#CORE}],3m)<75'
              name: 'High temperature on Core {#CORE} (>85°C)'
              priority: HIGH
            - uuid: '81c6260196884849925e016147101867'
              expression: 'last(/Template basicCPUTemp/basicCPUTemp.core[{#CORE}])>95'
              name: 'Critical temperature on Core {#CORE} (>95°C)'
              priority: DISASTER
  graphs:
    - uuid: 'ff78a7a1ff9740198ff9c40fc3fce6ea'
      name: 'CPU Temperature'
      graph_items:
        - drawtype: BOLD_LINE
          color: 1A7C11
          calc_fnc: ALL
          item:
            host: 'Template basicCPUTemp'
            key: basicCPUTemp.min
        - sortorder: '1'
          drawtype: BOLD_LINE
          color: 2774A4
          calc_fnc: ALL
          item:
            host: 'Template basicCPUTemp'
            key: basicCPUTemp.avg
        - sortorder: '2'
          drawtype: BOLD_LINE
          color: F63100
          calc_fnc: ALL
          item:
            host: 'Template basicCPUTemp'
            key: basicCPUTemp.max

```
