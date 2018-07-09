Настройка EMC Networker - RMAN
==============================

Резервное копирование RMAN
==========================

<pre style="font-size:80%">
RUN {
ALLOCATE CHANNEL ch1 DEVICE TYPE '<b>SBT_TAPE</b>';
SEND DEVICE TYPE 'SBT_TAPE' 'NSR_ENV=(NSR_SERVER=<b>backup-server.com</b>, NSR_SAVESET_RETENTION=<b>2days</b>)';

BACKUP FULL DATABASE TAG "FULL_DATABASE_DATAFILES";

SQL 'ALTER SYSTEM ARCHIVE LOG CURRENT';

BACKUP ARCHIVELOG ALL TAG "FULL_DATABASE_ARCHIVELOGS";
BACKUP CURRENT CONTROLFILE TAG "FULL_DATABASE_CONTROLFILE";
BACKUP SPFILE TAG "FULL_DATABASE_SPFILE";

RELEASE CHANNEL ch1;
}
</pre>


Восстановление RMAN
=======================

Поиск соответствующей резервной копии (Emc Networker)
-----------------------------------------------------

<pre style="font-size:80%">
[oracle@host]$ mminfo -c 'orac-client.com' -s  'backup-server.com' -o tR -r "name,ssid,level,fragsize,savetime(25),nsavetime"
 name                          ssid         lvl   size      date     time        save time
<b>RMAN:<b style="color: #080">c-367044326-20180706-03</b></b>   3846112434 manual 7169 KB    06.07.2018 10:57:03 1530856623
RMAN:ARCDB_ebt7aj3t_1_1        3862889646 manual 257 KB     06.07.2018 10:57:02 1530856622
RMAN:ARCDB_eat7aj3p_1_1        3879666858 manual 7169 KB    06.07.2018 10:56:58 1530856618
RMAN:ARCDB_e9t7aj04_1_1        3896443956 manual 3029 MB    06.07.2018 10:55:00 1530856500
RMAN:ARCDB_e8t7aiv0_1_1        3913221163 manual 2817 KB    06.07.2018 10:54:25 1530856465
</pre>

### Рашифровка

* `RMAN:c-367044326-20180706-03` - имя РК
* `367044326` - DBID

Восстановление SPFILE
---------------------

Перполагается база выключена

<pre style="font-size:80%">
[oracle@arc-db lib]$ rman target /
Recovery Manager: Release 10.2.0.1.0 - Production on Fri Jul 6 11:54:01 2018
Copyright (c) 1982, 2005, Oracle.  All rights reserved.
connected to target database (not started)

RMAN> SET DBID <b>367044326</b>; <i style="color:gray">Значение автоматически не копируется и должно быть взято из внешено источника (mminfo)</i>
executing command: SET DBID

RMAN> startup nomount;
Oracle instance started

Total System Global Area    1174405120 bytes

Fixed Size                     1219040 bytes
Variable Size                184550944 bytes
Database Buffers             973078528 bytes
Redo Buffers                  15556608 bytes

RMAN> run {
2> allocate channel ch1 type 'SBT_TAPE';
3> send device type 'SBT_TAPE' 'NSR_ENV=(NSR_SERVER=<b>backup-server.com</b>)'; <i style="color:gray">Указываем адрес EMC Networker сервера</i>
4> restore spfile to '<b>/home/oracle/test/spfileSID.ora</b>' from '<b>c-367044326-20180706-03</b>'; <i style="color:gray">Куда и откуда (c-367044326-20180706-03 - из mminfo) восстановить SPFILE</i>
5> release channel ch1;
6> }

using target database control file instead of recovery catalog
allocated channel: ch1
channel ch1: sid=156 devtype=SBT_TAPE
channel ch1: NMDA Oracle v9.2.1.1

sent command to channel: ch1

Starting restore at 06-JUL-18

channel ch1: autobackup found: c-367044326-20180706-03
channel ch1: SPFILE restore from autobackup complete
Finished restore at 06-JUL-18

released channel: ch1
</pre>

Восстановление CONTROLFILE
--------------------------

<pre style="font-size:80%">
RMAN> run {
2> allocate channel ch1 type 'SBT_TAPE';
3> send device type 'SBT_TAPE' 'NSR_ENV=(NSR_SERVER=<b>backup-server.com</b>)'; <i style="color:gray">Указываем адрес EMC Networker сервера</i>
4> restore controlfile from  'c-367044326-20180706-03'; <i style="color:gray">откуда (c-367044326-20180706-03 - из mminfo) восстановить CONTROLFILE</i>
5> }

