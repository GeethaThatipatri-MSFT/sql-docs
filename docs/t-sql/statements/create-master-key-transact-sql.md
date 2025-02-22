---
title: "CREATE MASTER KEY (Transact-SQL)"
description: "CREATE MASTER KEY (Transact-SQL)"
ms.prod: sql
ms.prod_service: "database-engine, sql-database, synapse-analytics, pdw"
ms.technology: t-sql
ms.topic: reference
f1_keywords: 
  - "CREATE_MASTER_KEY_TSQL"
  - "CREATE MASTER KEY"
dev_langs: 
  - "TSQL"
helpviewer_keywords: 
  - "encryption [SQL Server], Database Master Key"
  - "database master key [SQL Server]"
  - "CREATE MASTER KEY statement"
  - "cryptography [SQL Server], Database Master Key"
  - "database master key [SQL Server], creating"
ms.assetid: 1710a305-1a4f-48ec-836c-11ffd0356d76
author: VanMSFT
ms.author: vanto
ms.reviewer: ""
ms.custom: ""
ms.date: "09/12/2019"
monikerRange: ">=aps-pdw-2016||=azuresqldb-current||=azure-sqldw-latest||>=sql-server-2016||>=sql-server-linux-2017||=azuresqldb-mi-current"
---

# CREATE MASTER KEY (Transact-SQL)

[!INCLUDE [sql-asdb-asdbmi-asa-pdw](../../includes/applies-to-version/sql-asdb-asdbmi-asa-pdw.md)]

Creates a database master key in the master database.

![Topic link icon](../../database-engine/configure-windows/media/topic-link.gif "Topic link icon") [Transact-SQL Syntax Conventions](../../t-sql/language-elements/transact-sql-syntax-conventions-transact-sql.md)

## Syntax

```syntaxsql
CREATE MASTER KEY [ ENCRYPTION BY PASSWORD ='password' ]
[ ; ]
```

[!INCLUDE[sql-server-tsql-previous-offline-documentation](../../includes/sql-server-tsql-previous-offline-documentation.md)]

## Arguments

PASSWORD ='*password*'
Is the password that is used to encrypt the master key in the database. *password* must meet the Windows password policy requirements of the computer that is running the instance of [!INCLUDE[ssNoVersion](../../includes/ssnoversion-md.md)]. *password* is optional in [!INCLUDE[ssSDS](../../includes/sssds-md.md)] and [!INCLUDE[ssSDW_md](../../includes/sssdw-md.md)].

## Remarks

The database master key is a symmetric key used to protect the private keys of certificates and asymmetric keys that are present in the database. When it is created, the master key is encrypted by using the AES_256 algorithm and a user-supplied password. In [!INCLUDE[ssKatmai](../../includes/sskatmai-md.md)] and [!INCLUDE[ssKilimanjaro](../../includes/sskilimanjaro-md.md)], the Triple DES algorithm is used. To enable the automatic decryption of the master key, a copy of the key is encrypted by using the service master key and stored in both the database and in master. Typically, the copy stored in master is silently updated whenever the master key is changed. This default can be changed by using the DROP ENCRYPTION BY SERVICE MASTER KEY option of [ALTER MASTER KEY](../../t-sql/statements/alter-master-key-transact-sql.md). A master key that is not encrypted by the service master key must be opened by using the [OPEN MASTER KEY](../../t-sql/statements/open-master-key-transact-sql.md) statement and a password.

The is_master_key_encrypted_by_server column of the sys.databases catalog view in master indicates whether the database master key is encrypted by the service master key.

Information about the database master key is visible in the sys.symmetric_keys catalog view.

For SQL Server and Parallel Data Warehouse, the master key is typically protected by the service master key and at least one password. In case of the database being physically moved to a different server (log shipping, restoring backup, etc.), the database will contain a copy of the master key encrypted by the original server service master key (unless this encryption was explicitly removed using `ALTER MASTER KEY DDL`), and a copy of it encrypted by each password specified during either `CREATE MASTER KEY` or subsequent `ALTER MASTER KEY DDL` operations. In order to recover the master key, and all the data encrypted using the master key as the root in the key hierarchy after the database has been moved, the user will have either use `OPEN MASTER KEY` statement using one of the passwords used to protect the master key, restore a backup of the master key, or restore a backup of the original service master key on the new server.

For [!INCLUDE[ssSDS](../../includes/sssds-md.md)] and [!INCLUDE[ssSDW_md](../../includes/sssdw-md.md)], the password protection is not considered to be a safety mechanism to prevent a data loss scenario in situations where the database may be moved from one server to another, as the Service Master Key protection on the Master Key is managed by Microsoft Azure platform. Therefore, the Master Key password is optional in [!INCLUDE[ssSDS](../../includes/sssds-md.md)] and [!INCLUDE[ssSDW_md](../../includes/sssdw-md.md)].

> [!IMPORTANT]
> You should back up the master key by using [BACKUP MASTER KEY](../../t-sql/statements/backup-master-key-transact-sql.md) and store the backup in a secure, off-site location.

The service master key and database master keys are protected by using the AES-256 algorithm.

## Permissions

Requires CONTROL permission on the database.

## Examples

Use the following example to create a database master key in the `master`database. The key is encrypted using the password `23987hxJ#KL95234nl0zBe`.

```sql
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '23987hxJ#KL95234nl0zBe';
GO
```

## See Also

- [sys.symmetric_keys &#40;Transact-SQL&#41;](../../relational-databases/system-catalog-views/sys-symmetric-keys-transact-sql.md)
- [sys.databases &#40;Transact-SQL&#41;](../../relational-databases/system-catalog-views/sys-databases-transact-sql.md)
- [OPEN MASTER KEY &#40;Transact-SQL&#41;](../../t-sql/statements/open-master-key-transact-sql.md)
- [ALTER MASTER KEY &#40;Transact-SQL&#41;](../../t-sql/statements/alter-master-key-transact-sql.md)
- [DROP MASTER KEY &#40;Transact-SQL&#41;](../../t-sql/statements/drop-master-key-transact-sql.md)
- [CLOSE MASTER KEY &#40;Transact-SQL&#41;](../../t-sql/statements/close-master-key-transact-sql.md)
- [Encryption Hierarchy](../../relational-databases/security/encryption/encryption-hierarchy.md)
