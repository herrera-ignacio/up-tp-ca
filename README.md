# TP Integrador — Computación Aplicada

Universidad de Palermo · Ingeniería en Informática

Trabajo práctico integrador individual: puesta en marcha de un servidor GNU/Linux Debian 12
con servicios web, base de datos, almacenamiento dedicado y backups automatizados.

## Integrante

| Apellido, Nombre | Legajo |
|---|---|
| Herrera, Ignacio | 0119111 |

## Entorno

| Parámetro | Valor |
|---|---|
| Hipervisor | Oracle VirtualBox, adaptador en modo puente |
| Sistema operativo | Debian 12.15 (bookworm), actualizado desde Debian 11 (bullseye) |
| Hostname | `TPServer` |
| Dirección IP | `192.168.100.200/24`, gateway `192.168.100.1`, estática en `/etc/network/interfaces` |
| Interfaz de red | `enp0s3` |

### Servicios

| Servicio | Detalle |
|---|---|
| OpenSSH | acceso de `root` por clave pública/privada, `PermitRootLogin prohibit-password` |
| Apache 2.4 + PHP 8.2 | `DocumentRoot` en `/www_dir`, sirve `index.php` y `logo.png` |
| MariaDB 10.11 | base `ingenieria` cargada desde `db.sql`, usuario de la aplicación `lcars` |
| cron | tareas de backup automatizadas |

### Almacenamiento

Disco adicional de 10 GB con dos particiones estándar (tipo 83), montadas por UUID
en `/etc/fstab` para que persistan al reiniciar.

| Partición | Tamaño | Punto de montaje |
|---|---|---|
| `/dev/sdc1` | 3 GB | `/www_dir` |
| `/dev/sdc2` | 6 GB | `/backup_dir` |

El archivo `/opt/particion` contiene el volcado de `/proc/partitions`, que es un
sistema de archivos virtual generado en memoria y se pierde al apagar la máquina.

### Backups

El script `/opt/scripts/backup_full.sh` recibe origen y destino como argumentos,
valida que el origen exista y que el destino sea una partición montada antes de
ejecutarse, e incluye una opción `-help`. Genera archivos con la fecha en formato ANSI:
`<nombre_origen>_bkp_YYYYMMDD.tar.gz`.

Tareas programadas en el crontab de `root`:

```cron
0 0 * * *      /opt/scripts/backup_full.sh /var/log /backup_dir >> /var/log/backup_full.log 2>&1
0 23 * * 1,3,5 /opt/scripts/backup_full.sh /www_dir /backup_dir >> /var/log/backup_full.log 2>&1
```

> El enunciado indica `/var/logs`, pero en Debian el directorio es `/var/log`. También
> indica el formato `YYYMMDD`, pero su ejemplo (`log_bkp_20240302.tar.gz`) usa `YYYYMMDD`.

## Contenido del repositorio

| Archivo | Directorio de origen |
|---|---|
| `root.tar.gz` | `/root` |
| `etc.tar.gz` | `/etc` |
| `opt.tar.gz` | `/opt` |
| `www_dir.tar.gz` | `/www_dir` |
| `backup_dir.tar.gz` | `/backup_dir` |
| `var.tar.gz.partaa` … `var.tar.gz.partag` | `/var`, dividido en 7 partes de hasta 10 MB |

## Reconstrucción

### `/var`

Se subió dividido en partes, como pide la consigna. GitHub rechaza archivos de más de
100 MB y advierte a partir de 50 MB; con partes de 10 MB ninguna se acerca a esos
límites. Las partes se concatenan en orden alfabético (`partaa`, `partab`, …), que es
el que `*` expande por defecto:

```bash
cat var.tar.gz.part* > var.tar.gz
```

Conviene verificar la integridad antes de extraer. `gzip -t` falla si falta alguna
parte o si quedaron desordenadas:

```bash
gzip -t var.tar.gz && echo "OK"
```

```bash
tar -xzpf var.tar.gz
```

### El resto de los directorios

```bash
tar -xzpf root.tar.gz
tar -xzpf etc.tar.gz
tar -xzpf opt.tar.gz
tar -xzpf www_dir.tar.gz
tar -xzpf backup_dir.tar.gz
```

Todos los archivos guardan rutas relativas (`etc/…` y no `/etc/…`), así que se
extraen en el directorio actual. Para restaurarlos sobre un sistema se agrega `-C /`
y se ejecuta como `root`, que es lo que preserva permisos y propietarios:

