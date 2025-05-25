## Web Üzerinden Dinamik Python Komutları Yazma Rehberi (Detaylı)

Bu rehber, Telegram userbot'unuza web sitenizdeki "Python Komutu Ekle" bölümünü kullanarak kendi özel komutlarınızı nasıl yazacağınızı kapsamlı bir şekilde açıklar. Bu dinamik komutlar, botun ana kod yapısını değiştirmeden, hızlı ve esnek bir şekilde yeni işlevler eklemenizi sağlar. Ancak, bu esneklik bazı önemli güvenlik kısıtlamaları ve farklı bir çalışma mantığı ile birlikte gelir.

### 📜 Komutların Çalışma Ortamı: Güvenlik Kalkanı ve Kısıtlamalar!

Web üzerinden eklediğiniz Python kodları, ana bot motorunuz (`bot_core/bot_runner.py`) tarafından özel bir "güvenlikli" (sandboxed) ortamda çalıştırılır. Bu, `exec()` fonksiyonu kullanılarak gerçekleştirilir ve botunuzun genel stabilitesi ile sunucunuzun güvenliğini korumak için tasarlanmıştır. Bu ortamın getirdiği temel kısıtlamalar şunlardır:

1.  **`import` Yasağı (En Önemli Kural!):**
    *   Kodunuzun içinde `import requests` veya `import datetime` gibi ifadelerle harici veya standart Python kütüphanelerini **kullanamazsınız**.
    *   Bu kural, `__builtins__.__import__` fonksiyonunun bu ortamda kısıtlanmış olması nedeniyle `ImportError: __import__ not found` hatası almanıza yol açar.
    *   **Tek İstisna `asyncio`:** `asyncio` modülü, bot motoru tarafından size global bir değişken olarak zaten sağlanmıştır. Bu nedenle kodunuzun başında `import asyncio` yazmanıza **gerek yoktur** ve doğrudan `await asyncio.sleep(0.5)` gibi kullanabilirsiniz. Eğer script'inizde `import asyncio` yazarsanız, bu da `ImportError` hatasına neden olacaktır!

2.  **`len()` Fonksiyonu Yok:**
    *   Bir string'in, listenin veya başka bir koleksiyonun uzunluğunu ölçmek için kullanılan standart Python `len()` fonksiyonu bu ortamda **kullanılamaz**.
    *   Kullanmaya çalıştığınızda `NameError: name 'len' is not defined` hatası alırsınız.
    *   **Alternatif:** Bir şeyin uzunluğunu veya eleman sayısını bulmak için manuel döngüler ve sayaçlar kullanmanız gerekir.
        ```python
        # Örnek: Manuel karakter sayımı (len kullanmadan)
        my_string = "merhaba"
        count = 0
        for character in my_string:
            count = count + 1
        # print("Karakter sayısı:", count) # Bu print sunucu loguna gider
        ```

3.  **`Exception` Sınıfları ve Özel Hata Yakalama Yok:**
    *   `try...except ValueError as e:` veya `try...except Exception as e:` gibi belirli hata türlerini yakalamak için standart `Exception` sınıfları (ve türevleri) bu ortamda **kullanılamaz**.
    *   Bunları kullanmaya çalıştığınızda `NameError: name 'Exception' is not defined` (veya benzeri) hatası alırsınız.
    *   **Alternatif:** Genel bir hata yakalama bloğu olan `try...except:` (isimsiz `except`) kullanabilirsiniz. Bu, herhangi bir hatayı yakalar ancak hatanın türü veya detayları hakkında bilgi vermez. Bu nedenle, hata ayıklama için `print()` ile loglama yapmak çok daha önemli hale gelir.
        ```python
        # Örnek: Genel hata yakalama
        try:
            # Riskli olabilecek kodunuz
            risky_value = int("abc") # Bu bir ValueError üretir
        except: # Hata türü belirtilmiyor
            print("!KOMUT_ADI: Kod çalışırken bir hata oluştu! Detaylar için logları kontrol et.")
            await reply("Beklenmedik bir sorun oluştu.")
        ```

