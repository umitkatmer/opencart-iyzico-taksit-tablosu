# İyzico Taksit Tablosu - OpenCart 3.x Kurulum Rehberi

Bu rehber, OpenCart 3.x için İyzico taksit tablosu fonksiyonunu nasıl kuracağınızı anlatır. Bu modül sadece taksit bilgilerini gösterir, ödeme işlemi yapmaz.

## Dosya Yapısı

### 1. Controller Dosyası - Taksit API'si
**Dosya Yolu:** `catalog/controller/product/product.php`

Bu dosyaya aşağıdaki fonksiyonları ekleyin:

```php
public function getInstallmentInfo() {
    // Ürün ID'sini istekte bekliyoruz
    if (isset($this->request->get['product_id'])) {
        $product_id = (int)$this->request->get['product_id'];
    } else {
        $json['success'] = false;
        $json['error'] = 'Ürün ID bulunamadı';
        $this->response->addHeader('Content-Type: application/json');
        $this->response->setOutput(json_encode($json));
        return;
    }

    // Bin numarası varsa alıyoruz (kredi kartının ilk 6 hanesi)
    $binNumber = isset($this->request->get['bin_number']) ? $this->request->get['bin_number'] : null;

    // Ürün bilgilerini alıyoruz
    $this->load->model('catalog/product');
    $product_info = $this->model_catalog_product->getProduct($product_id);
    if (!$product_info) {
        $json['success'] = false;
        $json['error'] = 'Ürün bilgisi alınamadı';
        $this->response->addHeader('Content-Type: application/json');
        $this->response->setOutput(json_encode($json));
        return;
    }

    // Fiyat: special varsa onun, yoksa normal fiyatın raw (sayısal) değeri
    $price = $product_info['special'] ? (float)$product_info['special'] : (float)$product_info['price'];

    try {
        // API bilgileri (modül ayarlarından alınmalı)
        $apiKey    = $this->config->get('payment_iyzico_api_key');
        $secretKey = $this->config->get('payment_iyzico_secret_key');
        // v2 API için production kullan
        $baseUrl   = 'https://api.iyzipay.com';

        // API isteği için gerekli parametreler
        $conversationId = uniqid() . '_' . time();
        $timestamp = date('Y-m-d H:i:s');
        $locale         = 'tr';

        // İstek gövdesi hazırlama
        $request = [
            'locale'         => $locale,
            'conversationId' => $conversationId,
            'price'          => number_format($price, 2, '.', ''),
            'currency'       => 'TRY' // Para birimi eklendi
        ];

        // Bin numarası varsa ekleyelim
        if ($binNumber) {
            $request['binNumber'] = $binNumber;
        }

        // JSON isteği oluşturma
        $jsonRequest = json_encode($request);

        // İyzico için random key oluşturma (timestamp based)
        $randomKey = $this->generateRandomKey();

        // URI path
        $uriPath = "/payment/iyzipos/installment";

        // İyzico v2 authorization oluşturma
        $authContent = $this->generateAuthContent($uriPath, $jsonRequest, $apiKey, $secretKey, $randomKey);

        // CURL isteği oluşturma
        $curl = curl_init();
        curl_setopt($curl, CURLOPT_URL, $baseUrl . "/payment/iyzipos/installment");
        curl_setopt($curl, CURLOPT_CUSTOMREQUEST, "POST");
        curl_setopt($curl, CURLOPT_POSTFIELDS, $jsonRequest);
        curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($curl, CURLOPT_HTTPHEADER, [
            "Authorization: " . $authContent,
            "Content-Type: application/json",
            "x-iyzi-rnd: " . $randomKey,
        ]);
        curl_setopt($curl, CURLOPT_CONNECTTIMEOUT, 30);
        curl_setopt($curl, CURLOPT_TIMEOUT, 60);
        curl_setopt($curl, CURLOPT_SSL_VERIFYPEER, true);
        curl_setopt($curl, CURLOPT_SSL_VERIFYHOST, 2);

        // İsteği gönder ve yanıtı al
        $response = curl_exec($curl);
        $httpStatus = curl_getinfo($curl, CURLINFO_HTTP_CODE);

        // CURL hatası kontrolü
        if ($response === false) {
            throw new Exception('CURL Hatası: ' . curl_error($curl));
        }

        curl_close($curl);

        // Yanıtı işleme
        $result = json_decode($response, true);

        if ($httpStatus != 200) {
            throw new Exception('API Hatası: HTTP ' . $httpStatus . ' - ' .
                (isset($result['errorMessage']) ? $result['errorMessage'] : 'Bilinmeyen hata'));
        }

        if (!isset($result['status']) || $result['status'] != 'success') {
            throw new Exception('API Yanıt Hatası: ' .
                (isset($result['errorMessage']) ? $result['errorMessage'] : 'Başarısız istek'));
        }

        // Taksit detaylarını işleme
        $json = $this->processInstallmentDetails($result, $price);

    } catch (Exception $e) {
        $json['success'] = false;
        $json['error'] = $e->getMessage();

        // Hata logla
        $this->log->write('İyzico Taksit Bilgisi Hatası: ' . $e->getMessage());
    }

    $this->response->addHeader('Content-Type: application/json');
    $this->response->setOutput(json_encode($json));
}

/**
 * Random key oluşturur (timestamp based)
 *
 * @return string Random key
 */
private function generateRandomKey() {
    // JavaScript versiyonuna uygun: timestamp + "123456789"
    return (string)(time() * 1000) . "123456789";
}

/**
 * Auth içeriği oluşturur (iyzico imza) - v2 format
 *
 * @param string $uriPath API endpoint path
 * @param string $requestData JSON request data
 * @param string $apiKey API Anahtarı
 * @param string $secretKey Gizli Anahtar
 * @param string $randomKey Random Key (timestamp based)
 * @return string Auth içeriği
 */
private function generateAuthContent($uriPath, $requestData, $apiKey, $secretKey, $randomKey) {
    // Payload oluştur: randomKey + uriPath + requestData
    $payload = $randomKey . $uriPath . $requestData;

    // HMAC-SHA256 ile şifrele
    $encryptedData = hash_hmac('sha256', $payload, $secretKey);

    // Authorization string oluştur
    $authorizationString = "apiKey:" . $apiKey .
                          "&randomKey:" . $randomKey .
                          "&signature:" . $encryptedData;

    // Base64 encode et
    $base64EncodedAuthorization = base64_encode($authorizationString);

    // IYZWSv2 prefix ekle
    return "IYZWSv2 " . $base64EncodedAuthorization;
}

/**
 * Taksit detaylarını işleyip formatlı hale getirir
 *
 * @param array $result API yanıtı
 * @param float $price Ürün fiyatı
 * @return array Formatlı taksit bilgileri
 */
private function processInstallmentDetails($result, $price) {
    $json = [];

    if (isset($result['installmentDetails']) && is_array($result['installmentDetails'])) {
        $installmentDetails = $result['installmentDetails'];
        $banks = [];
        $maxInstallment = 0;

        foreach ($installmentDetails as $detail) {
            $bankName = $detail['bankName'];
            $cardFamilyName = isset($detail['cardFamilyName']) ? $detail['cardFamilyName'] : '';
            $cardType = isset($detail['cardType']) ? $detail['cardType'] : '';
            $cardAssociation = isset($detail['cardAssociation']) ? $detail['cardAssociation'] : '';
            $force3ds = isset($detail['force3ds']) ? (bool)$detail['force3ds'] : false;
            $installmentPrices = $detail['installmentPrices'];

            // Banka logosu için slug oluşturma
            $bankSlug = $this->generateBankSlug($bankName);
            $logoPath = 'catalog/view/theme/default/image/banks/' . $bankSlug . '.png';

            $bankData = [
                'name'          => $bankName,
                'cardFamily'    => $cardFamilyName,
                'cardType'      => $cardType,
                'bankSlug'      => $bankSlug,
                'cardAssociation' => $cardAssociation,
                'force3ds'      => $force3ds,
                'logo'          => $logoPath,
                'installments'  => []
            ];

            foreach ($installmentPrices as $installmentPrice) {
                $installmentNumber = (int)$installmentPrice['installmentNumber'];
                $totalPrice = (float)$installmentPrice['totalPrice'];

                // Taksit oranı hesaplama (peşin fiyata göre yüzde artış)
                $installmentRate = 0;
                if ($price > 0 && $totalPrice > $price) {
                    $installmentRate = round((($totalPrice / $price) - 1) * 100, 2);
                }

                // Tek çekim (taksit=1) için toplam fiyatın ürün fiyatı ile aynı olduğundan emin olalım
                if ($installmentNumber == 1) {
                    $totalPrice = $price;
                    $installmentPrice['installmentPrice'] = $price;
                    $installmentRate = 0;
                }

                $bankData['installments'][$installmentNumber] = [
                    'totalPrice'       => $totalPrice,
                    'installmentPrice' => (float)$installmentPrice['installmentPrice'],
                    'installmentRate'  => $installmentRate,
                    'formattedTotalPrice' => $this->currency->format($totalPrice, $this->session->data['currency']),
                    'formattedInstallmentPrice' => $this->currency->format((float)$installmentPrice['installmentPrice'], $this->session->data['currency'])
                ];

                if ($installmentNumber > $maxInstallment) {
                    $maxInstallment = $installmentNumber;
                }
            }

            $banks[] = $bankData;
        }

        // Bankaları alfabetik sırala
        usort($banks, function($a, $b) {
            return strcmp($a['name'], $b['name']);
        });

        $json['success'] = true;
        $json['banks'] = $banks;
        $json['maxInstallment'] = $maxInstallment;
        $json['price'] = $price;
        $json['formattedPrice'] = $this->currency->format($price, $this->session->data['currency']);
    } else {
        $json['success'] = false;
        $json['error'] = 'Taksit bilgisi alınamadı';
    }

    return $json;
}

/**
 * Banka adından slug oluşturur
 *
 * @param string $bankName Banka adı
 * @return string Slug
 */
private function generateBankSlug($bankName) {
    $turkishChars = ['ı', 'ğ', 'ü', 'ş', 'ö', 'ç', 'İ', 'Ğ', 'Ü', 'Ş', 'Ö', 'Ç', ' ', '&'];
    $englishChars = ['i', 'g', 'u', 's', 'o', 'c', 'i', 'g', 'u', 's', 'o', 'c', '-', 'and'];

    $bankName = str_replace($turkishChars, $englishChars, $bankName);
    $bankName = preg_replace('/[^a-zA-Z0-9\-]/', '', $bankName);
    $bankName = strtolower(trim($bankName));
    $bankName = preg_replace('/-+/', '-', $bankName);

    return $bankName;
}
```

