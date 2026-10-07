```
sales - 1 TB
  file1.par
  ..
  ..
  file-n.par


select * from sales wehre year = 2026
  SCAN 1 TB 

sales_partitioned - 1 TB
    year=2026 - 50 GB
       month=01 - 5 GB 
         file1.par
         ..
         filen.par

       month=02 - 4 GB 
          ..

    year=2025 - 60 GB
        month=08
        ...
           file1...

    ...



select * from sales_partitioned  amount > 100;
  SCAN 1 TB , query does not include partition column names 

Partition pruning 

select * from sales_partitioned  amount > 100 and YEAR=2026;
  SCAN   year=2026, 50 GB  , partition column present in year


select * from sales_partitioned  amount > 100 and YEAR=2026 and month=01;
  SCAN   year=2026/month=01 - 5 GB   , partition column present in year and month


```
