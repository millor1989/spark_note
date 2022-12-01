### Jar 依赖冲突

当程序抛出 `ClassNotFound`、`NoSuchMethod`、`NoSuchInstance` 之类的异常时，可能就是出现了 Jar 包冲突。

Spark 应用执行时，一般涉及的 jar 文件有四部分：

1. Spark CLASSPATH（`$SPARK_HOME/jars`目录）中的 jar 包【System Classpath】。

   提交应用是，这些 jar 文件会打成一个 zip 包上传到集群中。

2. `spark-submit` 命令 `--jars` 选项指定的 jar 包【User Classpath】。

   提交应用时应用本身的 jar 包和这类 jar 包会上传到集群中。`--jar` 指定的 jar 包会被包含在 driver 和 executor 的 classpath 中。

3. `spark-submit` 命令 `--conf "spark.{driver/executor}.extraClassPath=someJar"` 选项指定的 jar 包【User Classpath】

4. YARN 环境 jar 包（Spark ON Yarn 时，Hadoop 相关的一些 jar 包）【System Classpath】

默认的 jar 引用优先级是 System Classpath 中的 jar 优先。

- 如果出现 jar 冲突引起的异常，首先要检查应用本身的 jar 包，排除应用中冲突的 jar 包。

- 如果是 User Classpath 中的 jar 与 System Classpath 中的 jar 冲突引起的，可以使用 `--conf "spark.{driver/executor}.userClassPathFirst=true"` 以优先使用 User Classpath 中的 jar。但是，这样可能会影响 Spark 本身一些功能的执行，需要权衡。

- 对于 User Classpath 中的 jar 与 System Classpath 中的 jar 冲突，不方便使用 `userClassPathFirst` 进行解决时。可以使用 `maven-shade-plugin` 插件的 `relocation` 功能将包路径转移。比如 `httpclient` 版本冲突时，作如下配置：

  ```xml
  <plugin>
  	<groupId>org.apache.maven.plugins</groupId>
  	<artifactId>maven-shade-plugin</artifactId>
  	<version>3.1.1</version>
  	<configuration>
  		<shadedArtifactAttached>true</shadedArtifactAttached>
  		<transformers>
  		</transformers>
  	</configuration>
  	<executions>
  		<execution>
  			<phase>package</phase>
  			<goals>
  				<goal>shade</goal>
  			</goals>
  			<configuration>
  				<relocations>
  					<relocation>
  						<pattern>org.apache.http</pattern>
  						<shadedPattern>shaded.org.apache.http</shadedPattern>
  					</relocation>
  				</relocations>
  			</configuration>
  		</execution>
  	</executions>
  </plugin>
  ```

  此时，实际上是将用户应用 shaded jar 中 `org.apache.http` 路径下的所有类文件转移到了 `shaded.org.apache.http` 路径之下。同时，shaded jar 中所有的 `org.apache.http` 引用也被更改为 `shaded.org.apache.http`。