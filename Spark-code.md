
```python
def odd(n):
    import random
    import time
    time.sleep(random.randint(5, 15)) # delay jobs for  5 to 15, assume that you huge data in partition
    print ("processing odd", n)
    return n % 2 == 1
```
