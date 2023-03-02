## postgresql-backup-script-minio-cronjob
## backup and restore postgres data bucket minio

### PostgreSQL arşivlerini yedeklemek, sıkıştırmak ve yedekleme dosyasını MinIO depolama sistemi kullanarak bir depolama servisine yüklemek için kullanılabilir.

## İlk olarak, ortam değişkenlerini oluşturulur:
```DATABASE=userdb_test```
```USERNAME=postgres```
```BACKUP_DIR=test```
```BACKUP_PWD=/products/pgsql/data/backup```
```BUCKET=myminio```
```BUCKET_NAME=db-backup-test```

##### DATABASE: yedeklenecek PostgreSQL belgelerinin adı
##### USERNAME: PostgreSQL arşivlerine erişmek için kullanılan kullanıcı adı
##### BACKUP_DIR: yedek saklamanın kaydedileceği dizin adı
##### BACKUP_PWD: yedek gözlemin kaydedileceği dizin tam yolu
##### BUCKET: yedek depolamanın yükleneceği MinIO nesne depolama servisi URL'si
##### BUCKET_NAME: yedek depolamanın yükleneceği MinIO depolama alanı adı

```BACKUP_FILE_NAME=$DATABASE-$(date +%Y-%m-%d-%H-%M-%S)```

BACKUP_FILE_NAME değişkeni, mevcut tarih saatini kullanarak yedek dosya adı oluşturur. Ardından, yedekleme dizinine geçilir ve PostgreSQL pg_dump yedeklenir ve BACKUP_FILE_NAME adlı dosyaya kopyalanır.

# Log file
```LOG_DIR=/products/pgsql/data/backup/log```
```LOG_FILE=$LOG_DIR/logfile-$(date +%Y-%m-%d-%H-%M-%S)```

```mkdir -p $LOG_DIR```

# İşlem tarihi adında yedek dosya oluştur
```BACKUP_FILE_NAME="$DATABASE-$(date +%Y-%m-%d-%H-%M-%S)"```

```cd $BACKUP_PWD/$BACKUP_DIR```

# Backup alınır ve tarihi adında dosya sıkıştırma yöntemi ile oluşturulur.
```pg_dump -d $DATABASE | gzip > $BACKUP_DIR/$BACKUP_FILE_NAME.sql.tar.gz```

## farklı bir backup yöntemi olan BackRest kullanarak yedekleme başlatılır ve logları $LOG_FILE dizinine yazılır.
#```pgbackrest --stanza=dbname backup >> $LOG_FILE 2>&1```

# Sıkıştırılır ve yeni bir dosya oluşturulur.
```tar -zcvf $BACKUP_FILE_NAME.tar.gz $BACKUP_FILE_NAME```

# minio client kopyalama komutunu çalıştırarak BACKUP_FILE_NAME.tar.gz MinIO nesne depolama servisine yüklenir.
```/usr/local/bin/mcli cp $BACKUP_FILE_NAME.tar.gz $BUCKET/$BUCKET_NAME/$BACKUP_FILE_NAME.tar.gz```

# Yedek silinir.
```rm $BACKUP_DIRECTORY/$BACKUP_FILE_NAME.tar.gz```

 ### PostgreSQL dosyalarının yedek dosyasını geri yüklemek için aşağıdaki işlemleri takip edebilirsiniz:

İlk olarak, yedek dosyasını açmak için yedekleme dosyasının tam yolunu belirtmeniz gerekiyor. Bu betikte, yedek dosyalar ```BACKUP_PWD/$BACKUP_DIR``` dizininde saklanıyor ve yedek dosyanın adı ```DATABASE-$(date +%Y-%m-%d-%H-%M-%S).tar.gz``` oluşturuluyor. Bu nedenle geri yükleme işlemi için yedekleme dosyasının tam yolunu aşağıdaki şekilde belirleyebilirsiniz:

```BACKUP_FILE_PATH=$BACKUP_PWD/$BACKUP_DIR/$DATABASE-$(date +%Y-%m-%d-%H-%M-%S).tar.gz```

# Yedek dosyasını açın
tar -zxvf $BACKUP_FILE_PATH

# PostgreSQL veritabanına geri yükleyin
```psql -U $USERNAME -d $DATABASE < $DATABASE.sql```

