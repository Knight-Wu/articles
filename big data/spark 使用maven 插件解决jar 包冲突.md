
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

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbNDkyMDk1NDk1LDc3Mzk3NjE3NV19
-->