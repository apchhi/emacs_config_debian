# Терминальный Emacs 30 на Debian 13 — воспроизводимая настройка Python IDE

> Проверенная конфигурация: Debian 13 (Trixie), KDE Plasma / Wayland, Konsole, Emacs 30.1 (`emacs-pgtk`).  
> Цель: лёгкая среда для ручной разработки Python без тяжёлых IDE-сборок, Evil Mode и Tree-sitter.
>
> **Статус:** в исходном сеансе успешно проверены Eglot + pylsp, ручное дополнение, Ruff (F6/F7), pytest (F8), `project.el`, Git VC, Markdown и YAML. Окончательный `init.el` приведён ниже как воспроизводимая сборка по результатам настройки; после восстановления нужно выполнить проверки.

## 1. Состав и принципы

- Emacs запускается **в Konsole**: `emacs -nw`.
- Конфигурация: `~/.emacs.d/init.el`.
- Встроенные функции: `eglot`, `project.el`, `vc`, `completion-preview-mode`, `compile`.
- Внешние инструменты: `pylsp` (APT), `ruff` (pipx), `pytest` (APT и, отдельно, в `.venv`).
- Тема Catppuccin Mocha устанавливается из MELPA.
- Markdown и YAML устанавливаются из APT.
- Tree-sitter-парсеры не нужны; `python-mode`, `js-json-mode`, `conf-toml-mode`, `sql-mode` работают без них.
- Автоматического исправления Ruff при сохранении нет: **F6 проверяет, F7 форматирует по команде**.
- **F8** использует `.venv/bin/python`, если он есть в корне Git-проекта, иначе системный `python3`.
- Установка не изменяет другие ОС и системный Python.

## 2. Установка пакетов

```bash
sudo apt update
sudo apt install --no-install-recommends \
  emacs-pgtk python3-pylsp python3-pytest python3-venv \
  git ripgrep elpa-markdown-mode elpa-yaml-mode pipx
```

Проверить:

```bash
emacs --version | head -n 2
python3 --version
pylsp --version
pytest --version
git --version
rg --version
```

На исходной машине были: Emacs 30.1, Python 3.13.5, pylsp 1.12.0, системный pytest 8.3.5, Git 2.47.3, ripgrep 14.1.1. Это исторические версии, **не требования** к следующей установке.

### Ruff через pipx

```bash
pipx ensurepath
pipx install ruff
~/.local/bin/ruff --version
```

Если `pipx ensurepath` сообщает, что путь уже в `PATH`, это нормально, `--force` не нужен. Ruff был проверен с версией 0.17.3.

## 3. Сохранить существующие настройки

Если Emacs использовался ранее, **не перезаписывайте** конфигурацию без резервной копии:

```bash
mkdir -p ~/.emacs.d
if [ -f ~/.emacs.d/init.el ]; then
  cp -a ~/.emacs.d/init.el ~/.emacs.d/init.el.before-setup
fi
```

Проверить, откуда Emacs загружает файл, можно внутри Emacs: `Esc`, затем `:` и выражение `user-init-file` (это эквивалент `M-:`). В нашем случае используется `~/.emacs.d/init.el`. При существующем `~/.emacs.d` конфигурация в `~/.config/emacs/init.el` может не загрузиться.

## 4. Catppuccin из MELPA

В `init.el` ниже MELPA подключается автоматически при запуске, но **пакет темы надо установить один раз**.

Для первой установки удобно сначала создать `init.el` с конфигурацией из раздела 5, временно **закомментировав строку `(load-theme 'catppuccin t)`**, затем запустить Emacs:

```bash
emacs -nw
```

В Emacs: `Esc` → `x` → `package-refresh-contents` → Enter, затем `Esc` → `x` → `package-install` → Enter → `catppuccin-theme` → Enter.

После установки убрать комментарий у `(load-theme 'catppuccin t)` и перезапустить Emacs.

> MELPA — внешний репозиторий Emacs. Для полной воспроизводимости можно зафиксировать версию темы или сохранить копию установленного пакета. Обновления MELPA выполняются **отдельно от APT**.

## 5. Полный `~/.emacs.d/init.el`

