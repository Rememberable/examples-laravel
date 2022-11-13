```shell
sudo service docker start
./vendor/bin/sail up
alias sail='[ -f sail ] && sh sail || sh vendor/bin/sail'
sail up
```
