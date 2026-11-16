**NAMES**
Давайти правильні імена частинам в коді є важливо, бо може економити час. Це допомогає зрозуміти іншим як працює код та потратити на це відносно мало зусиль. Кожен клас, метод і перемінні мають свої правила для назв. Не потрібно боятися переназивати частину коду, якщо це допомагає покращити розуміння. Притримуся строгих назв, замість жартів, хоча це правило очевидне.
Адеквтаний неймінг буде працювати абсолютно всюди, я б навіть сказав в любій сфері житті, не тільки програмування.

**FUNCTIONS**
1. Функції повинні бути короткими, мати в собі лише один рівень абстракції, виконувати лише одну дію.
2. Організовувати функції потрібно в такому стилі, щоб вони могли читатися з гори в низ
е це не працює

# Де це не працює?#

Розбиття на короткі методи не буде працювати, якщо сценарії абстракій будуть переплітатися і буде непряма залежність. Тоді доведеться під час читання перепригувати від однієї функції до іншої декілька разів. Також не працює там, де логіка є лінійною, а також у критичних до продуктивності ділянках коду

Приклад 1: Розбиття лінійної логіки заставляти читати код по декілька разів, ускладнює читання
```
public HttpResponse sendOrderToErp(Order ord) {
    HttpRequest req = createBaseRequest();
    setRequestHeaders(req);
    String payload = serializeOrder(ord);
    req.setBody(payload);
    return executeRequest(req);
}

private HttpRequest createBaseRequest() {
    HttpRequest req = new HttpRequest();
    req.setEndpoint('callout:ERP_System/orders');
    req.setMethod('POST');
    return req;
}

private void setRequestHeaders(HttpRequest req) {
    req.setHeader('Content-Type', 'application/json');
    req.setHeader('Accept', 'application/json');
}

private String serializeOrder(Order ord) {
    return JSON.serialize(new Map<String, Object>{
        'orderId' => ord.Id,
        'amount' => ord.TotalAmount
    });
}

private HttpResponse executeRequest(HttpRequest req) {
    Http http = new Http();
    return http.send(req);
}

```

Приклад 2: Тут ми обчислюємо фінальну ціну товару зі знижками та податками. Усі кроки залежать від одних і тих самих проміжних змінних.

**Погано (передача купи параметрів):**  
Винесення розрахунків у дрібні функції змушує нас протягувати по 3–4 аргументи в кожен метод. Сигнатури методів стають довшими за саму логіку всередині.

```
public Decimal calculateFinalPrice(Decimal basePrice, Boolean isVip, String countryCode, Decimal promoDiscount) {
    Decimal priceAfterVip = applyVipDiscount(basePrice, isVip);
    Decimal priceAfterPromo = applyPromoCode(priceAfterVip, promoDiscount, isVip);
    return addTaxes(priceAfterPromo, countryCode, basePrice);
}

private Decimal applyVipDiscount(Decimal price, Boolean isVip) {
    return isVip ? price * 0.9 : price;
}

private Decimal applyPromoCode(Decimal currentPrice, Decimal promoDiscount, Boolean isVip) {
    if (isVip && promoDiscount > 10) {
        return currentPrice - 10; // VIP не може отримати більше 10 додаткової знижки
    }
    return currentPrice - promoDiscount;
}

private Decimal addTaxes(Decimal discountedPrice, String countryCode, Decimal originalPrice) {
    Decimal taxRate = (countryCode == 'UA') ? 0.20 : 0.15;
    // Податок рахується від базової ціни, якщо знижка завелика
    Decimal taxableBase = (discountedPrice < originalPrice * 0.5) ? originalPrice : discountedPrice;
    return discountedPrice + (taxableBase * taxRate);
}

```

