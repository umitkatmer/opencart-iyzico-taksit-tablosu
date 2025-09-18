# İyzico Taksit Tablosu - OpenCart 3.x

OpenCart 3.x için İyzico taksit tablosu ekleme rehberi. Bu sistem Journal3 temasında test edilmiş ve çalışır durumda. Diğer temalar için de uyarlanabilir.

## Özellikler

- ✅ İyzico API v2 entegrasyonu
- ✅ Gerçek zamanlı taksit hesaplama
- ✅ Banka bazlı taksit oranları
- ✅ Responsive tasarım
- ✅ AJAX ile hızlı yükleme
- ✅ Bin numarası desteği (opsiyonel)

## Kurulum

### 1. Controller Dosyası Güncelleme

`catalog/controller/product/product.php` dosyasına aşağıdaki fonksiyonları ekleyin:

```php
public function getInstallmentInfo() {
    if (isset($this->request->get['product_id'])) {
        $product_id = (int)$this->request->get['product_id'];
    } else {
        $json['success'] = false;
        $json['error'] = 'Ürün ID bulunamadı';
        $this->response->addHeader('Content-Type: application/json');
        $this->response->setOutput(json_encode($json));
        return;
    }

    $binNumber = isset($this->request->get['bin_number']) ? $this->request->get['bin_number'] : null;

    $this->load->model('catalog/product');
    $product_info = $this->model_catalog_product->getProduct($product_id);
    if (!$product_info) {
        $json['success'] = false;
        $json['error'] = 'Ürün bilgisi alınamadı';
        $this->response->addHeader('Content-Type: application/json');
        $this->response->setOutput(json_encode($json));
        return;
    }

    $price = $product_info['special'] ? (float)$product_info['special'] : (float)$product_info['price'];

    try {
        $apiKey    = $this->config->get('payment_iyzico_api_key');
        $secretKey = $this->config->get('payment_iyzico_secret_key');
        $baseUrl   = 'https://api.iyzipay.com';

        $request = [
            'locale'         => 'tr',
            'conversationId' => uniqid() . '_' . time(),
            'price'          => number_format($price, 2, '.', ''),
            'currency'       => 'TRY'
        ];

        if ($binNumber) {
            $request['binNumber'] = $binNumber;
        }

        $jsonRequest = json_encode($request);
        $randomKey = $this->generateRandomKey();
        $uriPath = "/payment/iyzipos/installment";
        $authContent = $this->generateAuthContent($uriPath, $jsonRequest, $apiKey, $secretKey, $randomKey);

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

        $response = curl_exec($curl);
        $httpStatus = curl_getinfo($curl, CURLINFO_HTTP_CODE);

        if ($response === false) {
            throw new Exception('CURL Hatası: ' . curl_error($curl));
        }

        curl_close($curl);
        $result = json_decode($response, true);

        if ($httpStatus != 200) {
            throw new Exception('API Hatası: HTTP ' . $httpStatus);
        }

        if (!isset($result['status']) || $result['status'] != 'success') {
            throw new Exception('API Yanıt Hatası');
        }

        $json = $this->processInstallmentDetails($result, $price);

    } catch (Exception $e) {
        $json['success'] = false;
        $json['error'] = $e->getMessage();
        $this->log->write('İyzico Taksit Bilgisi Hatası: ' . $e->getMessage());
    }

    $this->response->addHeader('Content-Type: application/json');
    $this->response->setOutput(json_encode($json));
}

private function generateRandomKey() {
    return (string)(time() * 1000) . "123456789";
}

private function generateAuthContent($uriPath, $requestData, $apiKey, $secretKey, $randomKey) {
    $payload = $randomKey . $uriPath . $requestData;
    $encryptedData = hash_hmac('sha256', $payload, $secretKey);
    $authorizationString = "apiKey:" . $apiKey . "&randomKey:" . $randomKey . "&signature:" . $encryptedData;
    $base64EncodedAuthorization = base64_encode($authorizationString);
    return "IYZWSv2 " . $base64EncodedAuthorization;
}

private function processInstallmentDetails($result, $price) {
    $json = [];

    if (isset($result['installmentDetails']) && is_array($result['installmentDetails'])) {
        $banks = [];
        $maxInstallment = 0;

        foreach ($result['installmentDetails'] as $detail) {
            $bankData = [
                'name' => $detail['bankName'],
                'installments' => []
            ];

            foreach ($detail['installmentPrices'] as $installmentPrice) {
                $installmentNumber = (int)$installmentPrice['installmentNumber'];
                $totalPrice = (float)$installmentPrice['totalPrice'];

                $installmentRate = 0;
                if ($price > 0 && $totalPrice > $price) {
                    $installmentRate = round((($totalPrice / $price) - 1) * 100, 2);
                }

                if ($installmentNumber == 1) {
                    $totalPrice = $price;
                    $installmentPrice['installmentPrice'] = $price;
                    $installmentRate = 0;
                }

                $bankData['installments'][$installmentNumber] = [
                    'totalPrice' => $totalPrice,
                    'installmentPrice' => (float)$installmentPrice['installmentPrice'],
                    'installmentRate' => $installmentRate,
                    'formattedTotalPrice' => $this->currency->format($totalPrice, $this->session->data['currency']),
                    'formattedInstallmentPrice' => $this->currency->format((float)$installmentPrice['installmentPrice'], $this->session->data['currency'])
                ];

                if ($installmentNumber > $maxInstallment) {
                    $maxInstallment = $installmentNumber;
                }
            }
            $banks[] = $bankData;
        }

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

### 2. Frontend Entegrasyonu

Taksit tablosunu ürün sayfasında göstermek için aşağıdaki kodları tema dosyalarınıza ekleyin.

#### 2.1 Template Dosyası Düzenleme

`catalog/view/theme/journal3/template/product/product.twig` dosyasının sonuna ({{ footer }} satırından önce) şu kodu ekleyin:

```html
<script>

 $(document).ready(function() {
    var productId = {{ product_id }};
    var isMobile = $(window).width() <= 768;

    if (isMobile) {
        // Journal3 ürün detayları ekstra tab'larının sınıfı
        var mobileContainer = $('.panel-group.product_extra');
        var mobileTabId = 'mobile_product_tabs-installments';

        // Yeni accordion panel ekle
        var installmentPanel = '<div class="panel panel-default">';
        installmentPanel += '<div class="panel-heading">';
        installmentPanel += '<h4 class="panel-title">';
        installmentPanel += '<a data-toggle="collapse" href="#' + mobileTabId + '">TAKSİTLER</a>';
        installmentPanel += '</h4></div>';
        installmentPanel += '<div id="' + mobileTabId + '" class="panel-collapse collapse">';
        installmentPanel += '<div class="panel-body text-center"><i class="fa fa-spinner fa-spin fa-2x"></i><p>Taksit bilgileri yükleniyor...</p></div>';
        installmentPanel += '</div></div>';

        mobileContainer.append(installmentPanel);
    } else {
        // Journal3 ürün detayları desktop tab'larının sınıfı
        var tabsContainer = $('.tabs-container.product_extra.product_tabs');
        var tabList = tabsContainer.find('ul.nav-tabs');
        var tabContent = tabsContainer.find('.tab-content');
        var tabId = 'product_tabs-installments';

        // TAKSİTLER adında yeni tab ekliyoruz
        tabList.append('<li><a href="#' + tabId + '" data-toggle="tab">TAKSİTLER</a></li>');
        // Taksit tabının içeriği için container ekliyoruz
        tabContent.append('<div class="product_extra-installments tab-pane" id="' + tabId + '"><div class="installment-content text-center"><i class="fa fa-spinner fa-spin fa-2x"></i><p>Taksit bilgileri yükleniyor...</p></div></div>');
    }

    $.ajax({
        url: 'index.php?route=product/product/getInstallmentInfo',
        type: 'GET',
        data: { product_id: productId },
        dataType: 'json',
        success: function(data) {
            if (data.success) {
                var html = createInstallmentContent(data);
                if (isMobile) {
                    $('#' + mobileTabId + ' .panel-body').html(html);
                } else {
                    $('#' + tabId).html(html);
                }
            } else {
                var errorHtml = '<div class="alert alert-danger">Taksit bilgileri alınamadı.</div>';
                if (isMobile) {
                    $('#' + mobileTabId + ' .panel-body').html(errorHtml);
                } else {
                    $('#' + tabId).html(errorHtml);
                }
            }
        },
        error: function() {
            var errorHtml = '<div class="alert alert-danger">Taksit bilgileri yüklenemedi.</div>';
            if (isMobile) {
                $('#' + mobileTabId + ' .panel-body').html(errorHtml);
            } else {
                $('#' + tabId).html(errorHtml);
            }
        }
    });

    function createInstallmentContent(data) {
        var html = '<div class="container">';
        html += '<div class="row">';

        data.banks.forEach(function(bank) {
            html += '<div class="col-xs-12 col-sm-4">';
            html += '<div class="bank-card">';
            if (bank.logo) {
                html += '<div class="bank-logo"><img src="' + bank.logo + '" alt="' + bank.name + '"></div>';
            }
            html += '<div class="bank-name">' + bank.name + '</div>';
            html += '<table class="table table-bordered">';
            html += '<thead><tr><th>Taksit</th><th>Tutar</th><th>Toplam</th></tr></thead><tbody>';

            for (var i = 1; i <= data.maxInstallment; i++) {
                var option = bank.installments[i] || { formattedInstallmentPrice: '-', formattedTotalPrice: '-' };
                html += '<tr><td>' + i + '</td><td>' + option.formattedInstallmentPrice + '</td><td>' + option.formattedTotalPrice + '</td></tr>';
            }

            html += '</tbody></table></div></div>'; // bank-card, col-4
        });

        html += '</div></div>'; // row, container
        return html;
    }

    $('<style>').prop('type', 'text/css').html(`
        .bank-card { border-radius: 5px; padding: 10px; margin-bottom: 15px; text-align: center; background: #fff; }
        .bank-logo img { max-height: 30px; margin-bottom: 5px; }
        .bank-name { font-weight: bold; font-size: 14px; margin-bottom: 5px; }
        .table { font-size: 12px; }
        .table th, .table td { padding: 1px; text-align: center; }
        .table td { padding: 4px !important; }
    `).appendTo('head');
});

