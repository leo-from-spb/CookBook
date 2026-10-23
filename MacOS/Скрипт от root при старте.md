Скрипт от root при старте
=========================

Чтобы macOS при каждой загрузке выполняла скрипт от root, нужен **LaunchDaemon**. Это plist-файл в `/Library/LaunchDaemons/`, по которому `launchd` запускает программу от root, ещё до логина пользователя.

Пример ниже: скрипт, который возвращает компьютеру правильное имя через `scutil`.

Значения `scutil --set` сохраняются постоянно (в `/Library/Preferences/SystemConfiguration/preferences.plist`), но их может перезаписать кто-то другой, например MDM. Поэтому скрипт запускается не только при старте, но и при каждом изменении сетевой конфигурации.

Скрипт
------

Файл `/usr/local/sbin/fixName.sh`:

```zsh
#!/bin/zsh

# Ставит значение, только если оно отличается от нужного.
# Это важно: scutil --set пишет в каталог, за которым следит WatchPaths,
# и без проверки скрипт запускал бы сам себя по кругу.
set_name() {
	local key=$1 value=$2
	if [[ "$(/usr/sbin/scutil --get $key 2>/dev/null)" != "$value" ]]; then
		/usr/sbin/scutil --set $key "$value"
		echo "$(date '+%F %T') $key -> $value"
	fi
}

set_name ComputerName  Treant
set_name LocalHostName treant
set_name HostName      treant.labs.intellij.net
```

Что важно:

- Скрипт лежит в системном каталоге и принадлежит root. Если держать его, например, в `~/Scripts`, любая программа от моего пользователя сможет его подменить и получить root.
- Нужен shebang (`#!/bin/zsh`) в первой строке, иначе `launchd` не сможет запустить скрипт.
- `sudo` внутри не нужен, скрипт и так выполняется от root.
- `PATH` у `launchd` урезанный, поэтому команды лучше писать с полным путём (`/usr/sbin/scutil`).

Установить скрипт:

```shell
sudo mkdir -p /usr/local/sbin
sudo install -o root -g wheel -m 755 fixName.sh /usr/local/sbin/fixName.sh
```

plist
-----

Файл `/Library/LaunchDaemons/com.leonid.fixname.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>Label</key>
	<string>com.leonid.fixname</string>

	<key>ProgramArguments</key>
	<array>
		<string>/usr/local/sbin/fixName.sh</string>
	</array>

	<!-- запустить при загрузке -->
	<key>RunAtLoad</key>
	<true/>

	<!-- и при каждом изменении сетевой конфигурации -->
	<key>WatchPaths</key>
	<array>
		<string>/Library/Preferences/SystemConfiguration</string>
	</array>

	<key>StandardOutPath</key>
	<string>/var/log/fixname.log</string>
	<key>StandardErrorPath</key>
	<string>/var/log/fixname.log</string>
</dict>
</plist>
```

Права обязательно `root:wheel` и `644`, иначе `launchd` молча проигнорирует файл:

```shell
sudo chown root:wheel /Library/LaunchDaemons/com.leonid.fixname.plist
sudo chmod 644 /Library/LaunchDaemons/com.leonid.fixname.plist
plutil -lint /Library/LaunchDaemons/com.leonid.fixname.plist
```

Включить
--------

После перезагрузки задание подхватится само. Чтобы включить сразу, без перезагрузки:

```shell
sudo launchctl bootstrap system /Library/LaunchDaemons/com.leonid.fixname.plist
```

macOS покажет уведомление «Background item added», а задание появится в System Settings → General → Login Items & Extensions. Там его можно случайно выключить.

Проверить
---------

```shell
sudo launchctl print system/com.leonid.fixname           # статус и last exit code
sudo launchctl kickstart -k system/com.leonid.fixname    # запустить вручную
cat /var/log/fixname.log                                 # что скрипт поменял
```

Выключить
---------

```shell
sudo launchctl bootout system/com.leonid.fixname
sudo rm /Library/LaunchDaemons/com.leonid.fixname.plist
```

Если поменять plist, его нужно выгрузить (`bootout`) и загрузить заново (`bootstrap`).

Чем не стоит пользоваться
-------------------------

- `cron @reboot` формально работает, но Apple его не развивает.
- Login hooks (`defaults write com.apple.loginwindow LoginHook ...`) давно deprecated.
- LaunchAgent (`~/Library/LaunchAgents`) запускается от пользователя, а не от root.
