# Veri Senkronizasyon Sistemi

## Genel Bakış

Yapıyı büyük hacimli bir projeymiş ya da büyümesi bekleniyor gibi kurgulamak istedim. Farklı bileşenlerin birbirine bağımlı olmaması, kurgulanmış servislerin sonradan değiştirilmeden ya alternatifleriyle değişebilmesi ya da extends edilmesinin mümkün olduğu, mümkün olduğunca her şeyin configürasyondan ve ortam değişkenlerinden ayarlanabildiği bir yapı oluşturma düşüncesiyle bir altyapı hazırladım.

Projede kullandığım teknolojiler:

Laravel 12
PostgreSQL
Redis
VueJs

## Kurulum

Kurulum için gerekenler:
    - git
    - Docker
    - Docker Compose

Öncelikle repoyu clonelamak için:

```
git clone git@github.com:toramanlis/rg-labs-product-sync-case.git --recurse-submodules
```

Kuruluma başlarken oluşturulması gereken 3 adet `.env` dosyası var. Biri proje root dizininde docker tarafından kullanıclacak değerler için. Diğer ikisi de `backend` ve `frontend` dizinlerinin içinde ilgili değerlere ait. Her üçü için de birer `.env.example` mevcut. Local ortam için doğrudan kopyalanarak kullanılabilir.

```
cd rg-labs-product-sync-case
cp .env.example .env
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
```

Bu noktadan sonra paket kurulumları, migration ve seed işlemleri yapıldıktan sonra proje ayağa kaldırılabilir.

```
docker compose run app sail composer install
docker compose run app sail php artisan migrate
docker compose run app sail php artisan db:seed
docker compose run fe npm install
docker compose up
```

Ayrıca proje root dizininde bir `.devcontainer.example` dizini var. Bunu da `.devcontainer` ismiyle kopyalarsak VsCode ve Cursor ile devcontainer içnde çalışılabilir.


## Çalıştırma

Servisler ayağa kalktığında otomatik olarak backend ve frontend serverları ile queue worker ve scheduler çalışmaya başlayacaktır. Veritabanını hazırlamak için `app` containerının içinden:

```
php artisan artisan migrate
php artisan artisan db:seed
```

Host makineden:

```
docker compose exec app php artisan migrate
docker compose exec app php artisan db:seed
```

komutlarından biri çalıştırıldıktan sonra sistem tamamen hazır hale gelecektir.

Frontend: http://localhost:5173
Backend: http://localhost

Uçbirimlerinden erişime açık olacaktır. 

## Teknik Kararlar ve Gerekçeleri

### Provider Tanımları
Çoklu provider konusunu bir adım ileri götürme inisiyatifi aldım. Provider API'larını birer provider tipi olarak düşünüp, bu provider tiplerine ait farklı hesaplarla farklı entegrasyonlar yapmak mümkün olarak şekilde bir altyapı oluşturdum.

Çok sayıda markaya hizmet sağlam için geliştirilen sistemlerde özellikle bu seçenekleri istiyoruz. Örneğin bir e-ticaret altyapısında bir panelden birden fazla Trendyol mağazası yönetmek gibi ihtiyaçlar olabiliyor.

#### SyncLog Yapısı
Her sync operasyonu başına bir sync_log entrysi oluşturmak yerine, işlemin farklı adımlarında ayrı ayrı log kayıtları oluşturup, bunları birbirine bağlaması için her bir sync operasyon için üretilen bir sync_key alanı ekledim.

#### Event Based Loglama
Sync döngüsünün anlamlı noktalarında yapılan loglamaların event driven olmasını tercih ettim. Bunun sebebi, gelecekte bu noktalarda araya girmesi geren işlemler olduğunda senkronizasyon yapısına dokunmadan ve hata durumunda senkronizasyon işlemini durdurmayacak şekilde implemente edebilir olmak istemem.

### Hash Calculation Stratejisi

Hash algoritması olarak SHA256 seçmemin sebebi hız ve collision rate oranları. Buradaki kullanımda tahmin edilme gibi güvenlik edişelerinden çok eşsizlik aradığımız için her sistemde hazırda bulunan ve hızlı çalışan bir algoritma yeterli oldu.


#### Hangi Alanlar Hash'e Dahil

Hash hesaplamasına `name`, `price`, `stock` ve `description` alanlarını dahil ettim. PHPnin `json_encode` fonksiyonunu default flaglerle çalıştırdıktan sonra hash algoritmasına geçtim.

#### Dikat Ettiğim Noktalar
 - `price` alanının her zaman float olarak kullanılması: Provider bir sebele format değiştirip `99.99` olan fiyatı "99.9900" olarak göndermeye başlarsa bunu fiyat değişimi olarak algılamamak için
 - json_encode esnasında unicode karakterlerin encode edilmesi: Farklı charset/collation sebebiyle aynı karakterin farklı bit değerlerine dönüşmemesi için.

### Job Uniqueness Implementasyonu

Laravel'de hazırda bulunan `WithoutOverlapping` job middlewareini kullandım. Tek ürün, bir providerın ürünleri ve bütün providerların bütün ürünerini senkronize eden joblartanımladım. Tek ürün ve provider bazlı güncelleme yapam joblar lock key olarak `provider` içeren bir string kullanıyor. Bütün providerlar için çalışan job da zaten provider bazlı jobı tetiklediği için ayrıca kilitleme yapmadım.

#### Silinen Ürün Tespiti (Soft Delete)

Bizim sistemimizde olup da provider tarafında olmayan ürünleri silinmiş olarak varsayma ya da saymamayı seçenek olarak ekledim. Konfigürasyon üzerinden `providers.sync.soft_delete_missing` anahtarıyla kontrol ediliyor.

#### Idempotency
Ürünlerin eşsizliğini `provider_id` ve `external_id` üzerinden veritabanı seviyesinde garantilemek idempotency için yeterli oldu. Ürün verilerinin kaynağı her ürün bazında aynı olduğu için, aynı ürüne birden fazla kayıt açılmadıkça senkronizasyon işlemi birerbir aynı datayı veritabanındaki birebir aynı satırla eşleştiriyor. Kaç defa çalışırsa çalışsın elde edilen son durum aynı oluyor.

### Rate Limiting & Throttling

Provider tarafında rate limiting olduğunu varsayarak yaptığımız isteklere throttling uyguladım. "X sürede en fazla y adet request" şeklinde bir mantık uygulayarak parametreleri her provider için ayrı ayrı olacak şekilde `providers.gateways.<provider_type>.rate_tracking_period` ve `providers.gateways.<provider_type>.max_requests_per_period` config içinde, `<PROVIDER_TYPE>_DUMMYJSON_RATE_TRACKING_PERIOD` ve `<PROVIDER_TYPE>_MAX_REQUESTS_PER_PERIOD` olacak şekilde de env içinde tanımladım.

Throttling takibi için Laravel'in kendi cache sistemini ve arkasında da Redisi kullandım.

## API Dokümantasyonu

Backend API tanımları `backend/postman_collection.json` dosyasında mevcuttur.
