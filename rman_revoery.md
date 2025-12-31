# Rman での PITR 手順

以下は FREE（CDB=FREE / PDB=FREEPDB1）を想定した、RMANバックアップからの PITR（Point-In-Time Recovery）手順です。
CDB全体のPITR と PDB単体のPITR は手順が違うため、両方を示します。


# ① CDB 全体の PITR（最も基本・確実）
1. 目標時点を決める

例：

日時指定：2025-12-30 23:10:00

SCN 指定（より正確）

以降は 日時指定例で説明します。

# 2. DB を停止して MOUNT
```
sqlplus / as sysdba
shutdown immediate;
startup mount;
exit;
```

# 3. RMAN で PITR 実行
bashで
```
rman target /
```
rmanで
```
RUN {
  SET UNTIL TIME "TO_DATE('2025-12-30 23:10:00','YYYY-MM-DD HH24:MI:SS')";
  RESTORE DATABASE;
  RECOVER DATABASE;
}
```

# 4. OPEN（RESETLOGS 必須）
rmanで
```
ALTER DATABASE OPEN RESETLOGS;
```

# 5. PDB を OPEN
sqlplusで
```
sqlplus / as sysdba
alter pluggable database all open;
```

# 6. 確認
sqlplusで
```
archive log list;
show pdbs;
```


# ② PDB 単体の PITR（FREEPDB1 のみ戻す）
👉 CDB 全体を止めずに、PDBだけ戻せるのが利点

# 1. PDB を CLOSE
sqlplus
```
sqlplus / as sysdba
alter pluggable database FREEPDB1 close immediate;
exit;
```

# 2. RMAN で PDB PITR
bashで
```
2. RMAN で PDB PITR
```
rmanで
```
RUN {
  SET UNTIL TIME "TO_DATE('2025-12-30 23:10:00','YYYY-MM-DD HH24:MI:SS')";
  RESTORE PLUGGABLE DATABASE FREEPDB1;
  RECOVER PLUGGABLE DATABASE FREEPDB1;
}
```

# 3. PDB を OPEN（RESETLOGS 相当）
sqlplusで
```
sqlplus / as sysdba
alter pluggable database FREEPDB1 open resetlogs;
```

# 4. 確認
sqlplus
```
show pdbs;
select current_scn from v$database;
```


# ③ SCN 指定での PITR（推奨）
ログや監査で 正確な点 が分かっている場合。

rmanで
```
RUN {
  SET UNTIL SCN 123456789;
  RESTORE DATABASE;
  RECOVER DATABASE;
}
```
（PDB も同様に RESTORE PLUGGABLE DATABASE）


# ④ PITR が失敗する典型原因

| 原因                    | 対処                  |
| --------------------- | ------------------- |
| ORA-01152 / ORA-01547 | RESETLOGS で OPEN    |
| ORA-00308             | アーカイブログ不足           |
| ORA-38760             | 指定時点が RESETLOGS より前 |
| ORA-19870             | バックアップ不足            |


# ⑤ 事前に必ずやるべき確認（重要）
rmanで
```
LIST BACKUP SUMMARY;
LIST ARCHIVELOG ALL;
```
sqlplusで
```
select resetlogs_time, resetlogs_change# from v$database;
```

# ⑥ 実務的アドバイス（重要）
PDB PITR を基本にする（影響範囲が小さい）
PITR 後は 必ず新しいフルバックアップを取る
RESETLOGS 後のバックアップは 以前と非互換 ( PITR後は既存のバックアップで他の時間を指定してPITRし直すことはできない ) 

# まとめ( 最短 ) 
CDB PITR
```
shutdown → mount → restore → recover → open resetlogs
```
PDB PITR
```
close pdb → restore pdb → recover pdb → open resetlogs
```
