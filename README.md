# Лаб. П07-1/П07-2 — Ansible roles: nginx с vhosts через роль + введение в CI (GitHub Actions)

Практика по лекции «Ansible — roles» и «Введение в CI»: в одной лабораторной работе
собраны два связанных задания методички — (1) деплой nginx с несколькими
виртуальными хостами через переиспользуемую Ansible **роль** и (2) перенос
результата в отдельный GitHub-репозиторий с подключёнными **GitHub Actions**
(обычный тестовый pipeline + `ansible-lint`).

## Стенд

Та же инфраструктура, что и в П04/П05: две VM Ubuntu Server 24.04.3 LTS в Oracle
VirtualBox, NAT Network `lab4net` (`10.0.2.0/24`), переиспользована без изменений:

| ВМ | Роль | IP в `lab4net` | SSH с хоста |
|---|---|---|---|
| `devops-lab01` | Control node (Ansible, `gh` CLI) | `10.0.2.15` | `127.0.0.1:2223` |
| `devops-lab04-vm2` | Managed node (веб-сервер) | `10.0.2.16` | `127.0.0.1:2224` |

На `devops-lab01` уже установлен Ansible и настроен SSH-ключ `andrey` →
`runner@10.0.2.16` с passwordless sudo (из П04), переиспользуется как есть. `nginx`
на `devops-lab04-vm2` уже был установлен в П04/П05 — в этой лабе роль устанавливает
его заново идемпотентно (`ok`, без изменений) и добавляет виртуальные хосты.

## Содержимое