4.  **Diğer Dahili Fonksiyonlar (`__builtins__`) ve Tipler:**
    *   `__builtins__` sözlüğü bu ortamda büyük ölçüde boşaltıldığı veya kısıtlandığı için, `list()`, `dict()`, `sum()`, `map()`, `filter()` gibi birçok diğer standart dahili Python fonksiyonu ve tipi de kullanılamayabilir.
    *   **Temel Tipler ve Dönüşümler:** `str()`, `int()`, `float()` gibi temel tip dönüşüm fonksiyonlarının ve temel aritmetik işlemlerin (`+`, `-`, `*`, `/`) genellikle çalıştığı gözlemlenmiştir. Ancak bu, `bot_runner.py` içindeki `exec_globals` yapılandırmasına bağlıdır. Her zaman en kısıtlı durumu varsayarak kod yazmak ve bolca test etmek en iyisidir.
    *   Eğer bir `NameError` alıyorsanız, kullandığınız fonksiyonun veya tipin bu ortamda mevcut olmadığını varsayabilirsiniz.

### 🛠️ Kullanabileceğiniz Araçlar ve Dinamik Veriler

Bu kısıtlı ortama rağmen, komutlarınızı işlevsel kılmak için size sunulan bazı güçlü araçlar ve dinamik veriler vardır:

1.  **`event` Objesi (Komutun Kalbi):**
    *   Bu obje, komutunuzu tetikleyen Telegram mesajı ve bağlamı hakkında tüm hayati bilgileri içerir.
    *   **Sık Kullanılan Özellikler:**
        *   `event.text`: Kullanıcının gönderdiği mesajın tam metni (örn: `!senin_komutun argüman1 argüman2`).
        *   `event.raw_text`: `event.text` ile genellikle aynıdır, mesajın ham metnini içerir.
        *   `event.sender_id`: Komutu gönderen kullanıcının benzersiz Telegram ID'si. `{userid}` gibi bir placeholder doğrudan yoktur, bu bilgiyi `event.sender_id` üzerinden alırsınız.
        *   `event.chat_id`: Komutun gönderildiği sohbetin (grup, özel mesaj, kanal) benzersiz ID'si.
        *   `event.message.id`: Gönderilen komut mesajının benzersiz ID'si.
        *   `event.message.reply_to_msg_id`: Eğer komutunuz bir başka mesaja yanıt olarak yazıldıysa, yanıt verilen o orijinal mesajın ID'si. Eğer bir yanıtlama durumu yoksa bu değer `None` (veya benzeri bir boş değer) olacaktır.
        *   `event.is_private`: Komut özel mesajda mı gönderildi? (`True` veya `False`).
        *   `event.is_group`: Komut bir grupta mı gönderildi? (`True` veya `False`).
        *   `event.message.date`: Mesajın gönderildiği tarih ve saat (datetime objesi olabilir, ancak formatlamak için `datetime` modülüne erişiminiz olmayabilir).
    *   **Argüman Ayrıştırma:** Komutunuza gönderilen argümanları `event.text` üzerinden manuel olarak ayrıştırmanız gerekir.
        ```python
        # Örnek: Argüman ayrıştırma
        full_text = event.text # Örn: "!islem_yap Ali 35"
        parts = full_text.split(" ", 2) # Komutu ve ilk iki argümanı ayır
                                        # "!islem_yap", "Ali", "35" 
                                        # veya "!islem_yap", "SadeceBirArgüman"
                                        # veya "!islem_yap"
        
        command_part = ""
        arg1 = ""
        arg2 = ""
        
        # parts listesinin uzunluğunu len() olmadan kontrol etme
        num_parts = 0
        for p_item in parts:
            num_parts = num_parts + 1
            
        if num_parts > 0:
            command_part = parts[0]
        if num_parts > 1:
            arg1 = parts[1]
        if num_parts > 2:
            arg2 = parts[2]
            
        # print(f"Komut: {command_part}, Arg1: {arg1}, Arg2: {arg2}")
        ```

