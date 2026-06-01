

Мы будем использовать утилиты из пакета `ldap-utils`, в частности команду `ldapadd`, для добавления записей из LDIF-файлов.

### 1. Создание организационных единиц (OU)

Прежде чем добавлять пользователей, необходимо создать контейнеры (организационные единицы), в которых они будут храниться. Обычно это `people` (или `users`) для пользователей и `groups` для групп.

Для этого создайте файл, например `base.ldif`:

```bash
sudo nano base.ldif
```

Вставьте в него следующее содержимое, **обязательно заменив** `dc=srv,dc=unlix.ru` на ваше доменное имя (суффикс), которое вы указали при установке slapd:

```ldif
dn: ou=people,dc=srv,dc=unlix.ru
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=srv,dc=unlix.ru
objectClass: organizationalUnit
ou: groups
```

Теперь добавьте эти записи в ваш LDAP-каталог с помощью команды `ldapadd`:

```bash
sudo ldapadd -x -D cn=admin,dc=srv,dc=unlix.ru -W -f base.ldif
```

Разберем ключи команды:
*   `-x`: использовать простую аутентификацию.
*   `-D`: `cn=admin,dc=srv,dc=unlix.ru` — **Distinguished Name (DN)** администратора LDAP. Замените суффикс на свой.
*   `-W`: будет запрошен пароль администратора.
*   `-f`: путь к вашему LDIF-файлу.

Система запросит пароль администратора и, при успешном выполнении, выведет `adding new entry "ou=people..."`.

### 2. Добавление пользователя (inetOrgPerson + posixAccount)

Для хранения Unix-атрибутов (UID, GID, домашняя директория, shell) используются объектные классы `posixAccount` и `shadowAccount`, а для хранения имени, фамилии и email — `inetOrgPerson`.

Создайте файл `adduser.ldif`:

```bash
sudo nano adduser.ldif
```

Заполните его, подставив свои значения. Обратите внимание, что DN пользователя состоит из `uid`, `ou=people` и вашего доменного суффикса.

```ldif
dn: uid=john,ou=people,dc=srv,dc=unlix.ru
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: John Doe
sn: Doe
uid: john
uidNumber: 1001
gidNumber: 1001
homeDirectory: /home/john
loginShell: /bin/bash
userPassword: {CRYPT}зашифрованный_пароль
```

#### Как получить зашифрованный пароль?

Так как хранить пароль в открытом виде небезопасно, его нужно зашифровать. Используйте утилиту `slappasswd`:

```bash
slappasswd
```

Команда запросит пароль дважды и выдаст строку, начинающуюся с `{CRYPT}` или `{SSHA}`. Скопируйте эту строку целиком и вставьте её в поле `userPassword` в файле `adduser.ldif`, заменив `зашифрованный_пароль`.

**Важно:** Значения `uidNumber` и `gidNumber` должны быть уникальными для каждого пользователя в пределах вашего каталога. `gidNumber` будет соответствовать основной группе пользователя (ее мы создадим далее).

Добавьте пользователя в LDAP:

```bash
sudo ldapadd -x -D cn=admin,dc=srv,dc=unlix.ru -W -f adduser.ldif
```

### 3. Добавление группы (posixGroup)

Пользователям Linux необходима группа. Группа в LDAP описывается объектным классом `posixGroup`.

Создайте файл `addgroup.ldif`:

```bash
sudo nano addgroup.ldif
```

Содержимое файла (замените `john` и `1001` на ваши значения):

```ldif
dn: cn=john,ou=groups,dc=srv,dc=unlix.ru
objectClass: posixGroup
cn: john
gidNumber: 1001
memberUid: john
```

*   `cn`: имя группы. Обычно совпадает с именем пользователя для его основной группы.
*   `gidNumber`: должен совпадать со значением `gidNumber` из записи пользователя.
*   `memberUid`: указывает, какой пользователь входит в эту группу.

Добавьте группу:

```bash
sudo ldapadd -x -D cn=admin,dc=srv,dc=unlix.ru -W -f addgroup.ldif
```

### Проверка результата

Чтобы убедиться, что пользователь и группа успешно добавлены, выполните поиск в каталоге:

```bash
ldapsearch -x -b "dc=srv,dc=unlix.ru" "(uid=john)"
```

Эта команда покажет все атрибуты созданной вами записи пользователя `john`.
