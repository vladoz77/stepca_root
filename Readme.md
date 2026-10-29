# step_ca_root

Ansible-роль, которая скачивает корневой сертификат [step-ca](https://smallstep.com/docs/step-ca/), **сверяет его по fingerprint** и устанавливает в системное хранилище доверия хоста.

После установки `curl`, `wget`, `docker pull`, `git` и другие утилиты, использующие системное хранилище, начинают доверять сертификатам, выпущенным вашим CA.

## Как это работает

1. Роль нормализует заданный fingerprint (убирает `:`, приводит к нижнему регистру).
2. Запрашивает корень у CA по `<step_ca_root_url>/root/<fingerprint>`. TLS при этом **не проверяется** (`validate_certs: false`): корень ещё не доверенный, проверить соединение нечем.
3. Считает SHA-256 полученного сертификата самостоятельно через `openssl` и сравнивает с ожидаемым. При расхождении роль падает, ничего не устанавливается.
4. Копирует сертификат в каталог хранилища ОС. Handler обновляет хранилище только если файл изменился.

Безопасность держится на шаге 3, а не на канале загрузки: подменённый ответ не даст нужный SHA-256. Сам fingerprint нужно получать по независимому доверенному каналу (с хоста CA), а не из ответа того же сервера.

## Требования

- Ansible (`ansible-core` 2.14+).
- На целевых хостах: пакет `openssl`, права root (`become: true`).
- Целевые хосты должны резолвить и достигать `step_ca_root_url`.
- Поддерживаемые ОС: семейства `Debian` (Debian, Ubuntu и производные) и `RedHat` (Rocky, RHEL, CentOS, Fedora). Семейство определяется по `ansible_facts['os_family']`.

## Переменные

| Переменная | По умолчанию | Описание |
|---|---|---|
| `step_ca_root_url` | `https://ca.lxd1.int.devhomelab.site` | Адрес CA, без завершающего `/` |
| `step_ca_root_fingerprint` | `""` | **Обязательная.** SHA-256 fingerprint корня; допустим любой формат (`58:04:6C...` или `58046c...`) |
| `step_ca_root_name` | `step-ca-root` | Имя файла сертификата в хранилище (без расширения) |

Переменные из `vars/<семейство ОС>.yaml` (каталог, расширение, команда обновления) менять не нужно.

## Как получить fingerprint

На хосте с CA:

```bash
docker exec stepca step certificate fingerprint /home/step/certs/root_ca.crt
```

Или, если корень уже скачан:

```bash
openssl x509 -in root_ca.crt -noout -fingerprint -sha256
step certificate fingerprint root_ca.crt
```

Значение публичное, секретом не является. Удобно хранить в `group_vars/all.yaml`.

## Использование

```yaml
- hosts: all
  become: true
  roles:
    - role: step_ca_root
      vars:
        step_ca_root_url: https://ca.lxd1.int.devhomelab.site
        step_ca_root_fingerprint: "58046c61ce750f9dd7418290ac20be1c8bcbefdacc85f412a78a9a2eab67d639"
```

Подключение из репозитория через `requirements.yml`:

```yaml
roles:
  - name: step_ca_root
    src: https://github.com/<user>/<repo>.git
    scm: git
    version: v1.0.0
```

### Запуск на локальной машине

```yaml
# playbook.yaml
- hosts: localhost
  connection: local
  become: true
  roles:
    - role: step_ca_root
      vars:
        step_ca_root_fingerprint: "<fingerprint>"
```

```bash
ansible-playbook playbook.yaml -K
```

## Проверка

```bash
curl https://ca.lxd1.int.devhomelab.site/health
```

Запрос без `-k` должен вернуть `{"status":"ok"}`. Если так, система доверяет корню.

## Структура роли

```
step_ca_root/
├── defaults/main.yaml
├── handlers/main.yaml
├── meta/main.yaml
├── molecule/
│   ├── bad_fingerprint/
│   │   ├── converge.yml
│   │   ├── molecule.yml
│   │   ├── prepare.yml
│   │   └── verify.yml
│   └── default/
│       ├── converge.yml
│       ├── molecule.yml
│       ├── prepare.yml
│       └── verify.yml
├── tasks/main.yaml
└── vars/
    ├── Debian.yaml
    └── RedHat.yaml
```

## Тестирование

Роль покрыта [Molecule](https://github.com/ansible/molecule) двумя сценариями, оба — против **настоящего** [`smallstep/step-ca`](https://hub.docker.com/r/smallstep/step-ca):

- **`molecule/default/`** — позитивный путь: верный fingerprint → корень установлен, системе можно доверять.
- **`molecule/bad_fingerprint/`** — негативный путь: неверный fingerprint → роль отказывает и корень **не** устанавливается.

Каждый сценарий поднимает три контейнера:

- **`stepca`** — реальный `step-ca`, который сам инициализирует PKI и слушает `https://stepca:9000`;
- **`debian12`** и **`rockylinux9`** — тестовые хосты (Debian и RedHat семейства).

### `default` — позитивный путь

1. `converge`: запрашивает fingerprint корня **у самого CA** (`docker exec stepca step certificate fingerprint …`) — так, как это сделал бы оператор, — и прогоняет роль с ним.
2. `idempotence`: повторный запуск роли не меняет систему.
3. `verify`: TLS-проверка — клиент с системным хранилищем хоста должен пройти handshake с `https://stepca:9000/health` (`validate_certs: true`). Если бы корень установился неверно, handshake упал бы.

### `bad_fingerprint` — негативный путь

1. `converge`: прогоняет роль с заведомо неверным fingerprint (`000…0`) и требует, чтобы роль **отказалась** (`block`/`rescue`). Шаг `idempotence` здесь не используется — роль всегда падает, стабильного состояния нет.
2. `verify`: та же TLS-проверка (`validate_certs: true`) должна **упасть** — CA не должен быть доверен после отклонённого запуска.

Пара сценариев доказывает обе стороны одного свойства: верный fingerprint → система доверяет корню, неверный → не доверяет.

Требования на хосте: Docker, `pip install molecule molecule-plugins`, коллекции `community.docker` и `ansible.posix`.

```bash
molecule test                      # позитивный сценарий (default)
molecule test -s bad_fingerprint   # негативный сценарий

molecule converge                  # только поднять и прогнать роль (отладка)
molecule verify                    # только проверки, после converge
molecule destroy                   # убрать контейнеры
```

> Сценарии используют одинаковые имена контейнеров (`stepca`, `debian12`, `rockylinux9`) и общую сеть `stepca-net`, поэтому запускать их нужно **последовательно**, а не параллельно.

## Ограничения

- Роль меняет только **системное** хранилище хоста. Приложения со своим хранилищем (Java/truststore, Firefox, часть Python-окружений) нужно настраивать отдельно.
- Контейнеры не наследуют хранилище хоста. Для Traefik в Docker корень нужно монтировать в контейнер отдельно (например, через `LEGO_CA_CERTIFICATES`).
- Docker и containerd читают системное хранилище при старте: после первой установки корня их может потребоваться перезапустить.
- Если CA отдаёт несколько корней (ротация), роль работает только с тем, чей fingerprint указан в `step_ca_root_fingerprint`.
- При смене корня (например, пересоздан volume CA) fingerprint изменится: обновите `step_ca_root_fingerprint`, иначе проверка не пройдёт.

## Лицензия

MIT