```bash
tar -xzpf etc.tar.gz -C /
```

Para inspeccionar el contenido sin extraer nada:

```bash
tar -tzf etc.tar.gz | head
```

## Cómo se generó esta entrega

Los comprimidos se generaron en la VM como `root`, desde un directorio de trabajo
fuera de todos los directorios a comprimir:

```bash
mkdir -p /srv/entrega && cd /srv/entrega

tar -czpf root.tar.gz       -C / root
tar -czpf etc.tar.gz        -C / etc
tar -czpf opt.tar.gz        -C / opt
tar -czpf www_dir.tar.gz    -C / www_dir
tar -czpf backup_dir.tar.gz -C / backup_dir

tar -czpf - -C / var | split -b 10M - var.tar.gz.part
```

Las opciones `-c` (crear), `-z` (gzip), `-p` (preservar permisos y propietarios) y
`-f` (archivo de salida) son las mismas en todos los casos. El `-C /` hace que las
rutas queden relativas. En `/var` el guion como archivo de salida manda el tar por
*stdout* directo al `split`, sin escribir el archivo completo en disco.

Antes de comprimir `/var` se limpiaron los paquetes descargados por APT durante las
actualizaciones, que no forman parte de la configuración del servidor.

Los archivos se descargaron a la máquina física con `scp`, usando la misma clave
privada configurada en el punto 2.2:

```bash
scp -i clave_privada.txt "root@192.168.100.200:/srv/entrega/*" .
```

Y se subieron desde ahí:

```bash
git init
git remote add origin git@github.com:herrera-ignacio/up-tp-ca.git
git add -A
git commit -m "Archivos de la VM"
git push -u origin main
```

Se subió desde la máquina física y no desde la VM para no generar credenciales
propias dentro del servidor: una clave SSH creada en `/root/.ssh` para autenticarse
contra GitHub habría terminado incluida en `root.tar.gz`.

## Verificación

### Checklist de la VM

Conviene reiniciar la VM antes de recorrerlo: lo que no sobrevive a un reinicio no
está terminado.

```bash
hostnamectl                                  # TPServer
cat /etc/debian_version                      # 12.x
ip a show enp0s3                             # 192.168.100.200/24, sin "dynamic"
grep -E "address|netmask|gateway" /etc/network/interfaces
systemctl is-active ssh apache2 mariadb cron # active x4
df -h | grep -E "www_dir|backup_dir"         # ambas montadas tras el reinicio
cat /opt/particion                           # sdc1 y sdc2 listadas
ls -lh /backup_dir                           # backups con fecha en el nombre
crontab -l                                   # las dos tareas de backup
mysql -e "SHOW DATABASES;"                   # la base ingenieria
```

Desde la máquina física, no desde la VM:

```bash
ping 192.168.100.200
ssh -i clave_privada.txt root@192.168.100.200
curl http://192.168.100.200/
```

### Integridad de los comprimidos

Cada `.tar.gz` se valida en dos pasos: `gzip -t` comprueba que la compresión esté
completa y `tar -tzf` recorre el índice del archivo sin extraer nada.

```bash
for f in *.tar.gz; do
    gzip -t "$f" && tar -tzf "$f" > /dev/null && echo "$f OK"
done
```

`/var` se valida después de unir las partes. Si falta una parte o quedaron
desordenadas, `gzip -t` falla:

```bash
cat var.tar.gz.part* > /tmp/var_test.tar.gz
gzip -t /tmp/var_test.tar.gz && echo "var OK"
tar -tzf /tmp/var_test.tar.gz | wc -l
rm /tmp/var_test.tar.gz
```

En la VM, la cantidad de entradas se puede comparar con `find /var | wc -l`. No van a
coincidir exactamente, porque `tar` omite los sockets y `/var` cambia con el uso, pero
tienen que quedar cerca.

## Nota sobre el contenido

Por el alcance de la consigna, esta entrega incluye material sensible del sistema:
claves de host de SSH (`/etc/ssh/ssh_host_*_key`), hashes de contraseñas
(`/etc/shadow` y su copia en `/var/backups/`), la tabla de usuarios de MariaDB y el
historial de comandos de `root`. **El repositorio es público**: las credenciales
utilizadas son exclusivas de esta máquina virtual de laboratorio y no se reutilizan
en ningún otro sistema.