```
public with sharing class DropboxService {

    private static final Integer CHUNK_SIZE = 4 * 1024 * 1024;

    private static HttpRequest createRequest(String path, String method, String body) {
        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:DropboxFiles'+path);
        req.setMethod(method);
        System.debug(body);
        req.setHeader('Content-Type', 'application/json');
        if (body != null) {
            req.setBody(body);
        }
        return req;
    }

    @AuraEnabled(cacheable=true)
    public static List<String> getDropboxItems(String path) {
        Http http = new Http();
        HttpRequest req = createRequest('/2/files/list_folder', 'POST', '{"path": "' + path + '"}');
        HttpResponse res = http.send(req);
        //Перевірка результату запиту. Сценраій 1
        if (res.getStatusCode() == 200) {
            System.debug(res.getBody());
            Map<String, Object> response = (Map<String, Object>) JSON.deserializeUntyped(res.getBody());
        // Витягнування entries з відповіді і мапінг. Сценраій 2
            List<Object> entries = (List<Object>) response.get('entries');
            List<String> items = new List<String>();
            for (Object entry : entries) {
                Map<String, Object> entryMap = (Map<String, Object>) entry;
                items.add((String) entryMap.get('name'));
            }
            return items;
        } else {
            //throw new AuraHandledException('Failed to fetch Dropbox items');
            System.debug(res.getBody());
            return null;
        }
    }

    // DropboxService.processUploadFile('068Hu00000bBYf4IAG','google.com');
    public static void processUploadFile(String fileId, String folderName){
    //Процесинг файлу файлу. Сценраій 1
        ContentVersion file = [SELECT Id, Title, VersionData, FileExtension, FileType FROM ContentVersion WHERE Id = :fileId LIMIT 1];
        System.debug(file);
        Blob csvBlob = file.VersionData;
        String csvContent = csvBlob.toString();
        folderName += '/'+file.Title+'.'+file.FileExtension;
        uploadFile(folderName, csvContent);
    }

    @AuraEnabled
    public static void uploadFile(String path, String content) {
    //Відправка файлу файлу. Сценраій 1
        Http http = new Http();
        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:Dropbox/2/files/upload');
        req.setMethod('POST');
        req.setHeader('Dropbox-API-Arg', '{"path": "' + path + '","mode": "add","autorename": true,"mute": false,"strict_conflict": false}');
        req.setHeader('Content-Type', 'application/octet-stream');
        req.setBody(content);
        HttpResponse res = http.send(req);
        if (res.getStatusCode() != 200) {
            throw new AuraHandledException('Failed to upload file to Dropbox');
        }
    }

    @AuraEnabled
    public static Blob downloadFile(String dropboxFilePath) {
    //Запит на сторонню систему. Сценраій 1
        HttpRequest req = new HttpRequest();
        if(String.isBlank(dropboxFilePath)){
            dropboxFilePath = '/google.com/system.jpeg';
        }
        req.setEndpoint('callout:Dropbox'+'/2/files/download');
        req.setMethod('POST');
        req.setHeader('Dropbox-API-Arg', '{"path": "' + dropboxFilePath + '"}');
        //req.setHeader('Authorization', 'Bearer тест токен');
        System.debug(req.getEndpoint());
        Http http = new Http();
        HttpResponse res = http.send(req);
        
        if (res.getStatusCode() == 200) {
        //Перевірка результату і інсерт даних в базу як ContentVersion. Сценраій 2
            System.debug(res.getBodyAsBlob());
            ContentVersion contentVersion_1 = new ContentVersion(
                Title='system', 
                PathOnClient ='system.jpeg',
                VersionData = res.getBodyAsBlob(), 
                origin = 'H',
                FirstPublishLocationId = '00QHu00003i3V7BMAU'
            );
            insert contentVersion_1;
            System.debug(contentVersion_1.Id);
            return res.getBodyAsBlob();
        } else {
            System.debug('Error: ' + res.getStatus());
            System.debug('Error Message: ' + res.getStatusCode());
            return null;
        }
    }

    @AuraEnabled
    public static void downloadZipFile(String dropboxFilePath, String zipFileName) {
        HttpRequest req = new HttpRequest();
        if(String.isBlank(dropboxFilePath)){
            dropboxFilePath = '/google.com';
        }
        req.setEndpoint('callout:Dropbox'+'/2/files/download_zip');
        req.setMethod('POST');
        req.setHeader('Dropbox-API-Arg', '{"path": "' + dropboxFilePath + '"}');
        //req.setHeader('Authorization', 'Bearer тест токен');
        System.debug(req.getEndpoint());
        Http http = new Http();
        HttpResponse res = http.send(req);
        
        if (res.getStatusCode() == 200) {
            
            ContentVersion contentVersion_1 = new ContentVersion(
                Title        =  zipFileName, 
                PathOnClient =  zipFileName,
                VersionData  =  res.getBodyAsBlob(), 
                origin       = 'H',
                FirstPublishLocationId = '00QHu00003i3V7BMAU'
            );
            insert contentVersion_1;
            System.debug(contentVersion_1.Id);
            
        } else {
            System.debug('Error: ' + res.getStatus());
            System.debug('Error Message: ' + res.getStatusCode());
            
        }
    }
}
```
1. `getDropboxItems`: які дії змішані в одному методі? Подивись на рядки 31–44. Назви 3 рівні (побудова запиту, перевірка статусу, розбір JSON). Відповідь: Побудова запиту , перевірка статусу, розбір JSON
2. `downloadFile` і `downloadZipFile`: знайди, що в них дублюється. Що станеться, якщо Dropbox змінить формат заголовка `Dropbox-API-Arg`? У скількох місцях треба правити? Відповідь: Обидва методи мають створення ContentVersion і це є дублюванням коду. Якщо Dropbox змінить формат, то треба буде змінювати це у всіх методах.
3. У `getDropboxItems` помилка Dropbox повертається як `null`, а в `downloadZipFile` взагалі нічого не повертає. Хто й як дізнається, що запит не вдався? Як це перегукується з принципом «функція не ховає помилки»? Відповідь: Функція повинна чесно говорити про помилку, а не приховувати її. Тобто для хендлінгу в LWC потрібно додати нотифікацію для юзера, це може бути toast, а для apex можна кидати кастомну помилку, щоб повідомити про проблему.
4. Рядок 31: `'{"path": "' + path + '"}'` будує JSON конкатенацією. Що станеться, якщо `path` містить лапку? Яка вбудована альтернатива? Відповідь: Якщо path містить лапки, то буде помилка побудови JSON. Щоб уникнути помилки, треба використати JSON.serialize, він дозволить побудувати json безпечно.
5. Рядок 58: `csvBlob.toString()` у `processUploadFile`. Що буде з бінарним файлом, наприклад зображенням? Чому ім'я змінної й розширення в `folderName` вводять в оману? Відповідь: Бінарний файл, типу зображення викличе помилку, бо UTF-8 очікує читабельні символи. Не знаю чому імена вводять в оману.
6. Клас `with sharing`, але `insert ContentVersion` не перевіряє CRUD/FLS. Це те саме, що ти вже побачив в `EasyCompareController`. Чи доречний тут `with sharing`? Відповідь: Можна просто використовувати User Mode в квері, щоб переконатися, що доступ до полів і обєкту буде безпечний.
7. Рядки 102 і 135: жорстко заданий Id `00Q…`, а в `downloadFile` ще й шлях /google.com/system.jpeg за замовчуванням. Що це за рішення з погляду «Meaningful Names» і ролі сервісу? Відповідь: Конфігурацію можна перенести в Custom Metadata Types, це дозволить для кожної орги мати відповідне налаштування.
8. `callout:DropboxFiles` у `createRequest` і `callout:Dropbox` в інших методах: чому два різні Named Credentials? І чому `uploadFile` взагалі не використовує `createRequest`? Відповідь: Це виглядає як помилка, які потібно виправляти.