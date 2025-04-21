# Control de Particionado MySQL ODBC - CUSTOS

_Template ODBC para crear Stored Procedures sobre la DB, particionar, mantener la creación de particiones a lo largo del tiempo y eliminarlas al alcanzar un período de validez._

## ¿En que casos necesito particionar? ❓
Cuando el Housekeeper no logra borrar los datos viejos a tiempo, es decir, no puede seguir el ritmo. Esto puede ocurrir cuando la base de datos comienza a tener un tamaño considerable, ya que el Housekeeper recorre registros antiguos tanto en las tablas de History como en las de Trends y borra esos registros seleccionados que ya excedieron el tiempo de almacenado.

![image](https://github.com/user-attachments/assets/c0f62540-c500-4e79-84bb-796c70279ba5)

Se puede ver en la captura, que el porcentaje de uso del housekeeper se mantiene al 100% constantemente y nunca termina de borrar registros antes de lo que debería ser su siguiente ejecución (a cada hora). Esto también lo podemos notar cuando comiencen a dispararse Triggers de advertencia sobre el uso del Housekeeper y también cuando nuestra DB crece exponencialmente.

[Extraido del Blog de Zabbix. Autor: Nathan Liefting](https://blog.zabbix.com/partitioning-a-zabbix-mysql-database-with-perl-or-stored-procedures/13531/?_gl=1*1ons68w*_gcl_au*MTA5MTc3OTk3My4xNzQwNDA0MjMx*_ga*Nzg5NjI4NzEyLjE2ODU0NzIxMjA.*_ga_1F6WJN99ZG*MTc0NTI0MzIwMy4xMDg4LjAuMTc0NTI0MzIwMy42MC4wLjA)

**Con el particionado propuesto, lograremos borrar bloques que contienen registros de un día entero, en lugar de eliminar registro por registro. Esto trae más ventajas que desventajas. Hay que tener en cuenta que no tendremos la posibilidad de establecer períodos de almacenamiento histórico por Item, sino que estableceremos un mismo período de almacenamiento para todos los Items.**

## Comenzando 🚀

_Esta es la versión del Template "Control de Particionado MySQL ODBC - CUSTOS" utilizado por CUSTOS Monitoring. Probado en las versiones de Zabbix 6.0 y 7.0._

Puedes ver nuestro [Webinar](https://www.youtube.com/watch?v=MVIaw_STErE) para ver el procedimiento y tener más información.

### Pre-requisitos 📋

* [Zabbix](https://www.zabbix.com/) **Versión 6.X a 7.X** - Solución de monitoreo OpenSource
* [Zabbix DB](https://www.zabbix.com/) - **MySQL / MariaDB** - Database de Zabbix
* Posibilidad de crear usuario SSH con permisos suficientes (o contar con uno)
* Posibilidad de crear usuario SQL con permisos suficientes (o contar con uno)

### Instalación🔧

_El siguiente instructivo permite seguir los pasos necesarios para particionar una DB desde cero (sin datos históricos)_

#### 1. Creación de usuario SSH

_Es el usuario con el cual ingresaremos mediante el Frontend de Zabbix para crear los Stored Procedures en la DB._

```
useradd custos
```

_Configuramos contraseña para dicho usuario_

```
passwd custos
```

#### 2. Instalamos Driver ODBC para MySQL 7 MariaDB

_Para poder ejecutar el mantenimiento de las particiones, obtener el tamaño de las mismas, etc._

```
dnf install mariadb-connector-odbc -y
```

#### 3. Creamos usuario en MySQL / MariaDB

_Usuario que será utilizado para conectarse por ODBC_

```
create user 'custos'@'127.0.0.1' identified by 'PASSWORD';
```

_Damos los permisos necesarios al usuario_

```
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, REFERENCES, INDEX, ALTER,
EXECUTE, CREATE ROUTINE, ALTER ROUTINE, CREATE TEMPORARY TABLES ON `zabbix`.* TO
`custos`@`127.0.0.1`;
```

#### 4. Particionado de la tabla auditlog (OPCIONAL)

_Si bien el particionado de cualquier tabla es opcional, en la versión de Zabbix 7.0 el auditlog almacena aún más información y puede llegar a tener un gran peso, al igual que una tabla de histórico._

_Recomendamos particionarla y para esto debemos realizar algunos pasos adicionales previos, con el objetivo de preparar la tabla. Para mayor información, visitar [Preparing auditlog table for partitioning](https://www.zabbix.com/documentation/7.0/en/manual/appendix/install/auditlog_primary_keys), documentación oficial de Zabbix_

#### Preparar la tabla auditlog para particionar

_Detenemos Zabbix Server_

```
systemctl stop zabbix-server
```

_Renombramos la tabla vieja_

```
RENAME TABLE auditlog TO auditlog_old
```

_Creamos la nueva tabla con los nuevos indices_

```
RENAME TABLE auditlog to auditlog_old
CREATE TABLE auditlog AS auditlog_old
ALTER TABLE auditlog ADD PRIMARY KEY (auditid,clock)
```
_Podemos copiar el contenido de auditlog_old a la nueva tabla auditlog_

```
INSERT INTO auditlog SELECT * FROM auditlog_old;
```

_Pero si pesan demasiado los datos, se recomienda migrarlos gradualmente segun el 'clock', donde copiaremos todos los registros menores a determinado TIMESTAMP_
```
INSERT INTO auditlog SELECT * FROM auditlog_old WHERE="clock < TIMESTAMP"
```

_Y borrar la tabla antigua auditlog_old_
```
DROP TABLE auditlog_old;
```

#### 5. Template "Control Particionado MySQL - ODBC

* _Descargar e Importar el Template en Zabbix 6.X - 7.X_
* _Crear el Host "Zabbix DB" (o sobre "Zabbix server") y vincular el Template_

 _Configurar (como mínimo) las siguientes Macros_

|MACRO|VALUE|Description
|----|-----------|-------|
|{$CTRLPART.DB.USER}|custos|`Usuario para conectarse a la DB`|
|{$CTRLPART.DB.PASSWORD}|password|`Password del usuario en la DB`|
|{$CTRLPART.SSH.USER}|custos|`User donde se encuentra la DB para conexión por SSH`|
|{$CTRLPART.SSH.PASSWORD}|password|`Password donde se encuentra la DB para conexión por SSH`|

#### 6. Ejecución

* _Habilitar el Item "Creación de Stored procedures y ejecutar. Deshabilitarlo luego de comprobar la ejecución"_
* _Ejecutar el Item "Ejecución Mantenimiento Particionado ODBC"_
* _Ejecutar Items "Días disponibles en…"_
* _Deshabilitar Housekeeper para tablas de histórico, trends y auditlog (en caso de haberla particionado)_

_Ahora ya tenemos las particiones creadas en nuestra Zabbix DB para las tablas que ocupan un mayor espacio._

### 7. Validación ✅

_Métricas que podemos comprobar para asegurarnos que el particionado fue aplicado con éxito_

#### Creación de Stored procedures

_El último valor debe tener el siguiente mensaje, indicando que no existieron errores_

```
Latest value:

OK
```

#### Ejecución Mantenimiento Particionado ODBC

_El último valor debe tener acciones indicando que se crearon nuevas particiones y se eliminaron particiones obsoletas (si corresponde)_

```
Latest value:

[{"schemaname":"zabbix","tablename":"history","action":"CREATE PARTITION","partitionname":"N/A"},{"schemaname":"zabbix","tablename":"history","action":" DROP PARTITION","partitionname":"N/A"},{"schemaname":"zabbix","tablename":"history_log","action":"CREATE PARTITION","partitionname":"N/A"},{"schemaname":"zabbix","tablename":"history_log","action":" DROP PARTITION","partitionname":"N/A"},{"schemaname":"zabbix","tablename":"history_str","action":"CREATE PARTITION","partitionname":"N/A"},{"schemaname":"zabbix","tablename":"history_str","action":" DROP PARTITION","partitionname":"N/A"},{"schemaname":"zabbix","tablename":"history_text","action":"CREATE PARTITION","partitionname":"N/A"},{"schemaname":"zabbix","tablename":"history_text","action":" DROP PARTITION","partitionname":"N/A"},{"schemaname":"zabbix","tablename":"history_uint","action":"CREATE PARTITION","partitionname":"N/A"},{"schemaname":"zabbix","tablename":"history_uint","action":" DROP PARTITION","partitionname":"N/A"},{"schemaname":"zabbix","tablename":"history_bin","action":"CREATE PARTITION","partitionname":"N/A"},{"schemaname":"zabbix","tablename":"history_bin","action":" DROP PARTITION","partitionname":"N/A"},{"schemaname":"zabbix","tablename":"trends","action":"CREATE PARTITION","partitionname":"N/A"},{"schemaname":"zabbix","tablename":"trends","action":" DROP PARTITION","partitionname":"N/A"},{"schemaname":"zabbix","tablename":"trends_uint","action":"CREATE PARTITION","partitionname":"N/A"},{"schemaname":"zabbix","tablename":"trends_uint","action":" DROP PARTITION","partitionname":"N/A"},{"schemaname":"zabbix","tablename":"auditlog","action":"CREATE PARTITION","partitionname":"N/A"},{"schemaname":"zabbix","tablename":"auditlog","action":" DROP PARTITION","partitionname":"N/A"}]
```

#### Días disponibles en...

_Si se crearon las particiones correctamente, este Item deberia devolver la cantidad de particiones creadas a futuro_

```
Latest value:

10
```

### Problemas frecuentes ⁉️

_En construcción_

## Despliegue en diferentes escenarios según la implementación de Zabbix que tengamos ️

#### Instalacion de Zabbix desde Cero

_Ya que no contaremos con datos históricos, no debemos preocuparnos por la copia de datos cosechados, y se puede implementar directamente la solución_

#### Instalación de Software Zabbix + Migración de datos desde otra DB

_Instalamos Zabbix en su nueva versión, exportamos el schema y definiciones de la DB antigua, y luego hacemos 'dumps' segmentados para importar el histórico_

#### Zabbix ya instalado

_En este caso, podremos usar la técnica de renombrar las tablas donde tenemos datos y crear nuevas tablas vacías las cuales serán particionadas y gradualmente copiaremos los datos entre las tablas_

## Macros usadas 📖

_Las siguientes son todas las Macros existentes en el Template_

|MACRO|VALUE|Description
|----|-------|-----------|
|{$CTRLPART.DB.CONSTR}|Driver=/usr/lib64/libmaodbc.so;Server=127.0.0.1;Port=3306;Database=zabbix|`Conection String a la DB`|
|{$CTRLPART.DB.DSN}|<SET_DSN>|`Nombre del DSN para conectarse`|
|{$CTRLPART.DB.HOST}|127.0.0.1|`Host donde se encuentra la DB para la creación de los stored procedures`|
|{$CTRLPART.DB.NAME}|ZABBIX|`Nombre de la base de datos en donde se mantienen las particiones`|
|{$CTRLPART.DB.PASSWORD}|<SET_DB_USER_PASSWORD>|`Password del usuario en la DB`|
|{$CTRLPART.DB.PORT}|3306|`Puerto para la conexión a la DB`|
|{$CTRLPART.DB.SIZE.CRIT}|50000000|`Tamaño de partición necesario para triggerear una alerta CRITICAL`|
|{$CTRLPART.DB.SIZE.WARN}|40000000|`Tamaño de partición necesario para triggerear una alerta AVERAGE`|
|{$CTRLPART.DB.USER}|<SET_DB_USER_USERNAME>|`Usuario para conectarse a la DB`|
|{$CTRLPART.DIASLEFT.HIGH}|2|`Dias limite para alarma HIGH`|
|{$CTRLPART.DIASLEFT.INFO}|9|`Dias limite alarma de INFO`|
|{$CTRLPART.DIASLEFT.WARNING}|7|`Dias limite alarma de WARNING`|
|{$CTRLPART.HISTORY.HOURSINT}|24|`Horas de intervalo de las tablas de history.  Acepta contexto con el nombre de la tabla`|
|{$CTRLPART.HISTORY.KEEPINT}|30|`Intervalos a conservar en las tablas de history. Acepta contexto con el nombre de la tabla`|
|{$CTRLPART.HISTORY.NEXTINT}|10|`Cantidad  de particiones a futuro para las tablas de history.  Acepta contexto con el nombre de la tabla`|
|{$CTRLPART.SSH.HOST}|127.0.0.1|`Host donde se encuentra la DB para conexión por SSH`|
|{$CTRLPART.SSH.PASSWORD}|<SET_SSH_USER_PASSWORD>|`Password donde se encuentra la DB para conexión por SSH`|
|{$CTRLPART.SSH.PORT}|22|`Puerto para conexión SSH al Host DB`|
|{$CTRLPART.SSH.USER}|<SET_SSH_USER_USERNAME>|`User donde se encuentra la DB para conexión por SSH`|
|{$CTRLPART.TRENDS.HOURSINT}|24|`Horas de intervalo de las tablas de trends.  Acepta contexto con el nombre de la tabla`|
|{$CTRLPART.TRENDS.KEEPINT}|365|`Intervalos a conservar en las tablas de trends. Acepta contexto con el nombre de la tabla`|
|{$CTRLPART.TRENDS.NEXTINT}|10|`Cantidad de particiones a futuro para las tablas de trends.  Acepta contexto con el nombre de la tabla`|

## Items 📖

_Las siguientes son todas las Macros existentes en el Template_

|NAME|KEY|TYPE|Description
|----|-------|-----------|-----------|
All Partition Sizes in MB|db.odbc.get[particionSizesMB,{$CTRLPART.DB.DSN},{$CTRLPART.DB.CONSTR}]|Database monitor|``|
Creación de Stored procedures|ssh.run[mysql create stored procedures,{$CTRLPART.SSH.HOST},{$CTRLPART.SSH.PORT}]|SSH agent|``|
Dias disponibles en auditlog|db.odbc.get[Dias que quedan para auditlog,{$CTRLPART.DB.DSN},{$CTRLPART.DB.CONSTR}]|Database monitor|``|
Dias disponibles en history|db.odbc.get[Dias que quedan para history,{$CTRLPART.DB.DSN},{$CTRLPART.DB.CONSTR}]|Database monitor|``|
Dias Disponibles en history_bin|db.odbc.get[Dias que quedan para history_bin,{$CTRLPART.DB.DSN},{$CTRLPART.DB.CONSTR}]|Database monitor|``|
Dias disponibles en history_log|db.odbc.get[Dias que quedan para history_log,{$CTRLPART.DB.DSN},{$CTRLPART.DB.CONSTR}]|Database monitor|``|
Dias disponibles en history_str|db.odbc.get[Dias que quedan para history_str,{$CTRLPART.DB.DSN},{$CTRLPART.DB.CONSTR}]|Database monitor|``|
Dias disponibles en history_text|db.odbc.get[Dias que quedan para history_text,{$CTRLPART.DB.DSN},{$CTRLPART.DB.CONSTR}]|Database monitor|``|
Dias disponibles en history_uint|db.odbc.get[Dias que quedan para history_uint,{$CTRLPART.DB.DSN},{$CTRLPART.DB.CONSTR}]|Database monitor|``|
Dias disponibles en trends|db.odbc.get[Dias que quedan para trends,{$CTRLPART.DB.DSN},{$CTRLPART.DB.CONSTR}]|Database monitor|``|
Dias disponibles en trends_uint|db.odbc.get[Dias que quedan para trends_uint,{$CTRLPART.DB.DSN},{$CTRLPART.DB.CONSTR}]|Database monitor|``|
Ejecución Mantenimiento Particionado ODBC|db.odbc.get[Ejecucion Particionado ODBC,{$CTRLPART.DB.DSN},{$CTRLPART.DB.CONSTR}]|Database monitor|``|
Total Size of DB|db.odbc.select[sizeInMB,{$CTRLPART.DB.DSN},{$CTRLPART.DB.CONSTR}]|``|

## Triggers 📖

_Las siguientes son todos los Triggers existentes en el Template_

|NAME|Description|SEVERITY
|----|-----------|----|
Cantidad de días en auditlog menores a {$CTRLPART.DIASLEFT.HIGH:auditlog}|``|High|
Cantidad de días en auditlog menores a {$CTRLPART.DIASLEFT.WARNING:auditlog}|``|Warning|
Cantidad de días en auditlog menores a {$CTRLPART.DIASLEFT.INFO:auditlog}|``|Information|

```
Se repiten los Triggers para las demás tablas
```

## Discovery Rules 📖

_Las siguientes son todas las Discoverys existentes en el Template_

|NAME|KEY|TYPE|Description
|----|-------|-----------|-----------|
Partition LLD|allPartitions.discovery|Dependent item|``|

## Webinar ✒️

Puedes encontrar más información sobre esta implementación en nuestro [Webinar](https://www.youtube.com/watch?v=MVIaw_STErE).
También puedes acceder al material utilizado [PDF](https://github.com/CUSTOSMonitoring/ParticionadoMySQL/blob/main/Webinars/Mantenimiento%20de%20particiones%20en%20MySQL%20-%20Webinar%2004-07-2024.pdf).

## Licencia 📄

_En construccion_

## Próximos Cursos Oficiales de Zabbix 📌

* [TRAINER](https://uy.linkedin.com/in/gustavoguido) - Zabbix Trainer, Director de CUSTOS Monitoring

* [ZCU](https://www.custosmonitoring.com/events/zcu-2025) - Certified User
* [ZCS](https://www.custosmonitoring.com/events/zcs-2025) - Certified Specialist
* [ZCP](https://www.custosmonitoring.com/events/zcp-2025) - Certified Professional
* [ZCE](https://www.custosmonitoring.com/events/zce-2025) - Certified Expert

## Te invitamos a seguirnos 📌

* [WEB](https://www.custosmonitoring.com) - Conoce las empresas que quienes confían en nosotros

* [INSTAGRAM](https://rometools.github.io/rome/) - Participa de nuestras actividades
* [LINKEDIN](https://uy.linkedin.com/company/custos-monitoring) - Enterate de eventos y participaciones
* [FACEBOOK](https://www.facebook.com/custos.monitoring) - Participa de nuestras actividades
* [YOUTUBE](https://www.youtube.com/@custosmonitoring) - Aprende con nosotros
* [TELEGRAM](https://t.me/+SczcNbS_mogJg1fU) - La mayor comunidad de Zabbix en Telegram en Español

* [CORREO](mailto:info@custos.uy) - Escríbenos si tienes alguna duda, sugerencia o proyecto
