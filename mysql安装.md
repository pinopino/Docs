mysql非msi安装包的安装
=========
#### [官网](https://dev.mysql.com/downloads/mysql/)下载mysql的安装文件，是一个zip压缩包，其实可以理解为这种方式安装的mysql是绿色安装的。

#### 解压后，有很多额外的步骤需要执行：
1. 找到解压后文件的bin目录，设置环境变量

2. 可以在cmd窗口中试试直接执行mysql，会报错，但至少能够表明环境变量设置ok了，继续解决错误

3. cmd（管理员运行），执行：
    ```csharp
    mysqld --install
    ```

4. 继续，初始化mysql数据库：
    ```csharp
    mysqld --initialize --user=root --console
    ```
    > _记得拷贝mysql自动为你生成的root初始密码_

5. 继续，启动mysql服务：
    ```csharp
    net start mysql
    ```
    > _这一步可能会有问题，原因在于虽然设置了环境变量，但你最好还是切换到mysql的bin目录来执行net start命令_

6. 继续，登录mysql，重置密码（你不重置系统也会强制要求你重置的）
    ```csharp
    mysql> set password=password('123456'); 
    ```
    或者：
    ```csharp
    mysql> update user set password=password('123456') where user='root';
    ```
    > 这一步也可能会有问题，原因在于mysql在8.0.4之后换掉了密码认证的插件，之前是*mysql_native_password*，而现在使用的是*caching_sha2_password*；
    进一步的还可能导致你较为老旧的mysql管理客户端连不上mysql服务器（因为它们可能使用的还是*mysql_natvie_password*方式）；

    所以修改语句为：
    ```csharp
    mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '新密码';
    ```

7. 继续，绿色版的mysql没有my.ini配置文件，你得自己想办法配置一个（扔到mysql的根目录），常见的一般设置展示如下：
    ```csharp
    [mysqld]
    # 设置3306端口
    port=3306
    # 设置mysql的安装目录
    basedir=D:\Program Files\mysql-8.0.16-winx64
    # 设置mysql数据库的数据的存放目录
    datadir=D:\Program Files\mysql-8.0.16-winx64\data
    # 允许最大连接数
    max_connections=200
    # 允许连接失败的次数。这是为了防止有人从该主机试图攻击数据库系统
    max_connect_errors=10
    # 服务端使用的字符集默认为 UTF8 More Byte 4
    character-set-server=utf8mb4
    # 创建新表时将使用的默认存储引擎
    default-storage-engine=INNODB
    # 默认使用"mysql_native_password"插件认证
    default_authentication_plugin=mysql_native_password
    [mysql]
    # 设置mysql客户端默认字符集
    default-character-set=utf8mb4
    [client]
    # 设置mysql客户端连接服务端时默认使用的端口
    default-character-set=utf8mb4
    ```
    > 这当中有两点值得注意：
       1. 字符集推荐设置成*utf8mb4*而不是*utf8*，具体原因请自行google
       2. 如前所述，考虑到兼容老旧的客户端管理工具，*default_authentication_plugin*推荐设置成*mysql_native_password*

8. 继续， ini文件配置完毕后需要重启mysql服务：
    ```csharp
    net stop mysql
    net start mysql
    ```
9. 最后，登录mysql，验证下之前的操作：
    ```csharp
    mysql> show variables like '%char%';
    ```
    能看到类似：
    ```shell
    +--------------------------+------------------------------------------------------+
    | Variable_name            | Value                                                |
    +--------------------------+------------------------------------------------------+
    | character_set_client     | utf8mb4                                              |
    | character_set_connection | utf8mb4                                              |
    | character_set_database   | utf8mb4                                              |
    | character_set_filesystem | binary                                               |
    | character_set_results    | utf8mb4                                              |
    | character_set_server     | utf8mb4                                              |
    | character_set_system     | utf8                                                 |
    | character_sets_dir       | D:\Program Files\mysql-8.0.16-winx64\share\charsets\ |
    +--------------------------+------------------------------------------------------+
    ```
    则可以认为mysql已经正常跑起来了。