</script>
```

#### 2.2 Kod Açıklaması

**ÖNEMLİ NOTLAR:**


🖥️ **Desktop için:**
- `.tabs-container.product_extra.product_tabs` → Journal3'te ürün detayda göstermek için TAKSİTLER diye bir yeni tab açın ve ön tarafdan onun sınıfını js içine ekleyin eğer aynı ise hiçbirşey yapmanıza gerekyok
 

⚡ **Otomatik çalışır:**
- JavaScript kodu otomatik olarak HTML'i oluşturur
- Ekstra HTML dosyası eklemenize gerek yoktur
- CSS stilleri de kod içinde dahildir

#### 2.3 CSS Stilleri

CSS stilleri JavaScript kodu içinde inline olarak eklenir. Ekstra CSS dosyası eklemenize gerek yoktur.

**JavaScript içindeki CSS:**
- `.bank-card` - Banka kartı tasarımı
- `.bank-logo` - Banka logoları için
- `.bank-name` - Banka adı stilleri
- `.table` - Taksit tablosu stilleri

### 3. Banka Logoları (Opsiyonel)

Banka logolarını göstermek için `catalog/view/theme/default/image/banks/` klasörüne logo dosyalarını ekleyin:

- `akbank.png`
- `garanti-bbva.png`
- `isbank.png`
- `yapikredi.png`

## Gereksinimler

- OpenCart 3.x
- İyzico API anahtarları
- cURL desteği

## API Ayarları

Admin panelinden şu ayarları yapın:
- `payment_iyzico_api_key`
- `payment_iyzico_secret_key`

## Kurulum Sonrası

1. **Controller fonksiyonlarını** `catalog/controller/product/product.php` dosyasına ekleyin
2. **JavaScript kodunu** ürün sayfası template dosyanıza ekleyin
3. **HTML container'ını** uygun yere yerleştirin
4. **CSS stillerini** tema dosyanıza ekleyin
5. **API anahtarlarını** admin panelinden ayarlayın

Kurulum tamamlandıktan sonra ürün sayfalarında taksit tablosu otomatik görünecektir.

## Önemli Notlar

⚠️ **Bu modül sadece taksit bilgilerini gösterir, ödeme işlemi yapmaz**

- Test ortamında deneyin
- API anahtarlarını güvenli saklayın
- Production ortamında SSL kullanın

## Sorun Giderme

**Taksit bilgisi alınamıyor:**
- API anahtarlarını kontrol edin
- İnternet bağlantısını test edin

**Boş taksit tablosu:**
- Ürün fiyatının 0'dan büyük olduğunu kontrol edin
- İyzico API durumunu kontrol edin

## İletişim

**ÜMİT KATMER**
- Email: info@umitkatmer.com.tr
- Website: https://umitkatmer.com.tr

## Destek

Bu kod işinize yaradı ise, yardım kuruluşlarına bağış yaparak destek olabilirsiniz:

- [LÖSEV - Lösemili Çocuklar Vakfı](https://www.losev.org.tr)
- [TSK Dayanışma Vakfı](https://onlineodeme.tskdv.org.tr/DAYVAKOB/)
- [AHBAP](https://ahbap.org)
