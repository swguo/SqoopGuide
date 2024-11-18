以下是一份繁體中文的操作手冊，描述了安裝 **Sqoop** 和 **MySQL**，以及如何完成從 MySQL 導入 HDFS，並從 HDFS 匯入回 MySQL 的完整過程：

---

# **Sqoop 與 MySQL 數據整合操作手冊**

此操作手冊介紹如何在 Ubuntu 環境下安裝 **Sqoop** 和 **MySQL**，配置北風資料庫，並使用 Sqoop 實現：
1. **從 MySQL 導入數據到 HDFS**。
2. **從 HDFS 匯入數據到 MySQL**。

---

## **安裝與配置步驟**

### **1. 安裝並配置 Sqoop**
登入至 hadoop 帳號
```bash
    sudo su - hadoop
```
#### **步驟 1.1: 下載並安裝 Sqoop**
1. 下載 Sqoop 壓縮檔案：
   ```bash
   wget https://archive.apache.org/dist/sqoop/1.4.7/sqoop-1.4.7.bin__hadoop-2.6.0.tar.gz
   ```

2. 解壓並移動到指定目錄：
   ```bash
   tar -xvzf sqoop-1.4.7.bin__hadoop-2.6.0.tar.gz
   sudo mv sqoop-1.4.7.bin__hadoop-2.6.0 /usr/local/sqoop
   ```

3. 配置環境變數：
   - 編輯 `~/.bashrc`
     ```bash
     vim ~/.bashrc
     ```
     添加以下內容：
     ```bash
     export SQOOP_HOME=/usr/local/sqoop
     export PATH=$PATH:$SQOOP_HOME/bin
     ```
   - 更新配置：
     ```bash
     source ~/.bashrc
     ```

4. 驗證 Sqoop 是否成功安裝：
   ```bash
   sqoop version
   ```

---

### **2. 配置 Sqoop 與 MySQL 連接**

#### **步驟 2.1: 添加必要的 JAR 文件**
1. 下載 MySQL 驅動：
   ```bash
   wget https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.0.33/mysql-connector-j-8.0.33.jar
   sudo mv mysql-connector-j-8.0.33.jar /usr/local/sqoop/lib/
   ```

2. 添加 Apache Commons Lang JAR：
   ```bash
   wget https://repo1.maven.org/maven2/commons-lang/commons-lang/2.6/commons-lang-2.6.jar
   sudo mv commons-lang-2.6.jar /usr/local/sqoop/lib/
   ```

3. 配置 Java 和 Hadoop CLASSPATH：
   - 編輯 `~/.bashrc`：
     ```bash
     vim ~/.bashrc
     ```
     添加以下內容：
     ```bash
     export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
     export PATH=$JAVA_HOME/bin:$PATH
     export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:/usr/local/sqoop/lib/commons-lang-2.6.jar
     ```
   - 更新配置：
     ```bash
     source ~/.bashrc
     ```

---

### **3. 安裝並配置 MySQL**

#### **步驟 3.1: 安裝 MySQL**
1. 更新系統並安裝 MySQL：
   ```bash
   sudo apt update
   sudo apt install mysql-server -y
   ```

2. 啟動 MySQL 並執行安全設置：
   ```bash
   sudo service mysql start
   sudo mysql_secure_installation
   ```

---

#### **步驟 3.2: 創建北風資料庫**
1. 下載北風數據庫的 SQL 文件：
   ```bash
   wget https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/northwindextended/Northwind.MySQL5.sql
   ```

2. 登錄 MySQL 並創建數據庫：
   ```bash
   sudo mysql -u root -p
   ```

3. 在 MySQL 中執行以下命令：
   ```sql
   CREATE DATABASE northwind;
   USE northwind;
   SOURCE Northwind.MySQL5.sql;
   SHOW TABLES;
   ```

4. 為 Sqoop 配置 MySQL 用戶：
   ```sql
   CREATE USER 'root'@'localhost' IDENTIFIED BY 'hadoop_csim';
   GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' WITH GRANT OPTION;
   ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'hadoop_csim';
   FLUSH PRIVILEGES;
   ```
5. 離開資料庫
```sql
exit;
```
---

## **數據操作**

### **1. 使用 Sqoop 將 MySQL 數據導入 HDFS**

#### **步驟 1.1: 測試 Sqoop 與 MySQL 的連接**
執行以下命令，列出 MySQL 數據庫中的表：
```bash
sqoop list-tables \
  --connect jdbc:mysql://localhost/northwind \
  --username root --password "hadoop_csim" \
  --driver com.mysql.cj.jdbc.Driver
```

#### **步驟 1.2: 將數據導入 HDFS**
導入 `Customers` 表到 HDFS：
```bash
sqoop import \
  --connect jdbc:mysql://localhost/northwind \
  --username root --password "hadoop_csim" \
  --table Customers \
  --m 1 \
  --target-dir /user/hadoop/northwind/customers \
  --driver com.mysql.cj.jdbc.Driver
```

#### **步驟 1.3: 驗證數據是否成功導入**
檢查 HDFS 中是否存在數據：
```bash
hadoop fs -ls /user/hadoop/northwind/customers
hadoop fs -cat /user/hadoop/northwind/customers/part-m-00000
```

---

### **2. 使用 Sqoop 將 HDFS 數據匯入 MySQL**

#### **步驟 2.1: 創建目標表**
登錄 MySQL，並執行以下命令：
```bash
sudo mysql -u root -p
```
選擇 northwind 資料庫
```sql
USE northwind;
```

創建 export_customers 表格
```sql
CREATE TABLE export_customers (
    CustomerID VARCHAR(10),
    CompanyName VARCHAR(50),
    ContactName VARCHAR(50),
    ContactTitle VARCHAR(50),
    Address VARCHAR(100),
    City VARCHAR(50),
    Region VARCHAR(50),
    PostalCode VARCHAR(10),
    Country VARCHAR(50),
    Phone VARCHAR(20),
    Fax VARCHAR(20)
);

```
離開資料庫
```sql
exit;
```

#### **步驟 2.2: 將數據匯入 MySQL**
從 HDFS 匯入數據到 MySQL 的目標表：
```bash
sqoop export \
  --connect jdbc:mysql://localhost/northwind \
  --username root --password "hadoop_csim" \
  --table export_customers \
  --export-dir /user/hadoop/northwind/customers \
  --input-fields-terminated-by ',' \
  --lines-terminated-by '\n' \
  --driver com.mysql.cj.jdbc.Driver
```

#### **步驟 2.3: 驗證數據匯入結果**
```bash
    sudo mysql -u root -p
```
在 MySQL 中檢查數據：
```sql
SELECT COUNT(*) FROM export_customers;
SELECT COUNT(*) FROM Customers;
```

---

## **驗證結果**
- 確認兩個表的行數是否一致：
  ```sql
  SELECT COUNT(*) FROM Customers;         -- 應為 93 筆
  SELECT COUNT(*) FROM export_customers;  -- 應為 93 筆
  ```
離開資料庫
```sql
exit;
```
---

## **注意事項**
1. 確保 Hadoop 和 MySQL 已正確配置並運行。
2. Sqoop 使用的驅動版本應與 MySQL 版本相容。
3. 如遇權限問題，檢查 HDFS 和 MySQL 用戶權限。

--- 

透過此操作手冊，您可以高效完成 Sqoop 與 MySQL 的數據集成操作，實現 MySQL 和 HDFS 間的雙向數據傳輸。