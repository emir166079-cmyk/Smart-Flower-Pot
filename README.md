# MicroPython Kodu (ESP32 üzerinde çalışacak)
import machine
import time

# Pin tanımlamaları (Donanımınıza göre değişebilir)
pin_nem = machine.Pin(34, machine.Pin.IN) # Analog Pin
pin_isik = machine.Pin(35, machine.Pin.IN) # Analog Pin

# DHT sensörü için örnek (Işık ve nem sensörünü okuduğumuzu varsayalım)
# import dht
# d = dht.DHT22(machine.Pin(4)) 

def sensor_verisi_oku():
    # Nem okuma (0-4095 arası değer)
    nem_adc = machine.ADC(pin_nem).read()
    
    # Işık okuma (0-4095 arası değer)
    isik_adc = machine.ADC(pin_isik).read()
    
    # Sıcaklık okuma (Örnek olarak sabit değer verelim, gerçekte DHT'den okunacak)
    sicaklik = 25.0
    
    # Veriyi bir sözlük (JSON) formatında hazırlama
    veri = {
        "nem": nem_adc,
        "isik": isik_adc,
        "sicaklik": sicaklik
    }
    return veri

def veriyi_sunucuya_gonder(veri):
    # Bu kısım Wi-Fi bağlantısı kurma ve HTTP POST isteği gönderme kodunu içerir.
    # Örneğin: Urequests kütüphanesi ile bir API'ye JSON verisi göndermek.
    print(f"Sunucuya gönderilen veri: {veri}") 

while True:
    okunan_veri = sensor_verisi_oku()
    veriyi_sunucuya_gonder(okunan_veri)
    time.sleep(300) # Her 5 dakikada bir oku
    # Python Kodu (Sunucu Tarafı - Karar Mekanizması)

def ideal_degerleri_tanimla(bitki_adi):
    """Kullanıcının seçtiği bitki türüne göre ideal değerleri döndürür."""
    # Bu kısım bir veritabanı veya sabit bir sözlük olabilir (TÜBİTAK projesinin ana özgün noktalarından biri!)
    ideal_degerler = {
        "Orkide": {"min_nem": 60, "min_isik": 500, "min_sicaklik": 18},
        "Fesleğen": {"min_nem": 40, "min_isik": 1000, "min_sicaklik": 15},
        # ... Diğer bitkiler ...
    }
    return ideal_degerler.get(bitki_adi, {"min_nem": 50, "min_isik": 700, "min_sicaklik": 20}) # Varsayılan

def karar_ver(okunan_veri, bitki_adi="Fesleğen"):
    """Okunan veri ile ideal değerleri karşılaştırır ve uyarı mesajı oluşturur."""
    ideal = ideal_degerleri_tanimla(bitki_adi)
    uyarilar = []

    # 1. Nem Kontrolü (Nem yüzdesine dönüştürüldüğünü varsayalım)
    okunan_nem = okunan_veri.get("nem")
    if okunan_nem < ideal["min_nem"]:
        uyarilar.append(f"Toprağınız çok kuru ({okunan_nem}%), lütfen sulayın.")
    if okunan_nem > 90:
    uyarilar.append("Toprak çok ıslak, biraz kurumasını bekleyin.")

    # 2. Işık Kontrolü (Lux değerine dönüştürüldüğünü varsayalım)
    okunan_isik = okunan_veri.get("isik")
    if okunan_isik < ideal["min_isik"]:
        uyarilar.append(f"Işık ihtiyacı azaldı ({okunan_isik} Lux), pencereye yaklaştırın.")
    
    # 3. Sıcaklık Kontrolü
    okunan_sicaklik = okunan_veri.get("sicaklik")
    if okunan_sicaklik < ideal["min_sicaklik"]:
        uyarilar.append(f"Ortam çok soğuk ({okunan_sicaklik}°C), daha sıcak bir yere taşıyın.")
    
    if not uyarilar:
        return "Her şey yolunda görünüyor! Çiçeğiniz mutlu :)"
    else:
        # Tüm uyarıları birleştirme
        return "\n".join(uyarilar)

# --- Uygulama Örneği ---
# Bu veri ESP32'den gelmiş olsun (Nem ve Işık için 0-100 arasında bir yüzdeye çevrildiğini varsayalım)
gelen_veri = {
    "nem": 35,      # Çok az nemli
    "isik": 450,    # Az ışık
    "sicaklik": 22  # Sıcaklık ideal
}

mesaj = karar_ver(gelen_veri, bitki_adi="Fesleğen")
print("\n--- Akıllı Saksı Mesajı ---\n")
print(mesaj)

# Sesli Arayüz (Ek Adım): 
# Eğer sesli arayüz isterseniz, bu "mesaj" değişkenini kullanacak bir kütüphane (örneğin gTTS) ile ses dosyasına çevirmeniz gerekir.
# Örnek:
# from gtts import gTTS
# tts = gTTS(text=mesaj, lang='tr')
# tts.save("uyari.mp3")
# MicroPython Kodu (ESP32)



import machine
import time
import urequests
import ujson
import network

# Wi-Fi bilgilerini gir
SSID = "WiFi_ADIN"
PASSWORD = "WiFi_SIFREN"

# Sunucu IP ve port
SUNUCU = "http://SUNUCU_IP_ADRESI:5000/api/veri"

# Wi-Fi Bağlantısı
wifi = network.WLAN(network.STA_IF)
wifi.active(True)
wifi.connect(SSID, PASSWORD)
# Wi-Fi bağlanana kadar bekle
while not wifi.isconnected():
    print("Wi-Fi bağlanıyor...")
    time.sleep(1)

print("Wi-Fi bağlantısı başarılı:", wifi.ifconfig())

# Pin tanımlamaları
adc_nem = machine.ADC(machine.Pin(34))
adc_isik = machine.ADC(machine.Pin(35))

def sensor_verisi_oku():
    nem_adc = adc_nem.read()
    isik_adc = adc_isik.read()
    sicaklik = 25.0  # Sabit değer, DHT sensörü ile değiştirilebilir

    # ADC değerlerini % ve Lux gibi normalize et
    nem_yuzde = int((nem_adc / 4095) * 100)
    isik_lux = int((isik_adc / 4095) * 1000)

    veri = {
        "nem": nem_yuzde,
        "isik": isik_lux,
        "sicaklik": sicaklik
    }
    return veri

def veriyi_sunucuya_gonder(veri):
    try:
        response = urequests.post(
            SUNUCU,
            data=ujson.dumps(veri),
            headers={"Content-Type": "application/json"}
        )
        print("Sunucu yanıtı:", response.text)
        response.close()
    except Exception as e:
        print("Bağlantı hatası:", e)

# Ana döngü
while True:
    okunan_veri = sensor_verisi_oku()
    print("Okunan veri:", okunan_veri)
    veriyi_sunucuya_gonder(okunan_veri)
    time.sleep(300)  # Her 5 dakikada bir veri gönder

