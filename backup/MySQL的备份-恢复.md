
### 备份

**mysqldump**

**安装：**存储库下载[[地址](https://dev.mysql.com/downloads/)](https://dev.mysql.com/downloads/)，选择对应的包管理器

备份多个数据库脚本：

```shell
#!/bin/bash

# 配置部分
DB_HOST="localhost"
DB_USER="your_username"
DB_PASS="your_password"
DATABASES=("database1" "database2" "database3")  # 需要备份的数据库列表
BACKUP_DIR="/path/to/your/backup/directory"
DATE=$(date +'%Y-%m-%d_%H-%M-%S')

# 创建备份目录（如果不存在）
mkdir -p $BACKUP_DIR

# 备份每个数据库
for DB_NAME in "${DATABASES[@]}"; do
  BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_backup_$DATE.sql"
  mysqldump -h $DB_HOST -u $DB_USER -p$DB_PASS $DB_NAME > $BACKUP_FILE
  
  # 检查备份是否成功
  if [ $? -eq 0 ]; then
    echo "[$DATE] Database backup successful: $BACKUP_FILE"
  else
    echo "[$DATE] Database backup failed for $DB_NAME" >&2
  fi
done

# 保留最近7天的备份，删除更早的备份文件
find $BACKUP_DIR -type f -name "*_backup_*.sql" -mtime +7 -exec rm {} \;

echo "[$DATE] Old backups cleaned up"
```

备份整个数据库实例：

```shell
#!/bin/bash

# 配置部分
DB_HOST="localhost"
DB_USER="your_username"
DB_PASS="your_password"
BACKUP_DIR="/path/to/your/backup/directory"
DATE=$(date +'%Y-%m-%d_%H-%M-%S')
BACKUP_FILE="$BACKUP_DIR/all_databases_backup_$DATE.sql"

# 创建备份目录（如果不存在）
mkdir -p $BACKUP_DIR

# 备份整个 MySQL 实例
mysqldump -h $DB_HOST -u $DB_USER -p$DB_PASS --all-databases > $BACKUP_FILE

# 检查备份是否成功
if [ $? -eq 0 ]; then
  echo "[$DATE] All databases backup successful: $BACKUP_FILE"
else
  echo "[$DATE] All databases backup failed" >&2
  exit 1
fi

# 保留最近7天的备份，删除更早的备份文件
find $BACKUP_DIR -type f -name "all_databases_backup_*.sql" -mtime +7 -exec rm {} \;

echo "[$DATE] Old backups cleaned up"
```

### 恢复

```shell
mysql -u your_username -p -e "CREATE DATABASE IF NOT EXISTS database1;"
mysql -u your_username -p database1 < /path/to/your/backup/directory/database1_backup.sql
# 恢复整个实例
mysql -u your_username -p < /path/to/your/backup/directory/all_databases_backup.sql
```