## Kurulum Adımları

### 1. Dosya Güncelleme
1. `catalog/controller/product/product.php` dosyasını açın
2. Yukarıdaki fonksiyonları dosyanın sonuna (son `}` işaretinden önce) ekleyin

### 2. API Ayarları
Taksit tablosu çalışması için aşağıdaki ayarları OpenCart admin panelinden yapmanız gerekir:

- **API Key:** `payment_iyzico_api_key`
- **Secret Key:** `payment_iyzico_secret_key`

Bu ayarlar genellikle İyzico ödeme modülü kurulduğunda otomatik olarak ayarlanır.

### 3. Frontend Entegrasyonu

Ürün sayfasında taksit tablosunu göstermek için JavaScript kodu:

```javascript
function loadInstallmentInfo(productId, binNumber = null) {
    var url = 'index.php?route=product/product/getInstallmentInfo&product_id=' + productId;
    if (binNumber) {
        url += '&bin_number=' + binNumber;
    }

    $.ajax({
        url: url,
        type: 'GET',
        dataType: 'json',
        success: function(response) {
            if (response.success) {
                displayInstallmentTable(response);
            } else {
                console.error('Taksit bilgisi alınamadı:', response.error);
            }
        },
        error: function(xhr, status, error) {
            console.error('AJAX Hatası:', error);
        }
    });
}

function displayInstallmentTable(data) {
    var html = '<div class="installment-table">';
    html += '<h3>Taksit Seçenekleri</h3>';

    data.banks.forEach(function(bank) {
        html += '<div class="bank-installments">';
        html += '<h4>' + bank.name + '</h4>';
        html += '<table class="table table-striped">';
        html += '<thead><tr><th>Taksit</th><th>Aylık Ödeme</th><th>Toplam</th><th>Oran</th></tr></thead>';
        html += '<tbody>';

        Object.keys(bank.installments).forEach(function(installmentNumber) {
            var installment = bank.installments[installmentNumber];
            html += '<tr>';
            html += '<td>' + installmentNumber + '</td>';
            html += '<td>' + installment.formattedInstallmentPrice + '</td>';
            html += '<td>' + installment.formattedTotalPrice + '</td>';
            html += '<td>%' + installment.installmentRate + '</td>';
            html += '</tr>';
        });

        html += '</tbody></table></div>';
    });

    html += '</div>';

    // Taksit tablosunu sayfaya ekle
    $('#installment-container').html(html);
}

// Sayfa yüklendiğinde taksit bilgilerini al
$(document).ready(function() {
    var productId = $('input[name="product_id"]').val();
    if (productId) {
        loadInstallmentInfo(productId);
    }
});
```

