## spark use maven-shade-plugin to change local package name to prevent class conflict
* 问题
部署 spark single jar 服务器跑的时候, 发现 jar 包冲突, executor 的 jar 包版本比项目使用的 jar 包版本要低比较多, 然后出现了方法找不到的异常
* 方法
pom 文件中
```
<plugin>  
    <groupId>org.apache.maven.plugins</groupId>  
    <artifactId>maven-shade-plugin</artifactId>  
    <version>3.1.0</version>  
    <executions>  
        <execution>  
            <phase>package</phase>  
            <goals>  
                <goal>shade</goal>  
            </goals>  
            <configuration>  
                <relocations>  
                    <relocation>  
                        <!-- prevent class conflict with executor okio-->  
  <pattern>okio</pattern>  
                        <shadedPattern>shaded.okio</shadedPattern>  
                    </relocation>  
                </relocations>  
                <filters>  
                    <filter>  
                        <artifact>*:*</artifact>  
                    </filter>  
                </filters>  
            </configuration>  
        </execution>  
    </executions>  
</plugin>


```
所有包含 okio 的 package 都加上了 shaded 的前缀, 

![](https://drive.google.com/uc?id=15cWOBVYj1wq0yFSozsqbPtaVp9_8LzB2)

那么这些single jar 里面的 class 则必不可能再被用到了, 那么就会通过相同的 package+className 使用到 executor 上面的 class, 如果能用得上的话(应该是这样吧? 下次碰到再加以验证) 

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTg1MjA3MzcwMV19
-->
