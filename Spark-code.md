
```python
# this function invoked by executor for each number
# each number, there will be delay of 5 to 15 seconds
# we throw random error to demonstrate retry
# RDD - Resilient (Fault Tolerance) Distributed (Partition) Dataset
def odd(n):
    import random
    import time
    time.sleep(random.randint(5, 15)) # delay jobs for  5 to 15, assume that you huge data in partition
    print ("processing odd", n)
    # to simulate errors during spark data processing
    # assume, you connected to mysql/jdbc/pg/ network error
    e = random.randint(5, 15)
    if e % 3 == 0: raise Exception("Something bad")
    return n % 2 == 1
```

```
odd_rdd = numbers_rdd.filter (odd) # transformation, no action, no job
odd_rdd.collect() # jupyter shell, print last executed expression output
```