### 4. HTML Template Ekleme

Ürün template dosyasına (`catalog/view/theme/default/template/product/product.twig`) şu kodu ekleyin:

```html
<!-- Taksit Tablosu -->
<div id="installment-container" class="installment-section">
    <!-- Taksit bilgileri buraya yüklenecek -->
</div>
```

### 5. CSS Stilleri

```css
.installment-table {
    margin: 20px 0;
    padding: 15px;
    border: 1px solid #ddd;
    border-radius: 5px;
}

.bank-installments {
    margin-bottom: 20px;
}

.bank-installments h4 {
    margin-bottom: 10px;
    color: #333;
}

.installment-table table {
    width: 100%;
    margin-bottom: 15px;
}

.installment-table th {
    background-color: #f8f9fa;
    font-weight: bold;
    text-align: center;
}

.installment-table td {
    text-align: center;
    padding: 8px;
}
```

## Kullanım

Kurulum tamamlandıktan sonra:

1. Ürün sayfalarında otomatik olarak taksit tablosu görünecektir
2. API, ürün fiyatına göre mevcut taksit seçeneklerini gösterecektir
3. Farklı bankalar için farklı taksit oranları listelenecektir

## Önemli Notlar

- Bu modül sadece taksit bilgilerini gösterir, ödeme işlemi yapmaz
- İyzico API anahtarlarınızın doğru şekilde ayarlandığından emin olun
- Güvenlik için API anahtarlarını güvenli bir şekilde saklayın
- Test ortamında denedikten sonra canlı ortama geçin

## Sorun Giderme

**Taksit bilgisi alınamıyor:**
- API anahtarlarını kontrol edin
- İnternet bağlantısını kontrol edin
- Log dosyalarını inceleyin

**CORS Hatası:**
- AJAX isteklerinin aynı domain'den yapıldığından emin olun

**Boş taksit tablosu:**
- Ürün fiyatının 0'dan büyük olduğundan emin olun
- İyzico API'sinin çalıştığını kontrol edin
