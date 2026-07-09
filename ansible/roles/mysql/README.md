# MySQL

https://dev.mysql.com/doc/refman/8.4/en/linux-installation-apt-repo.html
https://endoflife.date/mysql


## Download APT Repo
https://dev.mysql.com/downloads/repo/apt/

Download APT Repo package file
https://dev.mysql.com/get/mysql-apt-config_0.8.34-1_all.deb
https://repo.mysql.com//mysql-apt-config_0.8.34-1_all.deb
(mysql-apt-config_0.8.34-1_all.deb)	MD5: ab23dec6619a7e65e8462646936ee49f

Download Signature:
https://dev.mysql.com/downloads/gpg/?file=mysql-apt-config_0.8.34-1_all.deb&p=37
```
-----BEGIN PGP SIGNATURE-----

iQIzBAABCAAdFiEEvKQ0F8O0hd0SjsbUt7O3iKjTeFwFAmftasMACgkQt7O3iKjT
eFziBA//YY2RhH6PIryJgiT8EwpMyR10Tshs0PHHnBen5RtDGCaoTogDJrwKcZOH
oWRt5qoEGVvglYOd8NlmBMb0ib5PxzowYNYBGx35z1do0E/MbATMYx3sHCMIKCxv
RC4MJ/LLA56iZhqiFBJyMfRJPeiZduDvaV4tRbgSa/S7CKjE0fUsUXeZIG2Mik4j
1jSAJWmmCr8lzsjPeTpNOoaF/sxpuyi2OJHr7eZehWZhD/OMeNDzZyjbJKxK8l2+
Gx+eg5nw/b9aYWFQiARClSxPaP+ApkQ/mUEWudA08GsbpMMQEazF9vQZcQ/BCuMz
m6RyeG2hBOsGXVpcHAy2g1fMABX9xR7AAKf9fu28+KbX7V8TL0+G0CXZ372+Crv1
m2JAjqsVfuuMuGu2vUncp6xdp/g1p+4AaTIYjbn4dgdscfKGiTILEHKfIY9MSkKc
UlL22zOBkC9aoq47A5+XW4ctZ1Jyq2etXdxuj5tSDHU8EwOCFAZRbkgoIUwOUOPr
ZGWORgax81rzVzx8StZXKtnK6p09qjhH/7bp6rw53DGGmbYsj9MZAMqy/i7HKZbA
IZd1/yFaUnjnS5oMJZDaeKBhS+ijXrhCrwpu6IsiDGh/Nh4dGiVwI5E0zm4cqF68
yaAvNKKBKRUr85OtGPf+Z6ep3sTXe8SyZHqn0heYVcoyUIg73QQ=
=lwua
-----END PGP SIGNATURE-----
```

Download GPG Key
https://dev.mysql.com/doc/refman/8.4/en/checking-gpg-signature.html

A8D3785C

```bash
gpg --recv-keys A8D3785C
apt-key adv --keyserver pgp.mit.edu --recv-keys A8D3785C
gpg --homedir /tmp --no-default-keyring --keyring /var/cache/ansible/mysql/mysql-keyring.gpg --keyserver pgp.mit.edu --recv-keys A8D3785C
```

pgp.mit.edu

## Manually Setup APT Repo
https://dev.mysql.com/doc/refman/8.4/en/linux-installation-apt-repo.html#repo-qg-apt-repo-manual-setup

/etc/apt/sources.list.d/mysql.list
```
deb http://repo.mysql.com/apt/{debian|ubuntu}/ {bookworm|jammy} {mysql-tools|mysql-8.4-lts|mysql-8.0}
```

## Verify Downloads
https://dev.mysql.com/doc/refman/8.4/en/verifying-package-integrity.html

## Use credential file

```bash
mysql --defaults-file=~/.my.cnf -e 'show databases'
mysql --defaults-file=/data/root/.gitea.my.cnf -e 'show databases'
```

```bash
# Extract config values
MERGED_CONFIG_FILES=$(cat "/etc/app/config_default.json" "/etc/app/config_override.json" | jq -s 'reduce .[] as $item ({}; . *= $item)')

MYSQL_USER=$(echo -E "${CONFIG_FILES}" | jq -r .DB_USER)
MYSQL_PASS=$(echo -E "${MERGED_CONFIG_FILES}" | jq -r .DB_PASS)
MYSQL_HOST=$(echo -E "${MERGED_CONFIG_FILES}" | jq -r .DB_HOST)
MYSQL_DB=$(echo -E "${MERGED_CONFIG_FILES}" | jq -r .DB_NAME)

MYSQL_DEFAULTS_FILE=$(cat <<CONTENT_HEREDOC
[client]
user="${MYSQL_USER}"
password="${MYSQL_PASS}"
host="${MYSQL_HOST}"
database="${MYSQL_DB}"
CONTENT_HEREDOC
)

# Display databases
mysql --defaults-file=<(echo -E "${MYSQL_DEFAULTS_FILE}") -e 'show databases' --skip-column-names --batch
```
