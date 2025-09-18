# İyzico Taksit Tablosu - OpenCart 3.x Kurulum Rehberi

Bu repo, OpenCart 3.x için **İyzico taksit tablosu** fonksiyonunun nasıl kurulacağını anlatır.  
⚠️ Bu modül sadece taksit bilgilerini gösterir, ödeme işlemi yapmaz.

---

## 📂 Dosya Yapısı

### 1. Controller Dosyası - Taksit API'si
**Dosya Yolu:** `catalog/controller/product/product.php`

```php
// Burada fonksiyonlar eklenecek
// Örnek: getInstallmentInfo(), generateRandomKey(), generateAuthContent(), processInstallmentDetails(), generateBankSlug()
