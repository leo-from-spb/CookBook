Избавление от изменчивых файлов
===============================

Чтобы избавиться от файлов, которые часто меняются сторонними программами, но их нельзя check-in'ить, можно использовать `update-index --assume-unchanged`:

Чтобы скрыть файлы:
```shell
git update-index --assume-unchanged path/to/files...
```

Чтобы вернуть файлы:
```shell
git update-index --no-assume-unchanged path/to/files...
```

Чтобы посмотреть скрытые файлы:
```shell
git ls-files -v | grep '^h'
```

Можно добавить алиасы:
```ini
[alias]
	hide = update-index --assume-unchanged
	unhide = update-index --no-assume-unchanged
	hidden = !git ls-files -v | grep '^h'

```