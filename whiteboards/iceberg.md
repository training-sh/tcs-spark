
```


SPARK support 
    OVERWRITE - delete old data and write new data
    APPEND - keep all old files, add new files

    
SPARK do not support UPDATE/DELETE statements

--

ICEBERG
    Table format
        Meta data
        uses parquet to store data [columnar format]
    
    ACID 
    Support insert/update/delete/merge/upsert
    Time Travel - go back in time to see what was the reocrds that point in time
    Rollback - if some operations went wrong , (update/delete/insert), you can rollback to right version
    Stream/Batch
    Upsert

Table that points to a directory

Files inside are immutable, once file is created, you cannot update/delete/insert records inside the file

add new file, remove the file , but cannot modify the files

i want support insert/delete/update (DML) plus meta data (DDL)

order id, amount
insert into orders (1, 100) - created file1.par

insert into orders (2, 200), (3, 300) - create file2.par

iceberg support update

update orders set amount=110 where id = 1
    will check the data in file1,file2 parquet where id = 1
        found file1.par has id = 1
        Copy on Write  (CoW)
            copy the content (matched files - file1.par) into new file (file3.par)
            when id = 1, update the value in memory and write to file3.par else copy record as it is into file3.par

update orders set amount=105 where id = 2
    file2.par content shall be copied into file4.par
        record 2 updated, 3 copied as it is

If I Query orders, will have duplicated records, file1, file2, file3, file4

soft delete file1, file2
    meta data to mark a file to be removed/deleted/ soft delete
    manifest files, will markers to mark a file is added/removed

        insert into orders (1, 100) - created file1.par
            manifest-01.avro (row based)
                verb             file
                add              file1.par

      
        insert into orders (2, 200), (3, 300) - create file2.par

            manifest-02.avro (row based)
                verb             file
                add              file2.par

        (update orders set amount=110 where id = 1)

            manifest-03.avro (row based)
                verb             file
                remove           file1.par
                add              file3.par



        (update orders set amount=105 where id = 2)

            manifest-03.avro (row based)
                verb             file
                remove           file2.par
                add              file4.par

        (delete oders where id = 3)
            manifest-04.avro (row based)
                verb             file
                remove           file4.par
                add              file5.par


ASSUME WE HAVE 100 million records in 10000 of parquet files
DELETE 1 record from this, it may copy 10 million - 1 records
UPDATE - copy all the records, [delete logic]
    i can mark some records as deleted, and only copy updated records


DELETE from order1 where id = 3

/orders
    file1.par
        1, 100 (write)
    file2.par
        2, 200 (write)
        3, 300 (write)

    file3.par (update orders set amount=110 where id = 1)
        1, 110 
    file4.par (update orders set amount=105 where id = 2)
        2, 205
        3, 300
    file5.par (delete where id = 3)
        2, 205 (copied)
```
