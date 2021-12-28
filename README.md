# Cara menggunakan MinIO sebagai penyimpanan file pada [Laravel](https://laravel.com) 

`Laravel` memiliki sistem penyimpanan file yang dapat disesuaikan dengan kemampuan untuk membuat driver khusus untuknya. Dalam resep ini kami akan menerapkan driver sistem file khusus untuk menggunakan server Minio untuk mengelola file.

## 1. Prasyarat

Install Minio Server from [disini](https://www.minio.io/downloads.html).

## 2. Install Required Dependency untuk Laravel

Install `league/flysystem` package untuk [`aws-s3`](https://github.com/coraxster/flysystem-aws-s3-v3-minio)  :
fork based on https://github.com/thephpleague/flysystem-aws-s3-v3
```
composer require coraxster/flysystem-aws-s3-v3-minio
```


## 3. Buat MinIO Storage ServiceProvider 
Buat `MinioStorageServiceProvider.php` file pada `app/Providers/` dengan konten berikut:

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;

use Aws\S3\S3Client;
use League\Flysystem\AwsS3v3\AwsS3Adapter;
use League\Flysystem\Filesystem;
use Storage;

class MinioStorageServiceProvider extends ServiceProvider
{
    /**
     * Bootstrap the application services.
     *
     * @return void
     */
    public function boot()
    {
      Storage::extend('minio', function ($app, $config) {
          $client = new S3Client([
              'credentials' => [
                  'key'    => $config["key"],
                  'secret' => $config["secret"]
              ],
              'region'      => $config["region"],
              'version'     => "latest",
              'bucket_endpoint' => false,
              'use_path_style_endpoint' => true,
              'endpoint'    => $config["endpoint"],
          ]);
          $options = [
              'override_visibility_on_copy' => true
          ];
          return new Filesystem(new AwsS3Adapter($client, $config["bucket"], '', $options));
      });
    }

    /**
     * Register the application services.
     *
     * @return void
     */
    public function register()
    {

    }
}
```

Register service provider dengan menambahkan pada file berikut `config/app.php` pada bagian `providers` :  
```php
App\Providers\MinioStorageServiceProvider::class
```

Tambahkan konfigurasi untuk minio di bagian `disk` dari file `config/filesystems.php` :

```php
  'disks' => [
    // other disks

    'minio' => [
        'driver' => 'minio',
        'key' => env('MINIO_KEY', 'your minio server key'),
        'secret' => env('MINIO_SECRET', 'your minio server secret'),
        'region' => 'us-east-1',
        'bucket' => env('MINIO_BUCKET','your minio bucket name'),
        'endpoint' => env('MINIO_ENDPOINT','http://localhost:9000')
    ]

  ]
```  
Catatan: `region` tidak diperlukan & dapat diatur ke apa pun.

## 4. Gunakan Penyimpanan dengan Minio di Laravel
Sekarang Anda dapat menggunakan metode `disk` pada fasad penyimpanan untuk menggunakan driver minio :
```php
Storage::disk('minio')->put('avatars/1', $fileContents);
```
Atau Anda dapat mengatur driver cloud default ke `minio` di file konfigurasi `filesystems.php`:
```php
'cloud' => env('FILESYSTEM_CLOUD', 'minio'),
```