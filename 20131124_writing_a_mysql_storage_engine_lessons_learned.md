# Introduction
This post documents a lot of nasty details I had to learn the hard way when I wrote a MySQL storage engine for the first time. The explanations are kept rather short and are indented for people writing a storage engine them-self. Most of this knowledge comes from reading the MySQL source code and online forums. I will add new points in the future.

* [Return values of row retrieval functions](#return-values-of-row-retrieval-functions)
* [Using extended MySQL classes](#using-extended-mysql-classes)
* [Row-level locking](#row-level-locking)
* [Understanding the `index_read()` flags](#understanding-the-index-read-flags)
* [Single vs. multi-statement transactions](#single-vs-multi-statement-transactions)
* [Detecting the start/end of a statement/transaction](#detecting-the-startend-of-a-statementtransaction)
* Index key handling
    * [Make sure to implement the `records_in_range()` approximation](#make-sure-to-implement-the-records-in-range-approximation)
    * [Keyread optimization](#keyread-optimization)
    * [Mapping row buffers to index keys](#mapping-row-buffers-to-index-keys)
    * [Comparing index keys](#comparing-index-keys)
    * [Testing keys for uniqueness on row insert/update](#testing-keys-for-uniqueness-on-row-insertupdate)

## Lessons learned

### Return values of row retrieval functions
MySQL calls a couple of handler methods of the storage engine in order to retrieve rows from a table. The methods in question are `rnd_next()`, `index_read()`, `index_read_last()`, `index_first()`, `index_last()`, `index_next()`, `index_prev()` and `index_read_idx()`. All these methods return `int` as result. Obviously, in case a row could be successfully read `0`, and if no matching row could be found `HA_ERR_END_OF_FILE` must be returned. It's less obvious that additionally, the `table->status` attribute must be updated accordingly to `0` on success and to `STATUS_NOT_FOUND` on failure. If you forget that you'll encounter unexpected behavior.

### Using extended MySQL classes
In order to use extended MySQL classes (like e.g. the `THD` class) in a MySQL plugin, `MYSQL_SERVER` needs to be defined as `1` before including any MySQL header. A typical include block looks like this:

```
#define MYSQL_SERVER 1 // required for THD class
#include <sql_table.h>
#include <sql_class.h>
#include <probes_mysql.h>
```

### Row-level locking
For a long time *table locking* was used to implement multi-version concurrency control (MVCC) in DBMS. However, modern storage engines tend to implement finer grained *row-level locking* to speed-up concurrent accesses to the same table. In MySQL row-level locking can be implemented in the `handler::store_lock()` method by cleverly adjusting the locking flags. When a pending, concurrent write operation is signaled to the storage engine, the `store_lock()` method returns `TL_WRITE_ALLOW_WRITE` as long as the table isn't locked and no table space operation is currently executed. In queries like `INSERT INTO table0 SELECT ... from table1 ...` MySQL by default protects `table1` from write operations by passing the `TL_READ_NO_INSERT` flag. Though, this flag conflicts with the `TL_WRITE_ALLOW_WRITE` flag, blocking all write operations on `table1`. The conflict can be solved by converting the lock to a normal read lock `TL_READ`.

```
THR_LOCK_DATA ** Handler::store_lock(THD *thd, THR_LOCK_DATA **to, enum thr_lock_type lockType)
{
    if (lockType != TL_IGNORE && lock.type == TL_UNLOCK)
    {
        // allow concurrent write operations on table
        if (lockType >= TL_WRITE_CONCURRENT_INSERT && lockType <= TL_WRITE)
        {
            if (!thd->in_lock_tables && !thd->tablespace_op)
                lockType = TL_WRITE_ALLOW_WRITE;
        }
        // allow write operations on tables currently read from
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

### Make sure to implement the records-in-range approximation
The method `records_in_range()` of the `handler` class is supposed to return an approximation of the number of rows between two keys in an index. It's used by the query optimizer to determine which table index to use for a query (e.g. for a `JOIN`). It happens easily to oversee this hugely important function, because there's a default implementation which makes it unnecessary to implement it. Unfortunately, the default implementation simply returns `10`, effectively drawing any index selection pointless.

### Keyread optimization
Some indexes can fully reconstruct the column data from an index key. Hence, query statements that only refer to the indexed columns don't need to read the corresponding table rows from disk. This is known as *keyread optimization*. To find out whether the whole row needs to be read or if all required row columns can be fully reconstructed from the index key, the `HA_EXTRA_KEYREAD` and `HA_EXTRA_NO_KEYREAD` flags passed to the `handler::extra()` method can be checked. Additionally, the storage engine must signal to MySQL that it supports the keyread optimization by additionally returning the `HA_KEYREAD_ONLY` flag from the `handler::index_flags()` method.

### Mapping row buffers to index keys
When a new row is inserted into a table or when an existing row is replaced, the table indexes must be updated accordingly. The new index keys are derived from the new row data. For efficiency reasons only those parts of the row that correspond to table columns indexed by the current index should end up in the key. The key format can be choosen arbitrarily. Though, MySQL provides the `key_copy()` function which comes in very handy when creating index keys. The following method `KeyFromRowBuffer()` maps a row buffer to a key in MySQL's default key format. The advantage of using MySQL's default keys format is that the *key compare function* can be implemented in merely a few lines of code (see the next point).

``` cpp
size_t Handler::KeyFromRowBuffer(uint8_t *rowData, size_t keyIndex, uint8_t *keyData) const
{
    auto &ki = table->key_info[keyIndex];
    key_copy(keyData, rowData, &ki, 0);
    return table->key_info[keyIndex].key_length;
}
```

### Comparing index keys
Every database index needs to compare index keys to store its data in sorted order and to retrieve data from the index. Usually, for this purpose a *key compare function* is passed to the index data structure. The key compare function has three arguments: the two keys to be compared `key0` and `key1` and a user defined parameter `param`. The user defined parameter can be used to pass in information about the underlying key structure from MySQL. The key compare function returns `0` if both keys are equal, `-1` if `key0 < key1` and `1` if `key0 > key1`. If you stick to the way of creating index keys described in the previous [point](Creating-index-keys-from-rows), you can use the following generic key compare function.

``` cpp
static int CmpKeys(uint8_t *key0, uint8_t *key1, const void *param)
{
    const KEY *key = (KEY *)param; // obtain pointer to KEY structure
    int res = 0;

    for (size_t i=0; i<key->user_defined_key_parts && !res; i++)
    {
        const auto &keyPart = key->key_part[i];
        const int off = (keyPart.null_bit ? 1 : 0); // to step over null-byte

        if (keyPart.null_bit) // does the key part have a null-byte?
            if (*key0 != *key1)
                return (int)((*key1)-(*key0));

        res = keyPart.field->key_cmp(key0+off, key1+off); // compare key parts
        key0 += keyPart.store_length; // go to next key part
        key1 += keyPart.store_length;
    }

    return res;
}
```

### Testing keys for uniqueness on row insert/update
MySQL distinguishes between *unique* and *non-unique* indexes. In unique indexes each table row can be uniquely identified by exactly one index key. A unique index can be created by invoking `CREATE UNIQUE INDEX index ON table (column0, column1, ...)` or directly when creating the table. A *primary key* is always implicitly unique because it's used together with *foreign keys* to establish links between different tables.  
When new rows are inserted into a table via `write_row()` or when existing rows are updated via `update_row()`, the storage engine must guarantee that the unique key constraint isn't violated for any unique index. This can be achieved by testing if the key to be inserted already exists in any of the unique indexes. An index is a unique index if the `HA_NOSAME` bit in the index flags is set or if the index is the primary key index. The index as such and the row to key mapping is implementation specific. For one simple way to map row buffers to keys read this [point](#Creating-index-keys-from-row-buffers).

``` cpp
bool Handler::IsKeyUnique(uint8_t *rowData)
{
    // optimizes inserts in some cases
    if (!thd_test_options(ha_thd(), OPTION_RELAXED_UNIQUE_CHECKS))
    {
        for (size_t i=0; i<table->s->keys; i++)
        {
            // it's a unique key?
            if ((table->key_info[i].flags&HA_NOSAME) || i == table->s->primary_key)
            {
                // create key + check if unique
                uint8_t keyData[MAX_KEY_LENGTH];
                KeyFromRowBuffer(rowData, i, keyData);
                
                if (Index[i]->ExistsKey(RowDataToKey(i, rowData)))
                    return false;
            }
        }
    }
    
    return true;
}

int Handler::write_row(uint8_t *rowData) // same for update_row()
{
    if (!IsKeyUnique(rowData))
        return HA_ERR_FOUND_DUP_KEY;

    // insert row into table and into index
    return 0;
}
```