
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
所有包含 okio 的 package 都加上了 shaded 的前缀, 
![enter image description here](https://drive.google.com/uc?id=15cWOBVYj1wq0yFSozsqbPtaVp9_8LzB2)
```

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbNDE2MjcwNzAsNDkyMDk1NDk1LDc3Mzk3Nj
E3NV19
-->