# DB et DB proxy

Documentation archivée le 1er mai 2023.

La BD est hébergée dans un stateful set. La BD est prévue pour être accédée par une socket Unix directement, ou par un tunnel SSL : d'où les deux services distincts, même s'il est vrai qu'il est moins pertinent d'avoir un service DB puisque ce service n'est pas prévu pour être accédé par le réseau.

Une conséquence de cette architecture est qu'en PHP, si on configure correctement le chemin par défaut des sockets, on peut utiliser le nom spécial `localhost` pour passer par la socket Unix et accéder, directement ou indirectement à la BD

## Initialisation
Un container d'initialisation est chargé d'initialiser l'ensemble des ressources (fichiers et base de données). Il pourra être envisagé que ce container puisse également procéder aux mises à jours des données (c'est à dire, effectuer les modifications qui impactent la BD et les fichiers propres à chaque paroisse - les images étant mises à jour en les reconstruisant).

Il est à noter que ce stateful set doit être opérationnel pour que les autres éléments puissent s'activer (synchronisation via containers d'initialisation).

## Configuration BD
Selon la documentation de MySQL <https://dev.mysql.com/doc/refman/8.0/en/memory-use.html>, MySQL est prévu pour tenir dans environ 512M de RAM. Ceci est confirmé en pratique par l'utilisation du performance_schema :

``` sql 

```mysql> SELECT SUBSTRING_INDEX(event_name,'/',2) AS
    ->        code_area, FORMAT_BYTES(SUM(current_alloc))
    ->        AS current_alloc
    ->        FROM sys.x$memory_global_by_current_bytes
    ->        GROUP BY SUBSTRING_INDEX(event_name,'/',2)
    ->        ORDER BY SUM(current_alloc) DESC;
+---------------------------+---------------+
| code_area                 | current_alloc |
+---------------------------+---------------+
| memory/performance_schema | 214.94 MiB    |
| memory/innodb             | 196.09 MiB    |
| memory/sql                | 9.39 MiB      |
| memory/mysys              | 8.58 MiB      |
| memory/temptable          | 1.00 MiB      |
| memory/mysqld_openssl     | 808.83 KiB    |
| memory/mysqlx             | 3.38 KiB      |
| memory/vio                | 1.16 KiB      |
| memory/myisam             |  728 bytes    |
| memory/csv                |  120 bytes    |
| memory/blackhole          |  120 bytes    |
+---------------------------+---------------+
11 rows in set (0.02 sec)
mysql> select * from sys.memory_global_total;
+-----------------+
| total_allocated |
+-----------------+
| 448.20 MiB      |
+-----------------+
1 row in set (0.01 sec)

```

``` bash

root@b64de810e1b6:/# ps -ua
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   4140  3368 pts/0    Ss   13:45   0:00 bash
mysql         12  1.1 10.0 2228664 389164 pts/0  Sl+  13:45   0:02 mysqld
root         155  0.0  0.0   4136  3392 pts/1    Ss   13:46   0:00 bash
root         226  0.0  0.0   6420  1600 pts/1    R+   13:49   0:00 ps -ua

```

On constate en particulier l'utilisation de 215MB de ram pour performance\_schema, alors que cette partie de la BD ne va pas réellement être utilisée. Du coup, il est envisageable de désactiver performance\_schema. Après désactivation, on ne peut plus utiliser la requête SQL, mais on peut continuer à utiliser `ps -ua` :
``` sql
mysql> SELECT SUBSTRING_INDEX(event_name,'/',2) AS
    ->        code_area, FORMAT_BYTES(SUM(current_alloc))
    ->        AS current_alloc
    ->        FROM sys.x$memory_global_by_current_bytes
    ->        GROUP BY SUBSTRING_INDEX(event_name,'/',2)
    ->        ORDER BY SUM(current_alloc) DESC;
Empty set (0.01 sec)
mysql> select * from sys.memory_global_total;
+-----------------+
| total_allocated |
+-----------------+
| NULL            |
+-----------------+
1 row in set (0.02 sec)


```

```
root@b64de810e1b6:/etc/mysql/conf.d# ps -ua
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   4140  3368 pts/0    Ss   13:45   0:00 bash
root         155  0.0  0.0   4136  3400 pts/1    Ss   13:46   0:00 bash
mysql        461  2.5  4.0 1940420 156368 pts/0  Sl+  13:52   0:01 mysqld
root         545  0.0  0.0   6420  1640 pts/1    R+   13:52   0:00 ps -ua
```
Comme j'ai fait les opérations dans un même container, on peut voir que plus de la moitié de la RAM n'est plus utilisée après avoir mis `performance_schema = 0`.

L'autre modification effectuée est de positionner `skip_networking = 1` de sorte que le port réseau ne soit plus écouté, étant qu'un accès direct est à proscrire sans support chiffré configuré.

## Proxy 
Le proxy est en fait une utilisation d'un tunnel socat vers la socket de mysql. On prend en fait soin de positionner le répertoire de la socket dans un `emptyDir` stocké en mémoire et déclaré dans les specs, et on partage ce volume entre le container de db et la terminaison serveur du proxy : c'est une application du design pattern sidecar.

