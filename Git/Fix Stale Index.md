Fix Stale Index
===============

Иногда Git выдаёт ошибки такого вида:
```
warning: could not open directory 'path/to/an/unexistent/directory/': No such file or directory
warning: could not open directory 'path/to/an/unexistent/directory/': No such file or directory
```

Это возникает из-за того, что git index (это некоторый индекс, который отслеживает состояние рабочей директории) гниёт.

Исправить можно так:

```shell
rm .git/index
git reset
```

Первая команда удаляет индекс, а вторая — пересобирает его заново, поэтому может потребовать некоторое время.