allocated channel: ch1
channel ch1: sid=156 devtype=SBT_TAPE
channel ch1: NMDA Oracle v9.2.1.1

sent command to channel: ch1

Starting restore at 06-JUL-18

channel ch1: restoring control file
channel ch1: restore complete, elapsed time: 00:00:25
output filename=/oracle/oradata/db/control01.ctl
output filename=/oracle/oradata/db/control02.ctl
output filename=/oracle/oradata/db/control03.ctl
Finished restore at 06-JUL-18
released channel: ch1
</pre>

Просмотр истории backup-ов
--------------------------

<pre style="font-size:80%">
RMAN> alter database mount; <i style="color:gray">В CONTROLFILE содержиться история РК, читать CONTROLFILE можно в режиме MOUNT и выше</i>

database mounted

RMAN> list backup;

List of Backup Sets
===================

BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
30892   Incr 0  20.25M     SBT_TAPE    00:00:47     22-JUN-18      
        BP Key: 30892   Status: AVAILABLE  Compressed: NO  Tag: HOT_DB_BK_LEVEL0
        Handle: bk_30986_1_979465111   Media: 
  List of Datafiles in backup set 30892
  File LV Type Ckp SCN    Ckp Time  Name
  ---- -- ---- ---------- --------- ----

BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
31087   <b>Full</b>    2.89G      <b>SBT_TAPE</b>    00:03:20     <b>06-JUL-18</b>      
<i style="color:gray">31087 - номер РК</i>
		<i style="color:gray">Full - создана полная копия</i>
		                   <i style="color:gray">SBT_TAPE - копия на ленте</i>
						                            <i style="color:gray">06-JUL-18 - дата создания</i>
        BP Key: 31087   Status: AVAILABLE  Compressed: NO  Tag: <b>FULL_DATABASE_DATAFILES</b>
		                                                        <i style="color:gray">FULL_DATABASE_DATAFILES метка указанная при создании</i>
        Handle: <b>edt7b0jp_1_1</b>   Media: 
		        <i style="color:gray">edt7b0jp_1_1 - указывает на внутренне имя файла резервной копии</i>
  List of Datafiles in backup set 31087
  File LV Type Ckp SCN    Ckp Time  Name
  ---- -- ---- ---------- --------- ----
  1       Full 317349608711 06-JUL-18 /oracle/oradata/arcdb/system01.dbf
  2       Full 317349608711 06-JUL-18 /oracle/oradata/arcdb/undotbs01.dbf
  3       Full 317349608711 06-JUL-18 /oracle/oradata/arcdb/sysaux01.dbf
  4       Full 317349608711 06-JUL-18 /oracle/oradata/arcdb/users01.dbf
  5       Full 317349608711 06-JUL-18 /oracle/oradata/arcdb/arc01.dbf
  6       Full 317349608711 06-JUL-18 /oracle/oradata/arcdb/arc_ntmk_01.dbf
  7       Full 317349608711 06-JUL-18 /oracle/oradata/arcdb/hr01.dbf

BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
31088   Full    7.00M      SBT_TAPE    00:00:01     06-JUL-18      
        BP Key: 31088   Status: AVAILABLE  Compressed: NO  Tag: TAG20180706T145046
        Handle: c-367044326-20180706-04   Media: 
  Control File Included: Ckp SCN: 317349609084   Ckp time: 06-JUL-18
  SPFILE Included: Modification time: 05-JUL-18

BS Key  Size       Device Type Elapsed Time Completion Time
------- ---------- ----------- ------------ ---------------
31089   20.50M     SBT_TAPE    00:00:03     06-JUL-18      
        BP Key: 31089   Status: AVAILABLE  Compressed: NO  Tag: FULL_DATABASE_ARCHIVELOGS
        Handle: eft7b0qb_1_1   Media: 

  <b>List of Archived Logs</b> in backup set 31089
  <i style="color:gray">Резервная копия содержит архивные логи</i>
  Thrd Seq     Low SCN    Low Time  Next SCN   Next Time
  ---- ------- ---------- --------- ---------- ---------
  1    13612   317349584617 06-JUL-18 317349588360 06-JUL-18
  1    13613   317349588360 06-JUL-18 317349588434 06-JUL-18
  1    13614   317349588434 06-JUL-18 317349609096 06-JUL-18
  1    13615   <b>317349609096</b> 06-JUL-18 317349609101 06-JUL-18
               <i style="color:gray">317349609096 - номер SCN до которого будет произведенно восстановление</i>

BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
31090   Full    7.00M      SBT_TAPE    00:00:01     06-JUL-18      
        BP Key: 31090   Status: AVAILABLE  Compressed: NO  Tag: FULL_DATABASE_CONTROLFILE
        Handle: egt7b0qh_1_1   Media: 
  Control File Included: Ckp SCN: 317349609113   Ckp time: 06-JUL-18

BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
31091   Full    256.00K    SBT_TAPE    00:00:00     06-JUL-18      
        BP Key: 31091   Status: AVAILABLE  Compressed: NO  Tag: FULL_DATABASE_SPFILE
        Handle: eht7b0qi_1_1   Media: 
  SPFILE Included: Modification time: 05-JUL-18

BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
31092   Full    6.89M      DISK        00:00:00     06-JUL-18      
        BP Key: 31092   Status: AVAILABLE  Compressed: NO  Tag: TAG20180706T124452
        Piece Name: /oracle/flash_recovery_area/ARCDB/autobackup/2018_07_06/o1_mf_s_980772292_fmy7hnd5_.bkp
  Control File Included: Ckp SCN: 317349590649   Ckp time: 06-JUL-18
  SPFILE Included: Modification time: 06-JUL-18
</pre>

Восстановление базы на момент времени/SCN
-----------------------------------------

<pre style="font-size: 80%">
RMAN> run {
2> allocate channel ch1 type 'SBT_TAPE';
3> send device type 'SBT_TAPE' 'NSR_ENV=(NSR_SERVER=<b>backup-server.com</b>)'; <i style="color:gray">Указываем адрес EMC Networker сервера</i>
4> set until scn = <b>317349609096</b>; <i style="color:gray">До кого номера SCN восстанавливаем базу</i>
5> restore database;
6> recover database;
7> alter database open resetlogs;
8> }

allocated channel: ch1
channel ch1: sid=156 devtype=SBT_TAPE
channel ch1: NMDA Oracle v9.2.1.1

sent command to channel: ch1

executing command: SET until clause

Starting restore at 06-JUL-18

channel ch1: starting datafile backupset restore
channel ch1: specifying datafile(s) to restore from backup set
restoring datafile 00001 to /oracle/oradata/arcdb/system01.dbf
restoring datafile 00002 to /oracle/oradata/arcdb/undotbs01.dbf
restoring datafile 00003 to /oracle/oradata/arcdb/sysaux01.dbf
restoring datafile 00004 to /oracle/oradata/arcdb/users01.dbf
restoring datafile 00005 to /oracle/oradata/arcdb/arc01.dbf
restoring datafile 00006 to /oracle/oradata/arcdb/arc_ntmk_01.dbf
restoring datafile 00007 to /oracle/oradata/arcdb/hr01.dbf
channel ch1: reading from backup piece ARCDB_e9t7aj04_1_1
channel ch1: restored backup piece 1
piece handle=ARCDB_e9t7aj04_1_1 tag=TAG20180706T105500
channel ch1: restore complete, elapsed time: 00:00:56
Finished restore at 06-JUL-18

Starting recover at 06-JUL-18

starting media recovery

archive log thread 1 sequence 13609 is already on disk as file /oracle/flash_recovery_area/ARCDB/archivelog/2018_07_06/o1_mf_1_13609_fmy7hgmv_.arc
archive log thread 1 sequence 13613 is already on disk as file /oracle/flash_recovery_area/ARCDB/archivelog/2018_07_06/o1_mf_1_13613_fmy7366o_.arc
archive log thread 1 sequence 1 is already on disk as file /oracle/oradata/arcdb/redo02.log
archive log filename=/oracle/flash_recovery_area/ARCDB/archivelog/2018_07_06/o1_mf_1_13613_fmy7366o_.arc thread=1 sequence=13613
archive log filename=/oracle/oradata/arcdb/redo02.log thread=1 sequence=1
media recovery complete, elapsed time: 00:00:04
Finished recover at 06-JUL-18

database opened
released channel: ch1
</pre>