2.  **`await reply("mesaj")` Fonksiyonu (Kullanıcıyla İletişim):**
    *   Kullanıcının komutuna yanıt olarak Telegram'a yeni bir mesaj göndermenizi sağlar.
    *   Bu asenkron bir fonksiyondur, bu yüzden her zaman `await reply(...)` şeklinde kullanılmalıdır.
    *   **Markdown/HTML:** Bu `reply` fonksiyonunun Markdown veya HTML formatlamayı destekleyip desteklemediği, `bot_runner.py` içindeki tanımına bağlıdır. Genellikle bu dinamik ortamda `parse_mode` ayarı yapılamaz, bu yüzden düz metin olarak yanıt vermek en güvenlisidir. `event.edit` için de aynı durum geçerlidir.

3.  **`await event.edit("yeni mesaj")` Fonksiyonu (Mesaj Düzenleme):**
    *   Userbot'un gönderdiği komut mesajını düzenleyerek yanıt vermesini sağlar. Animasyonlu komutlar için idealdir.
    *   Bu da asenkron bir fonksiyondur: `await event.edit(...)`.

4.  **`asyncio` Objesi (Asenkron İşlemler):**
    *   Özellikle animasyonlar veya sıralı işlemler arasında bekleme (gecikme) yaratmak için `asyncio.sleep()` fonksiyonu kullanılır.
    *   Örnek: `await asyncio.sleep(0.5)` (0.5 saniye bekle).
    *   Unutmayın, script başında `import asyncio` **YAZMAYIN!** `asyncio` zaten globalde mevcuttur.