| Файл/папка | Назначение |
|---|---|
| `inventory.ini` | инвентарь: группа `webservers` → `10.0.2.16`, `ansible_user=runner` |
| `playbook.yml` | плейбук уровня play: `vars.sites` (список vhost'ов) + `roles: [nginx-vhosts]` |
| `roles/nginx-vhosts/tasks/main.yml` | install nginx → папки → index.html → nginx-конфиг (всё циклом по `sites`) |
| `roles/nginx-vhosts/handlers/main.yml` | `restart nginx` по `notify` |
| `roles/nginx-vhosts/templates/index.html.j2` | Jinja2-шаблон индексной страницы каждого vhost'а |
| `roles/nginx-vhosts/templates/site.conf.j2` | Jinja2-шаблон nginx server-блока каждого vhost'а |
| `run_playbook.sh` | обёртка `ansible-playbook ./playbook.yml -i ./inventory.ini --diff` |
| `.github/workflows/devops_course_pipeline.yml` | демо-pipeline из методички (`on: push`, echo/uptime/pwd/whoami) |
| `.github/workflows/lint.yml` | `ansible-lint` на `push` в `master` (`actions/checkout@v4` + `ansible/ansible-lint@main`) |
| `ci_logs/` | реальные логи обоих прогонов GitHub Actions (`gh run view --log`) |
| `screenshots/` | подтверждение работы — реальный вывод терминала и реальные прогоны Actions |
| `README.md` | этот отчёт |

## Задание и как оно выполнено

### П07-1 — «Деплой веб-сервера с помощью role Ansible»

> 1. Сделать role, которая устанавливает nginx c vhosts.
> 2. Роль устанавливает nginx.
> 3. Виртуальные сайты задавать параметрами роли (playbook/play/vars).
> 4. Для каждого vhost из шаблона `index.html.j2` создавать индексный файл.
> 5. Для каждого vhost из шаблона `{site}.conf.j2` создавать конфигурацию веб-сервера.
> 6. Проверка через `curl` или браузер с подменой DNS.

| # | Требование | Реализация |
|---|---|---|
| 1-2 | Роль устанавливает nginx | `roles/nginx-vhosts/tasks/main.yml`, задача `Install nginx` (`ansible.builtin.apt`) |
| 3 | Сайты — параметр роли | `playbook.yml`: `vars.sites: [site1.lab6.local, site2.lab6.local, site3.lab6.local]` на уровне **play**, роль использует `{{ sites }}` — ничего не захардкожено внутри роли, как и требует лекция («параметры нужно передавать, чтобы роль можно было переиспользовать») |
| 4 | index.html.j2 на каждый vhost | задача `Copy index from template` (`ansible.builtin.template`), цикл `loop: "{{ sites }}"` → `/var/www/{{ item }}/index.html` |
| 5 | `{site}.conf.j2` на каждый vhost | задача `Copy nginx config from template`, цикл по `sites` → `/etc/nginx/conf.d/site-{{ item }}.conf`, `notify: restart nginx` |
| 6 | Проверка curl/браузер | `curl 10.0.2.16 -H "Host: site*.lab6.local"` — честный HTTP-запрос с подменой заголовка `Host` на реальный запущенный nginx, без правки DNS/`/etc/hosts` |

Домены сайтов — заведомо фиктивные (`site1.lab6.local`, `.local`), а не реальные
внешние домены из примера методички, чтобы не создавать конфигурацию, ссылающуюся
на чужую/реальную инфраструктуру — по той же логике, что и выбор целей для `nmap`
в П05.

### П07-2 — «Добавление функций CI»

> 1. Создать репо на GitHub.
> 2. `git clone`.
> 3. Перенести файлы лабы.
> 4. Активировать Actions на стороне GitHub.
> 5. Создать и протестировать CI-пайплайны.

| # | Требование | Реализация |
|---|---|---|
| 1 | Новый репозиторий | `gh repo create devops-lab06-ansible-roles-ci --public --source=. --remote=origin` |
| 2-3 | Перенос файлов | Роль, плейбук, инвентарь и workflow'ы закоммичены и запушены в новый репозиторий (`git push -u origin master`) |
| 4 | Actions включены | Публичный репозиторий на GitHub — Actions включены по умолчанию; оба workflow реально выполнились по событию `push` |
| 5 | CI-пайплайны созданы и протестированы | Два workflow: `devops_course_pipeline.yml` (демо-пайплайн из методички) и `lint.yml` (`ansible-lint`) — оба запущены на настоящих GitHub-хостed раннерах, результаты подтверждены через `gh run list` / `gh run view --log` |

## Роль, плейбук и шаблоны

`playbook.yml`:
```yaml
---
- name: Install nginx with Virtual Hosts
  hosts: webservers
  remote_user: runner
  become: true
  become_method: sudo
  gather_facts: false

  vars:
    sites:
      - site1.lab6.local
      - site2.lab6.local
      - site3.lab6.local

  roles:
    - nginx-vhosts
```

`roles/nginx-vhosts/tasks/main.yml`:
```yaml
---
- name: Install nginx
  ansible.builtin.apt:
    name: nginx
    state: latest
    update_cache: true

- name: Create sites folders
  ansible.builtin.file:
    path: "/var/www/{{ item }}/"
    state: directory
    mode: "0755"
  loop: "{{ sites }}"

- name: Copy index from template
  ansible.builtin.template:
    src: index.html.j2
    dest: "/var/www/{{ item }}/index.html"
    owner: root
    group: root
    mode: "0644"
  loop: "{{ sites }}"

- name: Copy nginx config from template
  ansible.builtin.template:
    src: site.conf.j2
    dest: "/etc/nginx/conf.d/site-{{ item }}.conf"
    owner: root
    group: root
    mode: "0644"
  loop: "{{ sites }}"
  notify: restart nginx
```

`roles/nginx-vhosts/handlers/main.yml`:
```yaml
---
- name: restart nginx
  ansible.builtin.systemd:
    name: nginx
    state: restarted
```

`roles/nginx-vhosts/templates/site.conf.j2`:
```nginx
server {
        listen 80;
        listen [::]:80;

        root /var/www/{{ item }}/;
        index index.html;

        server_name {{ item }};

        location / {
                try_files $uri $uri/ =404;
        }
}
```

`.github/workflows/lint.yml`:
```yaml
name: lint

on:
  push:
    branches: ["master"]

jobs:
  build:
    name: Ansible Lint
    runs-on: ubuntu-24.04
    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      - name: Run ansible-lint
        uses: ansible/ansible-lint@main
```

## Развёртывание (команды, реально выполненные)

```bash
# --- control node devops-lab01: файлы роли загружены по SFTP в ~/lab6-ansible/ ---
find ~/lab6-ansible -type f | sort
# inventory.ini, playbook.yml, run_playbook.sh,
# roles/nginx-vhosts/{tasks,handlers,templates}/...

# --- сухой прогон ---
cd ~/lab6-ansible
ansible-playbook playbook.yml -i inventory.ini --check --diff
# nginx уже стоит (ok), создание vhost-папок/index/конфигов -> changed (x3 каждая),
# handler restart nginx -> changed
# PLAY RECAP: ok=5 changed=4 unreachable=0 failed=0

# --- реальный запуск ---
ansible-playbook playbook.yml -i inventory.ini --diff
# то же самое, но по-настоящему: /var/www/site*.lab6.local/index.html и
# /etc/nginx/conf.d/site-*.conf реально созданы, nginx реально перезапущен

# --- проверка через curl (Host header, без правки DNS) ---
curl -s 10.0.2.16 -H "Host: site1.lab6.local"
curl -s 10.0.2.16 -H "Host: site2.lab6.local"
curl -s 10.0.2.16 -H "Host: site3.lab6.local"
# каждый ответ содержит корректный "Hello! Site: siteN.lab6.local"

# --- подтверждение реальных файлов на managed node ---
ansible webservers -i inventory.ini -a "ls -la /etc/nginx/conf.d/" -b
ansible webservers -i inventory.ini -a "ls -la /var/www/" -b

# --- П07-2: новый репозиторий + CI ---
git init -q
git add .github inventory.ini playbook.yml roles run_playbook.sh
git commit -q -m "Lab 6: nginx vhosts Ansible role + GitHub Actions CI"

gh repo create devops-lab06-ansible-roles-ci --public \
  --description "DevOps Lab 6: Ansible roles (nginx vhosts) + GitHub Actions CI" \
  --source=. --remote=origin
git branch -M master
git push -u origin master

# --- проверка реальных прогонов Actions ---
gh run list --repo Andreykosberg/devops-lab06-ansible-roles-ci
gh run view <id-pipeline> --repo Andreykosberg/devops-lab06-ansible-roles-ci --log
gh run view <id-lint>     --repo Andreykosberg/devops-lab06-ansible-roles-ci --log
```

## Нюансы и наблюдения

- **Параметризация роли** — главное требование лекции: список `sites` объявлен в
  `playbook.yml` на уровне `vars:` play, а не внутри роли. Роль `nginx-vhosts`
  полностью переиспользуема — для другого набора vhost'ов достаточно поменять
  список `sites` в плейбуке, саму роль трогать не нужно.
- **nginx уже был установлен** (из П04/П05) — задача `Install nginx` в обоих
  прогонах (сухом и реальном) вернула `ok`, а не `changed`, что подтверждает
  идемпотентность `ansible.builtin.apt` с `state: latest`.
- **Диффы `--diff`** честно показали полный рендер каждого шаблона (создание папки
  `absent → directory`, полное содержимое `index.html`, полный nginx server-блок)
  для всех трёх vhost'ов — именно так, как это показано в примере методички.
- **Handler сработал один раз** — несмотря на то, что `notify: restart nginx`
  вызывается в цикле (3 раза, по одному на каждый `site.conf.j2`), Ansible
  поставил handler в очередь только один раз и выполнил его один раз в конце плея
  (видно по `RUNNING HANDLER` — один блок, а не три).
- **Проверка через `curl -H Host`, а не через правку DNS** — функционально
  эквивалентна правке `/etc/hosts`/`C:\Windows\System32\drivers\etc\hosts` из
  методички (там nginx тоже выбирает виртуальный хост по заголовку `Host`), но не
  требует прав администратора на хосте с браузером и не оставляет посторонних
  записей в системном файле hosts.
- **Домены — заведомо фиктивные** (`*.lab6.local`) вместо реальных внешних доменов
  из примера методички — чтобы не генерировать конфигурацию, ссылающуюся на чужую
  инфраструктуру, аналогично выбору целей `nmap` в П05.
- **`gh` и scope `workflow`** — первый `git push` с файлами в `.github/workflows/`
  был отклонён GitHub (`refusing to allow an OAuth App to create or update workflow
  ... without 'workflow' scope`), пока у локального `gh`-токена не было нужного
  OAuth-разрешения. Разрешение запрашивается явным интерактивным device-flow
  (`gh auth refresh -s workflow` → код подтверждения на `github.com/login/device`) —
  после подтверждения пользователем push прошёл успешно.
- **Оба workflow реально выполнились** на GitHub-хостед раннерах (`ubuntu-24.04`/
  `ubuntu-latest`) сразу после push, без ручного запуска:
  - `DevOps course pipeline` → **success**, вывод полностью совпадает с ожидаемым
    из методички (`I run my first Github Actions`, `uptime`, `pwd`
    → `/home/runner/work/devops-lab06-ansible-roles-ci/devops-lab06-ansible-roles-ci`,
    `whoami` → `runner`).
  - `lint` (`ansible-lint`) → **честный fail**, 4 реальные находки: `become_method`
    не входит в список допустимых значений схемы (`ansible.builtin.sudo` формально
    не в списке для используемой версии `ansible-lint`), имя роли `nginx-vhosts`
    не соответствует паттерну `^[a-z][a-z0-9_]*$` (дефис), имя handler'а не
    начинается с заглавной буквы, `state: latest` считается анти-паттерном
    идемпотентности. Это **не ошибка настройки CI**, а реальный результат
    статического анализа — ровно такая же картина (genuine fail на нескольких
    правилах) показана и в собственном примере лектора на слайде методички;
    результат сохранён как есть, без подгонки кода под линтер.

## Скриншоты

1. `screenshots/01-role-structure-playbook.png` — структура роли `nginx-vhosts` на
   control node + полное содержимое `playbook.yml` (vars.sites + roles).
2. `screenshots/02-tasks-handlers.png` — `roles/nginx-vhosts/tasks/main.yml` (install
   + 3 задачи с `loop`) и `handlers/main.yml` (`restart nginx`).
3. `screenshots/03-dry-run-check-diff.png` — `--check --diff`: nginx уже установлен
   (`ok`), остальные задачи `changed` с полными диффами по каждому vhost'у.
4. `screenshots/04-real-run.png` — реальный запуск (`--diff`, без `--check`):
   `ok=5 changed=4`, handler `restart nginx` сработал один раз.
5. `screenshots/05-curl-verify.png` — `curl -H Host` по всем трём vhost'ам (разный
   контент на каждый) + подтверждение реальных файлов на `devops-lab04-vm2`.
6. `screenshots/06-github-actions-runs.png` — создание репозитория, push, список
   реальных прогонов Actions (`gh run list`): pipeline — success, lint — failure.
7. `screenshots/07-pipeline-run-log.png` — реальный лог успешного прогона
   `devops_course_pipeline` (`gh run view --log`).
8. `screenshots/08-lint-run-log.png` — реальный лог честного fail `ansible-lint`
   с 4 находками (`gh run view --log`).