```elisp
;;; init.el --- Minimal terminal Emacs for Python -*- lexical-binding: t; -*-

;; Базовые настройки
(setq inhibit-startup-screen t)
(setq ring-bell-function 'ignore)
(global-display-line-numbers-mode 1)
(global-hl-line-mode 1)
(show-paren-mode 1)
(column-number-mode 1)
(setq-default indent-tabs-mode nil)
(setq-default tab-width 4)
(savehist-mode 1)
(recentf-mode 1)
(save-place-mode 1)
(setq confirm-kill-emacs #'yes-or-no-p)

;; Менеджер пакетов и оформление
(require 'package)
(add-to-list 'package-archives
             '("melpa" . "https://melpa.org/packages/") t)
(package-initialize)
(setq catppuccin-flavor 'mocha)
(load-theme 'catppuccin t)

;; Python LSP: pylsp (устанавливается через APT)
(require 'eglot)
(add-hook 'python-mode-hook #'eglot-ensure)
(add-hook 'python-ts-mode-hook #'eglot-ensure)
(setq eldoc-idle-delay 0.5)
(setq eglot-events-buffer-size 0)

;; Встроенное автодополнение Emacs 30
(setq completion-preview-minimum-symbol-length 3)
(add-hook 'prog-mode-hook #'completion-preview-mode)

;; F6: Ruff — только проверка сохранённого файла
(defun my/ruff-check ()
  "Проверить текущий файл Python через Ruff."
  (interactive)
  (unless buffer-file-name
    (user-error "Сначала сохрани файл"))
  (save-buffer)
  (let ((file (shell-quote-argument
               (file-relative-name buffer-file-name default-directory))))
    (compile
     (format "ruff --color never check --output-format concise %s" file))))
(global-set-key (kbd "<f6>") #'my/ruff-check)

;; F7: Ruff — форматирование только по запросу пользователя
(defun my/ruff-format ()
  "Форматировать текущий Python-буфер через Ruff."
  (interactive)
  (unless (derived-mode-p 'python-mode 'python-ts-mode)
    (user-error "Это не Python-буфер"))
  (unless buffer-file-name
    (user-error "Сначала сохрани файл"))
  (unless (executable-find "ruff")
    (user-error "Ruff не найден в PATH"))
  (let ((output (generate-new-buffer " *ruff-format*")))
    (unwind-protect
        (let ((status
               (call-process-region
                (point-min) (point-max)
                "ruff" nil output nil
                "format" "--stdin-filename" buffer-file-name "-")))
          (if (equal status 0)
              (progn
                (replace-buffer-contents output)
                (save-buffer)
                (message "Ruff: форматирование завершено"))
            (user-error "Ruff: ошибка форматирования (код %s)" status)))
      (kill-buffer output))))
(global-set-key (kbd "<f7>") #'my/ruff-format)

;; F8: pytest из .venv (если есть), иначе системный Python
(require 'project)
(defun my/pytest-run ()
  "Запустить pytest из корня текущего проекта."
  (interactive)
  (save-some-buffers t)
  (let* ((project (project-current nil))
         (root (if project (project-root project) default-directory))
         (venv-python (expand-file-name ".venv/bin/python" root))
         (python (if (file-executable-p venv-python)
                     venv-python
                   (or (executable-find "python3")
                       (user-error "Python3 не найден"))))
         (default-directory root)
         (compilation-buffer-name-function
          (lambda (_mode) "*pytest*")))
    (compile
     (format "%s -m pytest -q --color=no"
             (shell-quote-argument python)))))
(global-set-key (kbd "<f8>") #'my/pytest-run)

;;; init.el ends here
```

**Важно о `eglot-workspace-configuration`:** в ходе настройки пробовался блок `(setq-default eglot-workspace-configuration (lambda ...))`, но он был удалён: такой способ не соответствует документированному формату переменной. **Не добавляйте его обратно**. F8 уже работает с `.venv`; выбор `.venv` для анализа зависимостей через `pylsp` пока не автоматизирован и не подтверждён.

## 6. Тестовый Python-проект

```bash
mkdir -p ~/coding/python/python-test
cd ~/coding/python/python-test
git init
git branch -m main
python3 -m venv .venv
.venv/bin/python -m pip install pytest
```

Создайте `test_calculator.py`:

```python
def add(a, b):
    return a + b


def test_add():
    assert add(2, 3) == 5


def test_add_negative():
    assert add(-2, -3) == -5
```

Проверить:

```bash
.venv/bin/python -m pytest -q
```

Ожидается `2 passed`. Затем откройте:

```bash
emacs -nw test_calculator.py
```

Должен подключиться Eglot (`pylsp`). **F8** должен выводить команду с полным путём до `.venv/bin/python` и сообщать `2 passed`.

Для проверки Ruff можно создать `ruff_test.py`:

```python
import os
import sys


def greet(name):
    unused = 123
    return f"Hello, {name}"
```

`F6` должен обнаружить F401 (два импорта) и F841 (переменная). `F7` форматирует Python-код по запросу. Если Ruff завершает проверку с кодом 1 из-за найденных ошибок, это ожидаемо.

## 7. `.gitignore` для Python + Emacs

```gitignore
__pycache__/
*.py[cod]
.pytest_cache/
.ruff_cache/
.venv/
venv/
*~
\#*\#
.#*
```

Создайте `.gitignore` в корне проекта и проверьте:

```bash
git status --short
```

Временные файлы, автосохранения, кеши и `.venv` не должны попадать в список неотслеживаемых файлов. `.gitignore` **ничего не удаляет**.

## 8. Шпаргалка горячих клавиш

Сочетания с `→` нажимаются **последовательно**, не одновременно.

