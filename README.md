# docker-builder

__Прототип__. Скрипт для компиляции софта и сборки rpm/deb пакeтов внутри
docker контейнера. Все действия производятся внутри временного каталога
build-env, в котором нужно создать стандартную иерархию (по-минимуму):

* для rpm: volume/{SOURCES,SPECS}
* для deb: volume/debian/DEBIAN/{control,changelog}

build-env подключается в контейнер как volume. Готовые пакеты можно забрать в:

* build-env/RPMS/
* build-env/SRPMS/
* build-env/DEBS/

[Пример использование](https://github.com/tierpod/docker-makefiles).

## Зависимости

* python
* docker

## Установка

* Исполняемый bin/docker-builder.py положить куда-нибудь в PATH.
* Создать иерархию с нужными метаданными:
  * docker-builder.ini - файл настроек  
  * ${workdir}/Dockerfile.template - шаблон для генерации Dockerfile,
  * для rpm:
    * ${workdir}/volume/SPECS/file.spec - spec-файл,
    * ${workdir}/volume/SOURCES/ - дополнительные файлы, патчи и прочее.
  * для deb:
    * ${workdir}/volume/debian/DEBIAN/{control,changelog} - метаданные для 
	  сборки deb пакета
	* ${workdir}/volume/debian/ - бинарные файлы программы


## Использование

```
Build package inside docker container

positional arguments:
  {image,package,shell,generate,clear,show}
    image               Build docker image
    package             Build package inside docker container
    shell               Run interactive shell inside docker container
    generate            Generate and write Dockerfile files to disk
    clear               Clear temporary files
    show                Show configuration and exit

optional arguments:
  -h, --help            show this help message and exit
  -c CONFIG, --config CONFIG
                        Config file [default: docker-builder.ini]
  -d, --debug           Print more debug messages
  -s SECTION, --section SECTION
                        Switch between config sections [default: default]
```

Чтобы собрать из этого репозтория пакет, нужно на хосте с docker-ом (либо 
jenkins-slave с docker-ом):

```
# build
cd project/
docker-builder.py image
docker-builder.py package
```

При этом, желательно разделить на 2 этапа, а не выполнять в одной задаче.

## docker-builder.py

Использует переменные окружения:

* BUILD_NUMBER - добавляет номер сборки в версию пакета, удобно в связке с
  jenkins.

## docker-builder.ini

Стандартные параметры:

```
default_config = {
    # имя шаблона Dockerfile
    'dockerfile' : 'Dockerfile.template',
    # имя шаблона entrypoint.sh
    'entrypoint': 'entrypoint.sh',
    # сгенерировать release из версии git commit?
    'git': False,
    # имя docker image-а, в котором будет происходить компиляция
    'image': 'builder',
    # команда или скрипт для запуска на хосте ДО запуска конейнера
    'prepare': None,
    # имя spec-файла, только для rpm
    'target': None,
    # рабочая директория, по-умолчанию, текущая
    'workdir': None,
}
```

## Dockerfile.template

Шаблон для генерации Dockerfile, в процессе generate заменяются:

* {userid}, {groupid} - на uid, gid текущего пользователя. Необходимо, чтобы
  внутри контейнера создать пользователя без прав рута с правильными UID и GID
  для доступа к volume.
* {target} - для копирования временного spec-файла в /tmp, чтобы установить
  зависимости с помощью yum-builddep; для deb - можно вставить имя пакета,
  чтобы установить сборочные завимиости из репозитория.

## entrypoint.sh

entrypoint.sh, выполняется внутри контейнера (CMD в Dockerfile). Использует
следующие переменные окружения:

* TARGET - имя spec-файла, берётся из файла конфигурации,
* RELEASE - дополнительное поле, которое может присутсвовать в spec-файле (полезно
  для добавления версии из git, или BUILD_NUMBER из jenkins-а).
