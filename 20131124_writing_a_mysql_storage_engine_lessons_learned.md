# Introduction
This post documents a lot of nasty details I had to learn the hard way when I wrote a MySQL storage engine for the first time. The explanations are kept rather short and are indented for people writing a storage engine them-self. Most of this knowledge comes from reading the MySQL source code and online forums. I will add new points in the future. Currently, the following points are covered.

* [Keyread optimization](#keyread-optimization)
* [Using extended MySQL classes](#using-extended-mysql-classes)
* [Row-level locking](#row-level-locking)
* [Understanding the `index_read()` flags](#understanding-the-index-read-flags)
* [Single vs. multi-statement transactions](#single-vs-multi-statement-transactions)
* [Detecting the start/end of a statement/transaction](#detecting-the-startend-of-a-statementtransaction)

## Lessons learned
### Keyread optimization
Some indexes can fully reconstruct the column data from an index key. Hence, query statements that only refer to the indexed columns don't need to read the corresponding table row. This is known as *keyread optimization*. To find out whether the whole row needs to be read, or if all required row columns can be fully reconstructed from the index key, the `HA_EXTRA_KEYREAD` and `HA_EXTRA_NO_KEYREAD` flags passed to the method `handler::extra()` can be checked. Additionally, the index must signal the storage engine that it supports the keyread optimization by returning the additional flag `HA_KEYREAD_ONLY` from the `handler::index_flags()` method.

### Using extended MySQL classes
In order to use extended MySQL classes (like e.g. the `THD` class) in a MySQL plugin, `MYSQL_SERVER` needs to be defined as `1` before including any MySQL header. A typical include block looks like this:

```
#define MYSQL_SERVER 1 // required for THD class
#include <sql_table.h>
#include <sql_class.h>
#include <probes_mysql.h>
```

### Row-level locking
For a long time *table locking* was used to implement multi-version concurrency control (MVCC) in DBMS. However, modern storage engines tend to implement finer grained *row-level locking* to speed-up concurrent accesses to the same table. In MySQL row-level locking can be implemented in the `handler::store_lock()` method by cleverly adjusting the locking flags.

```
THR_LOCK_DATA ** handler::store_lock(THD *thd, THR_LOCK_DATA **to,
                                     enum thr_lock_type lockType)
{
    if (lockType != TL_IGNORE && lock.type == TL_UNLOCK)
    {
        if (lockType >= TL_WRITE_CONCURRENT_INSERT && lockType <= TL_WRITE)
        {
            if (!thd->in_lock_tables && !thd->tablespace_op)
                lockType = TL_WRITE_ALLOW_WRITE;
        }
        else if (lockType == TL_READ_NO_INSERT && !thd->in_lock_tables)
            lockType = TL_READ;

        lock.type = lockType;
    }

    *to++ = &lock;
    return to;
}
```

### Understanding the index-read flags
In the `handler::index_read()` method the next index value has to be read as indicated by the `ha_rkey_function` argument. As the MySQL flag constants are not very intuitive to grasp, the five different index read modes are explained with the help of an example. Let's assume a table column contains the following set of values: 1, 1, 1, 5, 5, 5, 7, 7, 7. In the table below the results of all possible index reads are depicted. The result of each index read is marked bold.

|SQL      |Key |Flag                  |Found                |
|---------|----|----------------------|---------------------|
|`a = 5`  |5   |`HA_READ_KEY_EXACT`   |1 1 1 **5** 5 5 7 7 7|
|`a = 6`  |6   |`HA_READ_KEY_EXACT`   |not found            |
|`a >= 5` |5   |`HA_READ_KEY_OR_NEXT` |1 1 1 **5** 5 5 7 7 7|
|`a >= 6` |6   |`HA_READ_KEY_OR_NEXT` |1 1 1 5 5 5 **7** 7 7|
|`a <= 5` |5   |`HA_READ_KEY_OR_PREV` |1 1 1 **5** 5 5 7 7 7|
|`a <= 6` |6   |`HA_READ_KEY_OR_PREV` |1 1 1 5 5 **5** 7 7 7|
|`a > 5`  |5   |`HA_READ_AFTER_KEY`   |1 1 1 5 5 5 **7** 7 7|
|`a > 6`  |6   |`HA_READ_AFTER_KEY`   |1 1 1 5 5 5 **7** 7 7|
|`a < 5`  |5   |`HA_READ_BEFORE_KEY`  |1 1 **1** 5 5 5 7 7 7|
|`a < 6`  |6   |`HA_READ_BEFORE_KEY`  |1 1 1 5 5 **5** 7 7 7|

### Single vs. multi-statement transactions
MySQL distinguishes between single and multi-statement transactions. Single statement transactions are transactions that are executed in auto-commit mode. In auto-commit mode MySQL performs a commit operation after each statement. Multi-statement transactions consist of a `BEGIN TRANSACTION` and a `COMMIT` command. All statements that are executed in between those two commands are not auto-committed. They are only committed when `COMMIT` is invoked. When a new transaction is started via the `trans_register_ha()` method, the second argument indicates if the transaction to be started is a single or multi-statement transaction. A simple way to figure this out is to use the `THD::in_multi_stmt_transaction_mode()` method. Hence, a new transaction is properly started by calling:

```
trans_register_ha(thd, thd->in_multi_stmt_transaction_mode(), ht);
```

### Detecting the start/end of a statement/transaction
The MySQL source code is already relatively old. This is also reflected by the storage engine API. Especially, the way storage engines usually detect the start and end of a statement or a transaction is rather obscure and a lot of people ask how to do it.
There are two methods that are usually used to detect if a statement or a transaction has started: `handler::external_lock()` and `handler::start_stmt()`. MySQL calls `external_lock()` once at the beginning of a statement for each table that is used in this statement. In case a table was locked previously by a `LOCK TABLES` command, `external_lock()` won't be called anymore. In that case the `start_stmt()` method is invoked once per table instance during `LOCK TABLES`. If auto-committing is enabled (see the previous point) each statement must be committed individually. Otherwise, all changes performed by the statements of one transaction must be committed at once. To determine which commit mode is active the `in_multi_stmt_transaction_mode()` method is used.

```
int Handler::external_lock(THD *thd, int lockType)
{
    ha_statistic_increment(&SSV::ha_external_lock_count);

    if (lockType == F_UNLCK) 
    {
        auto *tsx = (Transaction *)thd_get_ha_data(thd, ht);
        tsx->NumUsedTables--;

        if (!tsx->NumUsedTables)
        {
            // end of statement

            if (!thd->in_multi_stmt_transaction_mode())
            {
                // auto-commit statement
                tsx->Commit();
                thd_set_ha_data(thd, ht, nullptr);
            }
        }
    }
    else
    {
        auto *tsx = (Transaction *)thd_get_ha_data(thd, ht);
        if (!tsx)
        {
            // beginning of new transaction
            tsx = TsxMgr->NextTsx();
            thd_set_ha_data(thd, ht, tsx);
            trans_register_ha(thd, thd->in_multi_stmt_transaction_mode(), ht);
        }

        if (!tsx->NumUsedTables)
        {
            // beginning of new statement
        }

        tsx->NumUsedTables++;
    }

    return 0;
}

// handlerton callback function
int Commit(handlerton *hton, THD *thd, bool all)
{
    if (thd->in_multi_stmt_transaction_mode() || all)
    {
        // end of transaction =>
        // commit transaction (non auto-commit)
        status_var_increment(thd->status_var.ha_commit_count);
        auto *tsx = (Transaction *)thd_get_ha_data(thd, hton);
        tsx->Commit();
        thd_set_ha_data(thd, hton, nullptr);
        return 0;
    }
    else
    {
        // end of statement
    }

    return 0;
}
```