| Клавиши | Действие |
|---|---|
| **F6** | Ruff: проверить текущий файл |
| **F7** | Ruff: форматировать текущий Python-файл |
| **F8** | pytest: тесты проекта, предпочитая `.venv` |
| `C-x C-f` | Открыть файл |
| `C-x C-s` | Сохранить |
| `C-x C-c` | Выйти |
| `C-x 0` | Закрыть текущее окно |
| `C-x 1` | Оставить одно окно |
| `C-x 2` | Разделить окно сверху/снизу |
| `C-x 3` | Разделить окно слева/справа |
| `C-x o` | Перейти в другое окно |
| `C-x b` | Переключить буфер |
| `C-x C-b` | Список буферов |
| `C-x p f` | Найти файл проекта |
| `C-x p g` | Поиск регулярного выражения по проекту |
| `C-x p b` | Переключить буфер проекта |
| `C-s` / `C-r` | Поиск вперёд / назад |
| `C-g` | Отменить текущую команду |
| `C-/` | Undo |
| `M-x` | Выполнить команду Emacs |
| `M-.` | Перейти к определению (Xref) |
| `C-x \`` | Перейти к следующей ошибке |

`C` = Ctrl; `M` = Meta/Alt. Если системное `Alt+Shift` переключает раскладку, используйте **`Esc`, отпустить, затем следующую клавишу** для Meta-команд. Например, `M-x` = `Esc` → `x`. Для `M-:` = `Esc` → `Shift+;` в соответствующей раскладке.

## 9. Поддерживаемые режимы файлов

| Расширение | Режим |
|---|---|
| `.py` | `python-mode` |
| `.md` | `markdown-mode` (APT) |
| `.yaml` / `.yml` | `yaml-mode` (APT) |
| `.json` | `js-json-mode` |
| `.toml` | `conf-toml-mode` |
| `.sql` | `sql-mode` |

Встроенные функции `*-ts-mode` не требуют использования: на исходной машине Tree-sitter-грамматики отсутствовали.

## 10. Обновление и обслуживание

Системные пакеты Debian:

```bash
sudo apt update
sudo apt upgrade
```

Ruff:

```bash
pipx upgrade ruff
```

Catppuccin из MELPA: в Emacs запустить `M-x package-list-packages` (для обновления списка предварительно `M-x package-refresh-contents`); обновлять установленную тему контролируемо. Не требуется запускать фоновый менеджер плагинов.

Проверка загрузки конфигурации:

```bash
emacs --batch -l ~/.emacs.d/init.el \
  --eval '(message "Emacs config: OK")'
```

В исходной установке эта команда завершилась сообщением `Emacs config: OK` без ошибок. Обратите внимание: `--batch` сам по себе не загружает пользовательский `init.el`, поэтому здесь он явно подключается через `-l`.

Резервная копия:

```bash
cp -a ~/.emacs.d/init.el ~/.emacs.d/init.el.backup
```

Восстановление:

```bash
cp -a ~/.emacs.d/init.el.backup ~/.emacs.d/init.el
```

## 11. Измерение ресурсов

При открытом `emacs -nw` и подключённом pylsp:

```bash
ps -C emacs -o pid,rss,%mem,etime,args
ps -C pylsp -o pid,rss,%mem,etime,args
```

В исходном тесте RSS составлял **89 036 КиБ у Emacs** и **51 616 КиБ у pylsp**, суммарно около **137,4 МиБ** (сумма RSS не равна точной уникальной памяти: есть общие страницы). Это ориентир, а не гарантированный результат.

## 12. Известные особенности и устранение проблем

- **Тема не загружается:** проверьте, что `catppuccin-theme` установлен в Emacs через MELPA; временно закомментируйте `(load-theme 'catppuccin t)` для запуска без темы.
- **`user-init-file` равен `nil` в `emacs --batch`:** это нормально; для проверки загрузите файл явно с `-l`. В интерактивном Emacs `M-:` → `user-init-file` покажет реальный путь.
- **Ruff печатает `^[[` или `^[]8` в `*compilation*`:** используйте `ruff --color never check --output-format concise ...`.
- **pytest завершился с кодом 1:** это ожидаемо, если тесты провалены.
- **`ruff` не найден в Emacs:** проверьте `command -v ruff` и `~/.local/bin/ruff --version`; при необходимости перезапустите Konsole, чтобы подхватился `PATH`.
- **F8 не видит pytest:** установите pytest в окружение проекта: `.venv/bin/python -m pip install pytest`.
- **Eglot дополняет системные библиотеки вместо `.venv`:** это известный *не закрытый* вопрос данной конфигурации; не следует считать автоматическую настройку Jedi подтверждённой. Если возникнут реальные проблемы с импортами проекта, настройте Jedi через документированные параметры pylsp/директории проекта и проверьте путь разрешённого модуля.
- **`vc-dir` показывает резервные копии Emacs и Python-кеш:** проверьте `.gitignore`.
- **`Can't guess python-indent-offset`:** сообщение на пустом файле, Emacs выбрал отступ в четыре пробела.

---

**Результат:** терминальный Emacs под Debian 13 для Python с ручным контролем форматирования и тестов, встроенными Git/поиском/навигацией и минимумом внешних расширений. Документ не меняет архитектуру каких-либо прикладных проектов и предназначен исключительно для воспроизведения настройки редактора.
