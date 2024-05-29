![](https://i.imgur.com/q6lmcyN.png)

요런 오류가 뜨면 다음과 같은 과정을 하면 된다.

```
libtoolize --force
aclocal
autoheader
automake --force-missing --add-missing
autoconf
./configure
```