5.  **`print("log mesajı")` Fonksiyonu (Hata Ayıklama Aracı):**
    *   Bu fonksiyon, yazdığınız mesajları kullanıcıya **göndermez**. Bunun yerine, botunuzun çalıştığı sunucunun konsoluna (veya log dosyalarına) yazar.
    *   Komutunuzun hangi adımlardan geçtiğini, değişkenlerin anlık değerlerini veya potansiyel hata noktalarını anlamak için paha biçilmezdir.
    *   Log formatı genellikle şöyledir: `[PyCmd ID:X] Log mesajınız` (X, komutun veritabanı ID'sidir).

### ✍️ Komut Yazarken Dikkat Edilmesi Gerekenler (Detaylı)

1.  **`import` Kesinlikle Yok!** (Tekrar ve tekrar: `asyncio` hariç).
2.  **`len()` Yerine Manuel Sayım/Kontrol:** Döngüler ve sayaçlarla çalışmaya alışın.
3.  **Genel `except:` ile Hata Yakalama:** Hata olabileceğini düşündüğünüz her bloğu `try...except:` içine alın ve `except:` bloğunda `print()` ile detaylı loglama yapın.
4.  **Kod Basitliği ve Okunabilirlik:** Kısıtlı bir ortamda karmaşık kodlar yazmak hata yapma olasılığını artırır. Kodunuzu olabildiğince basit, anlaşılır ve adım adım ilerleyen bir yapıda tutun. Değişkenlerinize açıklayıcı isimler verin.
5.  **Bol ve Açıklayıcı `print()` Logları:**
    *   Komutun başına `print("!KOMUT_ADI: Başlatıldı.")` ekleyin.
    *   Önemli değişkenlerin değerlerini `print(f"!KOMUT_ADI: DegiskenX = {degiskenX_str}")` şeklinde loglayın (değişkeni `str()` ile stringe çevirmeye çalışın, eğer `str()` çalışmıyorsa manuel bir yol bulun).
    *   `try...except` bloklarının hem `try` kısmına girildiğini hem de `except` kısmına düşüldüğünü loglayın.
    *   `await reply()` veya `await event.edit()` çağrılarından hemen önce ve sonra log ekleyerek bu işlemlerin başarılı olup olmadığını kontrol edin.
6.  **Her Zaman `await`:** `reply`, `event.edit` ve `asyncio.sleep` asenkron olduğu için `await` ile çağrılmalıdır.
7.  **Kullanıcı Girdilerine Güvenmeyin:** `event.text` üzerinden aldığınız argümanları işlerken, kullanıcının her zaman beklediğiniz formatta veya türde veri girmeyebileceğini varsayın. Basit kontroller yapmaya çalışın (örn: bir sayıyı `int()` ile çevirmeye çalışmadan önce metnin sayısallığını kabaca kontrol etmek).
8.  **Performans ve Bloklama:** Çok uzun süren döngüler veya işlemler yazmaktan kaçının. Eğer bir döngü çok fazla iterasyon yapacaksa ve her adımda `await` içeren bir işlem yoksa, botun genel yanıt verme hızını yavaşlatabilir. Gerekirse uzun işlemleri parçalara ayırın veya aralara kısa `await asyncio.sleep(0)` ekleyerek diğer eventlerin işlenmesine izin verin (bu, event loop'u rahatlatır).
9.  **Kullanıcı Deneyimi:**
    *   Komut bir argüman bekliyorsa ve kullanıcı argüman girmemişse, "Kullanım: `!komut <gerekli_argüman>`" gibi açıklayıcı bir yanıt verin.
    *   Başarılı işlemlerden sonra kullanıcıya olumlu bir geri bildirim verin.
    *   Hata durumlarında genel ama anlaşılır bir hata mesajı gösterin ("Bir sorun oluştu, lütfen daha sonra tekrar deneyin." gibi).

### 🚀 Örnek Komutlar: Temelden İleriye (Kısıtlamalarla Uyumlu)

Aşağıdaki örnekler, yukarıda detaylandırılan kısıtlamalar (özellikle `len` ve `Exception` olmaması, `import` yasağı) dikkate alınarak yazılmıştır.

**1. En Temel: Merhaba Dünya!**
*   **Amaç:** Sistemin çalışıp çalışmadığını ve `reply` fonksiyonunu test etmek.
*   **Tetikleyici (Web Arayüzü):** `!merhaba_py`
*   **Python Kodu:**
    ```python
    # Bu kod web arayüzüne girilecek
    print("!merhaba_py: Komut başlatıldı.")
    try:
        await reply("Merhaba! Ben web'den eklenmiş dinamik bir Python komutuyum! 👋")
        print("!merhaba_py: Yanıt başarıyla gönderildi.")
    except:
        print("!merhaba_py: 'reply' fonksiyonu çalışırken bir hata oluştu.")
    ```

**2. Kullanıcı ve Sohbet ID'si Gösterme**
*   **Amaç:** `event` objesinden temel bilgileri alıp kullanıcıya göstermek. `str()` fonksiyonunun çalıştığını varsayar.
*   **Tetikleyici:** `!kimlik_py`
*   **Python Kodu:**
    ```python
    # Bu kod web arayüzüne girilecek
    print("!kimlik_py: Komut başlatıldı.")
    sender_id_str = "Alınamadı"
    chat_id_str = "Alınamadı"
    message_id_str = "Alınamadı"

    try:
        sender_id_str = str(event.sender_id)
    except:
        print("!kimlik_py: event.sender_id alınamadı veya str() ile çevrilemedi.")
    
    try:
        chat_id_str = str(event.chat_id)
    except:
        print("!kimlik_py: event.chat_id alınamadı veya str() ile çevrilemedi.")

    try:
        message_id_str = str(event.message.id)
    except:
        print("!kimlik_py: event.message.id alınamadı veya str() ile çevrilemedi.")

    response = "--- Senin Bilgilerin ---
"
    response = response + "Kullanıcı Telegram ID: " + sender_id_str + "
"
    response = response + "Bu Sohbetin ID: " + chat_id_str + "
"
    response = response + "Bu Mesajın ID: " + message_id_str
    
    try:
        await reply(response)
        print("!kimlik_py: Kimlik bilgileri gönderildi.")
    except:
        print("!kimlik_py: Yanıt gönderilirken hata oluştu.")
    ```

**3. Argüman Ayrıştırma ve Kullanma: Basit Echo Komutu**
*   **Amaç:** Kullanıcıdan argüman alıp onu tekrarlamak. `split()` metodunun çalıştığını varsayar.
*   **Tetikleyici:** `!soyle_py`
*   **Python Kodu:**
    ```python
    # Bu kod web arayüzüne girilecek
    print("!soyle_py: Komut başlatıldı.")
    full_text = event.text
    text_to_echo = ""
    arg_found = False
    
    try:
        parts = full_text.split(" ", 1) # Komut ve geri kalanını ayır
        
        num_parts = 0
        for _ in parts: # len() olmadan parça sayısını bul
            num_parts = num_parts + 1
            
        if num_parts > 1:
            text_to_echo = parts[1] # Argümanı al
            if text_to_echo: # Argüman boş değilse (boş string False döner)
                arg_found = True
        
        print("!soyle_py: Argüman ayrıştırma denendi. Argüman bulundu: " + str(arg_found) + ", Metin: '" + text_to_echo + "'")

    except:
        print("!soyle_py: Argüman ayrıştırma sırasında bir hata oluştu.")
        await reply("Mesajınızı işlerken bir sorunla karşılaştım.")

    if arg_found:
        await reply("Dedin ki: " + text_to_echo)
        print("!soyle_py: '" + text_to_echo + "' tekrarlandı.")
    else: # Ya argüman yok ya da ayrıştırmada sorun oldu ve text_to_echo boş kaldı
        await reply("Kullanım: !soyle_py <söylenecek metin>")
        print("!soyle_py: Argüman girilmedi veya boş.")
    ```

**4. Manuel Karakter Sayacı (Boşluklar Dahil)**
*   **Amaç:** `len()` olmadan bir metnin karakter sayısını bulmak.
*   **Tetikleyici:** `!say_char_py`
*   **Python Kodu:**
    ```python
    # Bu kod web arayezüne girilecek
    print("!say_char_py: Komut başlatıldı.")
    full_text = event.text
    text_to_count = ""
    arg_found_for_count = False
    
    try:
        parts = full_text.split(" ", 1)
        num_parts = 0
        for _ in parts: num_parts = num_parts + 1
        if num_parts > 1:
            text_to_count = parts[1]
            if text_to_count: arg_found_for_count = True
        print("!say_char_py: Argüman ayrıştırma denendi. Argüman: '" + text_to_count + "'")
    except:
        print("!say_char_py: Argüman ayrıştırma hatası.")
        await reply("Mesajınızı işlerken bir sorun oldu.")

    if arg_found_for_count:
        char_count = 0
        for character_item in text_to_count: # String üzerinde döngü
            char_count = char_count + 1
        
        try:
            char_count_str = str(char_count) # Sayıyı stringe çevir
            await reply("Girdiğiniz metinde (boşluklar dahil) " + char_count_str + " karakter var.")
            print("!say_char_py: Karakter sayısı: " + char_count_str)
        except:
            print("!say_char_py: char_count str() ile çevrilemedi veya reply hatası.")
            await reply("Karakter sayısını hesapladım ama gösterirken bir sorun oldu.")
    else:
        await reply("Kullanım: !say_char_py <sayılacak metin>")
        print("!say_char_py: Argüman girilmedi veya boş.")
    ```

**5. Animasyonlu Metinler (Mesaj Düzenleme ile Yükleme Barı)**
*   **Amaç:** `event.edit` ve `asyncio.sleep` kullanarak basit bir animasyon oluşturmak.
*   **Tetikleyici:** `!yukle_bar_py`
*   **Python Kodu:** (Daha önce çalıştığını belirttiğin, `import asyncio` içermeyen versiyon)
    ```python
    # Bu kod web arayezüne girilecek
    # asyncio globalde var, import etme.
    print("!yukle_bar_py: Komut başlatıldı.")
    try:
        bar_width = 10       # Barın toplam genişliği (daha kısa tutalım)
        filled_char = "▓"    # Dolu kısmı gösteren karakter
        empty_char = "░"     # Boş kısmı gösteren karakter
        current_fill = 0
        loop_continues = True
        
        # İlk mesajı gönder/düzenle
        initial_bar = "["
        temp_i = 0
        while temp_i < bar_width:
            initial_bar = initial_bar + empty_char
            temp_i = temp_i + 1
        initial_bar = initial_bar + "]"
        await event.edit("Yükleniyor... " + initial_bar)
        await asyncio.sleep(0.2) # Başlangıçta kısa bir bekleme

        while loop_continues:
            bar_display_string = "["
            fill_iterator = 0
            while fill_iterator < current_fill:
                bar_display_string = bar_display_string + filled_char
                fill_iterator = fill_iterator + 1
            
            empty_count = bar_width - current_fill
            empty_iterator = 0
            while empty_iterator < empty_count:
                bar_display_string = bar_display_string + empty_char
                empty_iterator = empty_iterator + 1
            
            bar_display_string = bar_display_string + "]"
            
            # Yüzdeyi hesaplama (int ve str çalıştığını varsayarak)
            percentage_str = "0"
            try:
                # Basit yüzde hesaplama, float kullanmadan
                if bar_width > 0 : # Sıfıra bölme hatasını engelle
                    percentage_val = (current_fill * 100) // bar_width # Tam sayı bölme
                    percentage_str = str(percentage_val)
            except:
                print("!yukle_bar_py: Yüzde hesaplama/çevirme hatası.")
                percentage_str = "Hata"


            await event.edit("Yükleniyor... " + percentage_str + "% " + bar_display_string)
            await asyncio.sleep(0.4) # Animasyon hızı
            
            current_fill = current_fill + 1
            if current_fill > bar_width: # Döngüden çıkış koşulu
                loop_continues = False

        final_bar_string = "["
        final_fill_iterator = 0
        while final_fill_iterator < bar_width:
            final_bar_string = final_bar_string + filled_char
            final_fill_iterator = final_fill_iterator + 1
        final_bar_string = final_bar_string + "]"
        
        await event.edit("✅ Yükleme Tamamlandı! %100 " + final_bar_string)
        print("!yukle_bar_py: Yükleme barı animasyonu tamamlandı.")

    except: # Genel bir hata yakalama
        print("!yukle_bar_py: Animasyon sırasında genel bir sorun oluştu.")
        try:
            await event.edit("⚠️ Animasyon Hatası Oluştu!")
        except:
            print("!yukle_bar_py: Hata mesajı dahi gönderilemedi/düzenlenemedi.")
    ```

### 🚷 İleri Düzey Eylemler ve AŞILAMAYAN Duvarlar: Kick, Ban, Profil Yönetimi vb.

Bu konu çok net bir şekilde anlaşılmalıdır: **Web arayüzünden eklediğiniz bu dinamik Python komutları ile doğrudan kullanıcıları bir gruptan atamaz (kick), yasaklayamaz (ban), susturamaz (mute), yönetici yapamaz veya kullanıcıların/grupların detaylı profil bilgilerini (üye listesi, izinler vb.) aktif olarak yönetemezsiniz.**

**Neden Kesinlikle Yapılamaz?**

*   **`TelegramClient` Erişimi Yok:** Bu tür moderasyon ve yönetim eylemleri (kick, ban, izinleri düzenleme, profil fotoğrafı güncelleme, üye listesini çekme vb.), Telegram API'sinde özel yetkiler gerektirir ve `TelegramClient` objesinin doğrudan metotlarının (örn: `client.kick_participant(...)`, `client.edit_admin_rights(...)`, `client.get_participants(...)`, `client.edit_photo(...)`) çağrılmasını zorunlu kılar.
*   **Güvenlik İzolasyonu:** Web'den eklenen komutların çalıştığı güvenlikli `exec()` ortamı, bu güçlü `client` objesine veya bu tür kritik metotlara **kesinlikle erişim vermez**. Bu, kasıtlı ve çok önemli bir güvenlik önlemidir. Eğer bu erişim olsaydı, web arayüzünden eklenecek kötü niyetli bir Python kodu ile:
    *   Tüm botun kontrolü ele geçirilebilir.
    *   Sahip olduğunuz tüm gruplarda/kanallarda istenmeyen eylemler (herkesi atma, spam yapma vb.) yapılabilir.
    *   Userbot hesabınızın güvenliği tehlikeye atılabilir.
*   **`event` Objesi Yetersiz:** `event` objesi üzerinden gelen bilgiler (`event.sender_id`, `event.chat_id` gibi) sadece temel tanımlayıcılardır ve bazı durum bilgilerini içerir. Bunlar üzerinden doğrudan "eylem yapma" (kick, ban vb.) fonksiyonları çağrılamaz. `event.client` özelliği bu ortamda genellikle kısıtlıdır veya yoktur.

**Peki Bu Tür İşlemler İçin Çözüm Ne?**

Bu tür güçlü moderasyon ve yönetim işlemleri için, projenizin ana kod yapısında bulunan **"Sabit Komutlar"** sistemini kullanmanız gerekmektedir. Yani `bot_core/commands/` dizini altına, daha önce `echo.py` örneğinde olduğu gibi, yeni `.py` dosyaları ekleyerek oluşturduğunuz komutlar bu işlevleri yerine getirebilir.

*   Bu "sabit" komutların `handler` ve `register` fonksiyonları, `bot_runner.py` tarafından kendilerine geçirilen `client` objesine (veya `event.client` üzerinden Telethon client'ına) tam erişime sahiptir.
*   Örneğin, bir `bot_core/commands/kick_user.py` dosyası oluşturup içine Telethon'un kullanıcı atma fonksiyonlarını kullanan bir Python kodu yazabilirsiniz. Bu komut, bot yeniden başlatıldığında otomatik olarak yüklenir ve güvenli bir şekilde çalışır.

**Özetle:** Web'den eklenen dinamik Python komutları; metin tabanlı işlemler, basit otomasyonlar, `event` objesinden veri gösterme, API olmayan basit hesaplamalar ve eğlenceli metin animasyonları için mükemmeldir. Ancak, Telegram üzerinde yetki gerektiren eylemler (üye yönetimi, grup ayarları, profil güncellemeleri vb.) için **kesinlikle uygun değildirler ve kullanılamazlar**. Bu tür ihtiyaçlar için ana projenizin "sabit komut" mekanizmasını kullanmalısınız.

### 💡 Ek İpuçları ve En İyi Uygulamalar

*   **Test, Test, Test:** Yazdığınız her komutu farklı senaryolarla (boş argüman, beklenmedik argüman, uzun metinler vb.) test edin.
*   **Logları Takip Edin:** Komutlarınız çalışmıyorsa veya beklenmedik davranıyorsa, ilk bakacağınız yer sunucu loglarınız olmalı. `print()` ifadeleri orada size yol gösterecektir.
*   **Adım Adım Geliştirin:** Karmaşık bir komut yazıyorsanız, önce en basit halini çalıştırın, sonra adım adım yeni özellikler ekleyin. Her adımdan sonra test edin.
*   **Yorum Satırları Kullanın:** Kodunuzun ne yaptığını açıklayan yorum satırları (`# Bu satır şunu yapar`) eklemek, hem sizin hem de başkalarının kodu daha sonra anlamasına yardımcı olur.
*   **Kullanıcı Dostu Yanıtlar:** Komutlarınızın kullanıcıya verdiği yanıtların açık, anlaşılır ve yardımcı olmasına özen gösterin.

Bu detaylı dokümantasyonun, web arayüzünden güvenli ve etkili Python komutları yazma yolculuğunuzda size kapsamlı bir rehber olacağını umuyorum! Unutmayın, bu ortamın kısıtlamaları aslında sizi daha yaratıcı ve dikkatli kod yazmaya teşvik edebilir. 
