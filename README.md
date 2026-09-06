# FlClashX на Orange Pi Zero 3W (Debian 13)

> Замена mihomo на FlClashX — GUI-клиент Clash Meta (ядро Clash Meta).
> Проверено: **Debian 13 (trixie), MATE desktop, ядро 6.6.98-sun60iw2, Allwinner A733**.

## Содержание

1. [Установка](#1-установка)
2. [Перенос профиля и подписки с другой машины](#2-перенос-профиля-и-подписки-с-другой-машины)
3. [Автозапуск (MATE)](#3-автозапуск-mate)
4. [Проверка работы](#4-проверка-работы)
5. [Проблема: прокси не поднимается (setuid FlClashCore)](#5-проблема-прокси-не-поднимается-setuid-flclashcore)
6. [Проблема: чёрное окно GUI (Flutter на PowerVR)](#6-проблема-чёрное-окно-gui-flutter-на-powervr)
7. [Полезные команды и пути](#7-полезные-команды-и-пути)
8. [Уроки](#8-уроки)

---

## 1. Установка

Нужен deb-пакет **arm64** (проверенная версия **0.3.2+2026012301**).
Скачайте его со страницы релизов проекта или возьмите с машины, где FlClashX уже установлен.

```bash
# 1. Зависимости (указаны в Depends пакета)
sudo apt install -y libayatana-appindicator3-dev libkeybinder-3.0-dev

# 2. Установка самого пакета
sudo dpkg -i FlClashX-linux-arm64.deb
```

Что ставится:
- `/usr/bin/FlClashX` — GUI (Flutter)
- `/usr/share/FlClashX/FlClashCore` — ядро Clash Meta (запускается GUI)
- данные приложения: `~/.local/share/com.follow.clashx/`

> ⚠️ **После установки сразу проверьте права на ядро** (см. [Проблему №5](#5-проблема-прокси-не-поднимается-setuid-flclashcore)):
> ```bash
> ls -la /usr/share/FlClashX/FlClashCore
> # должно быть: -rwsr-sr-x  (setuid!)
> # если -rwxr-xr-x — лечите, иначе прокси не поднимется
> ```

---

## 2. Перенос профиля и подписки с другой машины

Профиль FlClashX — это папка данных приложения. Самый надёжный способ перенести
подписку — скопировать её целиком с машины, где она уже настроена, а не вводить вручную.

```bash
# НА машине-доноре: собрать данные в архив
cd ~/.local/share
tar czf flclashx-data.tar.gz com.follow.clashx/

# скопировать на целевую машину (укажите свой user@host)
scp flclashx-data.tar.gz user@host:/tmp/

# НА целевой машине: распаковать
cd ~/.local/share
tar xzf /tmp/flclashx-data.tar.gz
```

Что внутри `~/.local/share/com.follow.clashx/`:
| Файл | Назначение |
|---|---|
| `shared_preferences.json` | настройки: `flutter.config` (appSetting, profiles, currentProfileId), `flutter.app_persistent_hwid` |
| `profiles/*.yaml` | конфиги профилей (подписка уже скачана, серверы внутри) |
| `GeoIP.dat`, `GeoSite.dat`, `ASN.mmdb`, `geoip.metadb` | гео-базы для ядра |
| `logs/` | логи FlClashX |
| `FlClashX.lock` | lock-файл (пустой) |

### ⚠️ HWID: заменить на новый!

В `shared_preferences.json` есть `flutter.app_persistent_hwid` — уникальный ID машины.
Если скопировать данные с другой машины как есть, **две машины будут с одним HWID**
(конфликт: подписка/лимиты, «устройство уже используется»). Сгенерируйте новый UUID:

```bash
python3 - <<'EOF'
import json, uuid

path = "/home/USER/.local/share/com.follow.clashx/shared_preferences.json"
d = json.load(open(path))
d["flutter.app_persistent_hwid"] = str(uuid.uuid4())
json.dump(d, open(path, "w"), indent=2)
print("HWID обновлён:", d["flutter.app_persistent_hwid"])
EOF
```

### Ключевые настройки в `flutter.config`

```json
"appSetting": {
  "autoRun": true,          // прокси сам подключается при старте GUI
  "autoLaunch": false,      // НЕ автозапуск через systemd/сессию
  "silentLaunch": false,
  "disclaimerAccepted": true
},
"currentProfileId": 1234567890,   // активный профиль (у вас будет свой ID)
"profiles": [...]                 // список профилей с URL подписки
```

> Если подписка истекла или нужна новая — URL подписки можно добавить через GUI:
> Profiles → «+» → вставить URL. Но при переносе готового профиля URL не обязателен —
> YAML уже содержит скачанные серверы.

---

## 3. Автозапуск (MATE)

**НЕ systemd-сервис!** FlClashX — GUI-приложение (Flutter), ему нужна X-сессия
(DISPLAY, XAUTHORITY). Если запустить его как systemd-сервис при загрузке — сервис
останется `inactive (dead)`, потому что X ещё не готов (проверено на практике).

Правильный способ — XDG autostart (папка `~/.config/autostart/`), срабатывает при
входе в MATE. Файл `flclashx.desktop` (готовый — в этом репозитории):

```ini
[Desktop Entry]
Type=Application
Name=FlClashX
Comment=Proxy GUI (Mesa software GL fix)
Exec=env LD_LIBRARY_PATH=/usr/lib/aarch64-linux-gnu LIBGL_ALWAYS_SOFTWARE=1 /usr/bin/FlClashX
X-GNOME-Autostart-enabled=true
Terminal=false
```

Установка:
```bash
cp flclashx.desktop ~/.config/autostart/
```

> Важно: `Exec` содержит фикс чёрного экрана (см. [Проблему №6](#6-проблема-чёрное-окно-gui-flutter-на-powervr)).
> Без `LD_LIBRARY_PATH`/`LIBGL_ALWAYS_SOFTWARE` окно после перезагрузки будет чёрным.

---

## 4. Проверка работы

```bash
# 1. Процессы: GUI от пользователя + core от ROOT (setuid работает)
ps -o user,pid,cmd -C FlClashX
ps -o user,pid,cmd -C FlClashCore
#   GUI: USER     /usr/bin/FlClashX
#   core: root    /usr/share/FlClashX/FlClashCore /tmp/FlClashXSocket_*.sock

# 2. Порт 7890 (mixed-port) слушается
ss -tlnp | grep 7890

# 3. Прокси реально ходит в интернет
curl -x http://127.0.0.1:7890 -s -o /dev/null -w "Telegram: HTTP:%{http_code} (%{time_total}s)\n" https://api.telegram.org
#   → HTTP:302 — работает
curl -x http://127.0.0.1:7890 -s https://ipinfo.io/country
#   → страна выхода (зависит от выбранного узла)

# 4. Лог приложения (должен быть чистый старт)
tail -f ~/.local/share/com.follow.clashx/logs/FlClashX_$(date +%F).log
#   ОК: "Subscription info updated successfully"
#   НЕ ОК: "updateGroups error: unknown error, keeping old groups" (см. Проблему №5)
```

**Сервисы, зависящие от прокси** (боты, приложения, настроенные на localhost-прокси)
продолжают работать без изменений — FlClashX слушает mixed-port 7890, тот же порт,
что обычно использовал mihomo. Менять их конфиги не нужно.

---

## 5. Проблема: прокси не поднимается (setuid FlClashCore)

### Симптомы
- GUI FlClashX запущен, процессы есть
- Но порт 7890 **не слушается**
- В логе `~/.local/share/com.follow.clashx/logs/FlClashX_*.log` каждые ~22 сек:
  ```
  updateGroups error: unknown error, keeping old groups
  ```
- GUI показывает «Disconnected»/не активирует профиль

### Причина
На правильно настроенной машине ядро `/usr/share/FlClashX/FlClashCore` имеет
**setuid-бит** и запускается от **root**:
```
-rwsr-sr-x 1 root root  FlClashCore   ← правильно
```
После `dpkg -i` бит **пропадает**:
```
-rwxr-xr-x 1 root root  FlClashCore   ← после установки (сломано)
```
Без setuid ядро не может подняться с нужными привилегиями → профиль не активируется
→ порт 7890 молчит.

### Решение
```bash
sudo chmod 6755 /usr/share/FlClashX/FlClashCore
ls -la /usr/share/FlClashX/FlClashCore   # → -rwsr-sr-x
```
Затем перезапустить FlClashX. Проверка: core должен быть от root:
```bash
ps -o user,pid,cmd -C FlClashCore
# root  ... /usr/share/FlClashX/FlClashCore /tmp/FlClashXSocket_*.sock
```

> **Урок:** при переносе FlClashX на новую машину всегда проверяйте setuid-бит на
> FlClashCore — dpkg его не восстанавливает (или deb-пакет его не сохраняет).

---

## 6. Проблема: чёрное окно GUI (Flutter на PowerVR)

### Симптомы
- Окно FlClashX открывается, но **полностью чёрное** — ни кнопок, ни текста
- Прокси при этом **работает** (порт 7890 слушается) — но управлять GUI нельзя
- В stderr при запуске:
  ```
  Gdk-WARNING: GL implementation doesn't support any form of non-power-of-two textures
  ```

### Причина
FlClashX — Flutter-приложение, рендерит через OpenGL. Если на плате установлен
аппаратный GPU-стек PowerVR (DDK), его враппер `libEGL.so` лежит в `/usr/local/lib`
и подхватывается первым (например, через `ld.so.conf.d/00-pvr-priority.conf`).
Этот PVR-враппер **не поддерживает NPOT-текстуры** (non-power-of-two), которые Flutter
использует обязательно → окно не отрисовывается.

Диагностика: посмотреть, какую libEGL реально грузит процесс:
```bash
PID=$(pgrep -x FlClashX)
grep -E "libEGL|libgbm" /proc/$PID/maps | awk '{print $6}' | sort -u
# /usr/local/lib/libEGL.so.1.0.0   ← PVR-враппер (плохо)
# /usr/lib/aarch64-linux-gnu/libEGL_mesa.so.0  ← системный Mesa (хорошо)
```

### Решение
Запускать FlClashX с двумя переменными:
1. `LD_LIBRARY_PATH=/usr/lib/aarch64-linux-gnu` — чтобы брался системный Mesa libEGL,
   а не PVR-враппер из `/usr/local/lib`
2. `LIBGL_ALWAYS_SOFTWARE=1` — софтовый рендер (llvmpipe), который поддерживает NPOT

```bash
env LD_LIBRARY_PATH=/usr/lib/aarch64-linux-gnu LIBGL_ALWAYS_SOFTWARE=1 /usr/bin/FlClashX
```

После этого:
- stderr чистый (нет Gdk-WARNING про NPOT)
- окно отрисовывается: статус **Running**, Dashboard, список прокси, вкладки Proxies/Profiles/Rules
- `/proc/PID/maps` показывает системный Mesa libEGL

> ⚠️ Обязательно прописать этот env в autostart-файл (см. [Раздел 3](#3-автозапуск-mate)),
> иначе после перезагрузки окно снова будет чёрным.

> Примечание: `LIBGL_ALWAYS_SOFTWARE=1` сам по себе не помогает (окно остаётся чёрным),
> потому что Flutter тянет PVR libEGL напрямую. Ключевой фикс — именно `LD_LIBRARY_PATH`.

---

## 7. Полезные команды и пути

```bash
# Логи приложения
~/.local/share/com.follow.clashx/logs/FlClashX_YYYY-MM-DD.log

# Перезапуск FlClashX
pkill -x FlClashX; pkill -x FlClashCore
env LD_LIBRARY_PATH=/usr/lib/aarch64-linux-gnu LIBGL_ALWAYS_SOFTWARE=1 nohup /usr/bin/FlClashX &

# Порты профиля (из YAML): mixed 7890, socks 7891, redir 7892, контроллер 127.0.0.1:9090
grep -E "mixed-port|socks-port|redir-port|allow-lan" ~/.local/share/com.follow.clashx/profiles/*.yaml

# Проверка прокси (должно быть 302/200)
curl -x http://127.0.0.1:7890 -sI https://api.telegram.org
```

---

## 8. Уроки

1. **setuid FlClashCore** — после установки/переноса проверяй `chmod 6755`, иначе
   «updateGroups error» и мёртвый порт 7890.
2. **Чёрный экран Flutter на PowerVR** — PVR-враппер libEGL из `/usr/local/lib`
   не умеет NPOT. Лечится `LD_LIBRARY_PATH=/usr/lib/aarch64-linux-gnu` +
   `LIBGL_ALWAYS_SOFTWARE=1` (обязательно и в autostart).
3. **Автозапуск GUI-приложения** — только XDG autostart (MATE), не systemd-сервис.
4. **Перенос профиля** — копируй папку `~/.local/share/com.follow.clashx/` целиком
   и **обязательно меняй HWID** (`flutter.app_persistent_hwid`) на новый UUID.
5. **Прокси-зависимые сервисы** — ничего не менять: mixed-port 7890 тот же, что был
   у mihomo. Настроенные на localhost-прокси приложения продолжают работать.

---

*Связанные репозитории: [orangepi-zero3w-mihomo-setup](https://github.com/Haidegger22/orangepi-zero3w-mihomo-setup)
(старая схема mihomo, Debian 11) · [orangepi-zero3w-gpu-pcie](https://github.com/Haidegger22/orangepi-zero3w-gpu-pcie)
(GPU-стек PowerVR под Debian 13)*
