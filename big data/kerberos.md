#### 常用命令

```
kadmin.local // 进入kerb 命令行
在命令行里面输入 ? , 相当于help

addprinc user@domain.com // 创建用户

xst -k user.keytab user@domain.com // 创建授权文件 keytab

 exit // 退出kerb 命令行
 
 
kinit -kt user.keytab user@CDH.MUCFC.COM // 切换用户
```
#### 疑问
* kerberos 命令帮助?


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTg2MzcyNzg3NF19
-->