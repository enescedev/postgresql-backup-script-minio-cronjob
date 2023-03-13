# Postgres Backup Cronjob Minio Storage Script
#### Postgresql veritabanı pg_basebackup yöntemi ile yedekleme ve arşivlenen dosyasını MinIO depolama sistemi kullanarak bir depolama servisine yüklemek için cronjob ile otomatikleştirilmiş script hazırlamak.

İlk olarak, ortam değişkenlerini oluşturan:
-  DATABASE: yedeklenecek PostgreSQL belgelerinin adı
-  USERNAME: PostgreSQL arşivlerine erişmek için kullanılan kullanıcı adı
-  BACKUP_DIR: yedek saklamanın kaydedileceği dizin adı
-  BACKUP_PWD: yedek gözlemin kaydedileceği dizin tam yolu
-  BUCKET: yedek depolamanın yükleneceği MinIO nesne depolama servisi URL'si
-  BUCKET_NAME: yedek depolamanın yükleneceği MinIO depolama alanı adı
-  LOG_FILE: log dosyalarının kaydedileceği dizin adı,
belirlenir ve ilgili dizinler oluşturulur.

Daha sonra, 'BACKUP_FILE_NAME' değişkeni , mevcut tarih saatini kullanarak yedek dosya adı oluşturur. 

Backup.sh adında bir dosya oluşturup aşağıdaki komutları dosyaya ekleyebiliriz.

```sh
!/bin/bash

BACKUP_DIR="/products/pgsql/data/15"
BACKUP_DATE=$(date +"%Y-%m-%d-%H-%M-%S")
BACKUP_FILENAME="${BACKUP_DATE}_backup.tar.gz"
BUCKET=myminio
BUCKET_NAME=db-backup-test
LOG_FILE="/products/pgsql/data/backup/logs/${BACKUP_DATE}-logfile.log"

# Yedekleme dosyasının kaydedileceği dizin
BACKUP_DIR="/products/pgsql/data/15"
# Yedekleme dosyasına verilecek isim, şimdiki zaman kullanılarak oluşturuluyor
BACKUP_DATE=$(date +"%Y-%m-%d-%H-%M-%S")
BACKUP_FILENAME="${BACKUP_DATE}_backup.tar.gz"
# Yedekleme dosyasının kaydedileceği minio depolama alanı
BUCKET=myminio
BUCKET_NAME=db-backup-test
# Yedekleme işleminin kayıt edileceği log dosyası
LOG_FILE="/products/pgsql/data/backup/logs/${BACKUP_DATE}-logfile.log"

# Yedekleme işlemi için çalışma dizinini değiştir
cd /products/pgsql/data/backup

# Log dosyasına başlangıç zamanını kaydet
echo "Backup started at $(date)" >> $LOG_FILE

# Basebackup işlemini gerçekleştir
echo "Starting base backup..."  >> $LOG_FILE
# -D: yedeklemenin kaydedileceği dizin, -Ft: yedekleme dosyasının türü (tar formatı), 
# -z: yedekleme dosyasını sıkıştır, -P: ilerleme çubuğu göster, -Xs: standby sunucu için gerekli işaretlerin eklenmesi, -R: standby sunucu için recovery.conf dosyasını oluştur
pg_basebackup -D "$BACKUP_DIR" -Ft -z -P -Xs -R

# Tar arşivi oluştur
echo "Creating tar archive..."  >> $LOG_FILE
# -c: yeni bir arşiv dosyası oluştur, -z: arşiv dosyasını sıkıştır, -v: işlemi ekranda göster, -f: arşiv dosyasının adı
tar -czvf "$BACKUP_FILENAME" "$BACKUP_DIR"

# Yedekleme dosyasını minio depolama alanına kopyala
echo "Copying backup file to minio..."  >> $LOG_FILE
/usr/local/bin/mcli mv $BACKUP_FILENAME $BUCKET/$BUCKET_NAME/$BACKUP_FILENAME

# Yerel yedekleme dosyasını sil
echo "Removing local backup files..."  >> $LOG_FILE
#rm -rf $BACKUP_FILENAME

# Log dosyasına bitiş zamanını kaydet
echo "Backup completed at $(date)" >> $LOG_FILE

# Kullanıcıya tamamlanan yedekleme dosyasının adını bildir
echo "Backup complete. File saved as $BACKUP_FILENAME." >> $LOG_FILE
```

Yedekleme dosyası önce pg_basebackup aracı kullanılarak oluşturulur, ardından tar arşivi içinde sıkıştırılır ve son olarak minio adlı bir depolama alanına kopyalanır. Yerel yedekleme dosyası (tar arşivi) varsayılan olarak silinir.

Ayrıca her adımda log dosyasına kayıt ekler, böylece yedekleme işlemi sırasında hangi adımların gerçekleştirildiği ve hangi aşamalarda hatalar oluştuğu takip edilebilir.

Yedekleme dosyasının kaydedileceği dizin, yedekleme dosyasına verilecek isim ve log dosyasının yolu gibi bazı parametreler başlangıçta tanımlanır. Bu parametrelerin hepsi script'in başlangıcında belirtilir.

Daha sonra, çalışma dizini değiştirilir ve yedekleme işlemi başlatılır. İlk adım, pg_basebackup aracı kullanarak bir "base backup" oluşturmaktır. Bu işlem, veritabanının durumunu kaydederek yedekleme dosyasını oluşturur. Bu işlem tamamlandıktan sonra, oluşturulan yedekleme dosyası tar arşivi içinde sıkıştırılır. Son adım, yedekleme dosyasının minio depolama alanına kopyalanmasıdır.

Yedekleme işlemi tamamlandığında, script yedekleme dosyasının adını ve tamamlanma zamanını log dosyasına kaydeder. Son olarak, kullanıcıya yedekleme işleminin tamamlandığı ve dosyanın nerede kaydedildiği bilgisi gösterilir.

## Cronjob Script

Yazılan script'i hergün çalıştıracak şekilde otomatikleştirmek için cronjob kullanıyoruz. 
- Aşağıdaki komutu kullanarak dosyayı düzenleyebiliriz;
```sh
crontab -e
```

- Aşağıdaki komut dosyanın her gün 12.15 de çalıştırılacak şekilde düzenlenmiş ve çalıştırılacağı dizin dosya adı verilmiş. Ayrıca oluşturulan çıktıyı logs dosyasının içerisine yazacak şekilde düzenlenmiştir.

```sh
15 12 * * * /products/pgsql/data/backup/backup.sh > /products/pgsql/data/backup/logs/cronjob-logfile.log 2>&1
```

> Not: Crontab oluşturulmasıyla ilgili daha detaylı bilgiye aşağıdaki linkten ulaşabilirsiniz.
> https://crontab.guru/

Aşağıdaki komut ile zamanlanmış cronjob işlemini görüntüleyebilirsiniz.
```sh
crontab -l
```

Bu işlemler sayesinde PostgreSQL arşivlerini yedekleme, sıkıştırma ve yedekleme dosyasındaki nesne depolama hizmetinden yararlanmaları otomatik hale getirir.
