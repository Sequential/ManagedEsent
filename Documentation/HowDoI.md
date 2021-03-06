# How Do I Pulse My Transaction?

ESENT implements [snapshot isolation](http://en.wikipedia.org/wiki/Snapshot_isolation) by keeping track of updates made since the oldest transaction in the system began ([multiversioning](http://en.wikipedia.org/wiki/Multiversion_concurrency_control)). This means that long-running transactions can consume lots of version store space in ESENT. To avoid running out of version store transactions should be periodically committed.

```c#
// Insert a lot of records. We batch updates into transactions for performance.
const int BatchSize = 100;
Api.JetBeginTransaction(sesid);
for (int i = 0; i < 256; i++)
{
    Api.JetPrepareUpdate(sesid, tableid, JET_prep.Insert);
    Api.JetUpdate(sesid, tableid);

    if (i % BatchSize == (BatchSize - 1))
    {
        // Long-running transactions will consume lots of version store space in ESENT.
        // Periodically commit the transaction to avoid running out of version store.
        Api.JetCommitTransaction(sesid, CommitTransactionGrbit.LazyFlush);
        Api.JetBeginTransaction(sesid);
    }
}
Api.JetCommitTransaction(sesid, CommitTransactionGrbit.None);
```

# How Do I Retrieve an Auto-Increment Column Value?
When inserting a new record you may want to know the value of the auto-increment column in the record (perhaps because it is being used as a key in an external system). It is possible to retrieve the value as soon as JetUpdate has been called by passing the RetrieveCopy option to JetRetrieveColumn, which will retrieve the column from the record currently under construction.

```c#
using (var update = new Update(sesid, tableid, JET_prep.Insert))
{
    int? autoinc = Api.RetrieveColumnAsInt32(
        sesid,
        tableid,
        autoincColumn,
        RetrieveColumnGrbit.RetrieveCopy);
    Console.WriteLine("{0}", autoinc);
    update.Save();
}
```

# How Do I Make a Multi-Column Key?
To make a key for an index that has multiple columns in it you need to make one call to JetMakeKey for each column. The first call should use the NewKey option. For example, given an index that covers 3 columns:

```c#
Api.MakeKey(sesid, tableid, true, MakeKeyGrbit.NewKey);
Api.MakeKey(sesid, tableid, 8, MakeKeyGrbit.None);
Api.MakeKey(sesid, tableid, "foo", Encoding.Unicode, MakeKeyGrbit.None);
```

It is not required to set all columns, just a prefix key can be made. In that case use SeekGE/SeekLE to find the first matching record.

```c#
Api.MakeKey(sesid, tableid, true, MakeKeyGrbit.NewKey);
Api.MakeKey(sesid, tableid, 8, MakeKeyGrbit.None);
Api.JetSeek(sesid, tableid, SeekGrbit.SeekGE);
```

# How Do I Scan Records?
This pattern can be used to enumerate all records in a table/index:

```c#
Api.MoveBeforeFirst(sesid, tableid);
while (Api.TryMoveNext(sesid, tableid))
{
    // [something](something)
}
```

# How do I Create an Index Range?
Creating an index range has two steps:
1. Seek to the first record in the index range (JetMakeKey+JetSeek or JetMove)
2. Create a key for the last record in the range and create the range (JetMakeKey+JetSetIndexRange)

Here is an example that creates an index range on a string prefix:

```c#
// Seek for any record that is >= "ba". If any record has "ba" as a prefix
// this will land on the first such record. This can also land on records
// that don't have the prefix, e.g. "qux".
Api.MakeKey(sesid, tableid, "ba", Encoding.Unicode, MakeKeyGrbit.NewKey);
// Use TrySeek if you aren't sure that any matching records exist.
Api.JetSeek(sesid, tableid, SeekGrbit.SeekGE);

// If we want to find just records that have "ba" as a prefix we need to 
// set up an index range. We are already positioned on the first record
// so now we need to set up the other end of the index range. Here we use
// the PartialColumnEndLimit to build a key which matches "ba*".
Api.MakeKey(sesid, tableid, "ba", Encoding.Unicode, MakeKeyGrbit.NewKey | MakeKeyGrbit.PartialColumnEndLimit);
if (Api.TrySetIndexRange(sesid, tableid, SetIndexRangeGrbit.RangeInclusive | SetIndexRangeGrbit.RangeUpperLimit))
{
    // the index range has been created and we are on the first record
    do
    {
        // [something](something)
    }
    while (TryMoveNext(sesid, tableid));
}
// Once TryMoveNext returns false we have reached the end of the index
// range. The index range is automatically removed.
```

Here is an example that demonstrates seeks on various string prefix wildcards:

```c#
// We have an index over a string column, which contains 8 records:
//  [A, ANT, B, BAT, C, CAT, D, DOG](A,-ANT,-B,-BAT,-C,-CAT,-D,-DOG)
// The following code demonstrates how to create index ranges over
// all string prefix combinations. The three things that are varied are:
//  1. The use of PartialColumnEndLimit
//  2. The use of SeekGE or SeekGT
//  3. The used of RangeInclusive

// "A**" <= key <= "C**" -> ["A", "ANT", "B", "BAT", "C", "CAT"](_A_,-_ANT_,-_B_,-_BAT_,-_C_,-_CAT_)
Api.MakeKey(sesid, tableid, "A", Encoding.Unicode, MakeKeyGrbit.NewKey);
Api.JetSeek(sesid, tableid, SeekGrbit.SeekGE);
Api.MakeKey(sesid, tableid, "C", Encoding.Unicode, MakeKeyGrbit.NewKey | MakeKeyGrbit.PartialColumnEndLimit);
Api.JetSetIndexRange(sesid, tableid, SetIndexRangeGrbit.RangeUpperLimit | SetIndexRangeGrbit.RangeInclusive);
CheckIndexRange(sesid, tableid, keyColumn, new[]() { "A", "ANT", "B", "BAT", "C", "CAT" });

// "A**" < key <= "C**" -> ["B", "BAT", "C", "CAT"](_B_,-_BAT_,-_C_,-_CAT_)
Api.MakeKey(sesid, tableid, "A", Encoding.Unicode, MakeKeyGrbit.NewKey | MakeKeyGrbit.PartialColumnEndLimit);
Api.JetSeek(sesid, tableid, SeekGrbit.SeekGT);
Api.MakeKey(sesid, tableid, "C", Encoding.Unicode, MakeKeyGrbit.NewKey | MakeKeyGrbit.PartialColumnEndLimit);
Api.JetSetIndexRange(sesid, tableid, SetIndexRangeGrbit.RangeUpperLimit | SetIndexRangeGrbit.RangeInclusive);
CheckIndexRange(sesid, tableid, keyColumn, new[]() { "B", "BAT", "C", "CAT" });

// "A**" <= key < "C**" -> ["A", "ANT", "B", "BAT"](_A_,-_ANT_,-_B_,-_BAT_)
Api.MakeKey(sesid, tableid, "A", Encoding.Unicode, MakeKeyGrbit.NewKey);
Api.JetSeek(sesid, tableid, SeekGrbit.SeekGE);
Api.MakeKey(sesid, tableid, "C", Encoding.Unicode, MakeKeyGrbit.NewKey);
Api.JetSetIndexRange(sesid, tableid, SetIndexRangeGrbit.RangeUpperLimit);
CheckIndexRange(sesid, tableid, keyColumn, new[]() { "A", "ANT", "B", "BAT" });

// "A**" < key < "C**" -> ["B", "BAT"](_B_,-_BAT_)
Api.MakeKey(sesid, tableid, "A", Encoding.Unicode, MakeKeyGrbit.NewKey | MakeKeyGrbit.PartialColumnEndLimit);
Api.JetSeek(sesid, tableid, SeekGrbit.SeekGT);
Api.MakeKey(sesid, tableid, "C", Encoding.Unicode, MakeKeyGrbit.NewKey);
Api.JetSetIndexRange(sesid, tableid, SetIndexRangeGrbit.RangeUpperLimit);
CheckIndexRange(sesid, tableid, keyColumn, new[]() { "B", "BAT" });
```

# How Do I Deal With Key Truncation?
ESENT stores its keys in normalized format to allow for fast lookups. There is a maximum length for the normalized key which, by default, is 255 bytes. Any data beyond this point is silently truncated when the key is constructed. Starting with Windows Vista that behaviour can be changed with the VistaGrbits.IndexDisallowTruncation option to JetCreateIndex.

Key truncation means that two different keys will appear to be the same if the difference occurs after the maximum length of the normalized key. This is particularily compex with strings because the size of a normalized Unicode string (produced by LCMapString) is often much larger than the input string and we can't guarantee exactly how many bytes of a string will be normalized (it depends on the locale, sort options and exact string being normalized).

Starting with Windows Vista ESENT also supports longer keys. The maximum key length depends on the database page size (2,000 bytes for 8kb-32kb pages and 1,000 bytes for 4kb pages). This is a per-index option. When you create a JET_INDEXCREATE do this:

```c#
cbKeyMost = SystemParameters.KeyMost
```
This will give you the largest key size possible on the current version of ESENT -- on XP and Windows 2003 you will just get 255 byte keys. Still, there will always be a limit on the maximum key size which means you have to do an index-range scan to deal with arbitrary-length keys. Given a table with a string key column we can see the truncation problem with this code:

```c#
// Insert some records. The key has the same value for the first
// 4096 characters and then is different. This string is too large
// for even the largest key sizes to differentiate.
// If the index was unique we would get a key duplicate error.
string prefix = new string('x', 4096);
using (var transaction = new Transaction(sesid))
{
    int i = 0;
    foreach (string k in new[]() { "a", "b", "c", "d" })
    {
        using (var update = new Update(sesid, tableid, JET_prep.Insert))
        {
            string key = prefix + k;
            Api.SetColumn(sesid, tableid, keyColumn, key, Encoding.Unicode);
            Api.SetColumn(sesid, tableid, dataColumn, i++);
            update.Save();
        }
    }

    transaction.Commit(CommitTransactionGrbit.LazyFlush);
}

// Seek for a record. This demonstrates the problem with key truncation.
// We seek for the key ending in 'd' but end up on the one ending in 'a'.
string seekKey = prefix + "d";
Api.JetSetCurrentIndex(sesid, tableid, "secondary");
Api.MakeKey(sesid, tableid, prefix + "d", Encoding.Unicode, MakeKeyGrbit.NewKey);
Api.JetSeek(sesid, tableid, SeekGrbit.SeekEQ);
string actualKey = Api.RetrieveColumnAsString(sesid, tableid, keyColumn);
Assert.AreNotEqual(seekKey, actualKey);
Assert.AreEqual(prefix + "a", actualKey);
```

To find the record we want we have to set up an index range to cover the key. That can be done like this:

```c#
private static bool TrySeekTruncatedString(JET_SESID sesid, JET_TABLEID tableid, string key, JET_COLUMNID keyColumn)
{
    // To find the record we want we can set up an index range and iterate
    // through it until we find the record we want. We use the desired key
    // as both the start and end of the range, along with the inclusive flag.
    Api.MakeKey(sesid, tableid, key, Encoding.Unicode, MakeKeyGrbit.NewKey);
    if (!Api.TrySeek(sesid, tableid, SeekGrbit.SeekEQ))
    {
        return false;
    }

    Api.MakeKey(sesid, tableid, key, Encoding.Unicode, MakeKeyGrbit.NewKey);
    Api.JetSetIndexRange(sesid, tableid, SetIndexRangeGrbit.RangeInclusive | SetIndexRangeGrbit.RangeUpperLimit);

    while (key != Api.RetrieveColumnAsString(sesid, tableid, keyColumn))
    {
        if (!Api.TryMoveNext(sesid, tableid))
        {
            return false;
        }
    }

    return true;
}
```

This technique will work on truncated and non-truncated keys so you should use it whenever you have concerns about key truncation.

# How Do I Lock Records?
ESENT uses [optimistic concurrency control](http://en.wikipedia.org/wiki/Optimistic_concurrency). This provides highly concurrent access to data but is tricky when multiple sessions are trying to operate on the same data. Snapshot isolation means that each session will see the same data, but multiple attempts to update the same data will result in one session suceeding and the other sessions getting write conflict errors. A session can lock a record with TryGetLock (which wraps JetGetLock). 

Getting a lock in ESENT is instantaneous; if another thread has the record locked or has updated the record, JetGetLock will fail. **There is no way to wait for the lock to be released**. Because ESENT uses Snapshot Isolation a lock conflict (JET_err.WriteConflict) is permanent -- if session A conflicts with an update made by session B the conflict won't go away if session B commits. Both sessions A and B have to commit before A can restart a transaction without conflict (that is also when A will see B's changes).

Here is a coding pattern that can be used if multiple threads need to process records from a task-type table. Using multiple threads in this way won't speed up a bulk deletion because the threads will compete for the same page latches, but this technique is useful if the worker threads have to do a lot of work to process each record.

```c#
// We must be in a transaction for locking to work.
using (var transaction = new Transaction(this.sesid))
{
    if (Api.TryMoveFirst(this.sesid, this.tableid))
    {
        do
        {
            if (Api.TryGetLock(this.sesid, this.tableid, GetLockGrbit.Write))
            {
                // Do something then delete the record
                // [something](something)
                Api.JetDelete(this.sesid, this.tableid);
            }
        }
        while (Api.TryMoveNext(this.sesid, this.tableid));
    }

    transaction.Commit(CommitTransactionGrbit.LazyFlush);
}
```
