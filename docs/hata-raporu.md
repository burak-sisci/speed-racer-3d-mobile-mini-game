# Speed Racer 3D - Hata Raporu ve Duzeltmeler

Tarih: 2026-03-20
Durum: Tum hatalar duzeltildi

---

## KRITIK HATALAR (Oyunu Bozan)

### 1. HTML'de Cift class Attribute - shop-buy-btn

**Dosya:** `index.html` satir 182 (eski)
**Sorun:** Satin Al butonunda iki ayri `class` attribute kullanilmis:
```html
<!-- HATALI -->
<button class="screen-btn orange" id="shop-buy-btn" class="hidden" disabled>
```
HTML standardina gore bir element uzerinde ayni attribute birden fazla kullanilamaz. Tarayici yalnizca ilk `class` degerini okur, ikincisini (`hidden`) tamamen yok sayar.

**Etki:** Satin Al butonu her zaman gorunur durumda kaliyordu. Magaza ekraninda sahip olunan araclar icin bile "SATIN AL" butonu gosteriliyordu. Butonun `hidden` class'i hicbir zaman uygulanmiyordu.

**Cozum:**
```html
<!-- DUZELTILMIS -->
<button class="screen-btn orange hidden" id="shop-buy-btn" disabled>
```
Her iki class tek bir attribute icinde birlestirildi.

---

### 2. Mobil Kontroller HTML'den Eksik

**Dosya:** `index.html`
**Sorun:** Son buyuk guncellemede (magaza, arac sistemi eklentisi) HTML tamamen yeniden yazilirken mobil kontrol butonlari (`btn-up`, `btn-down`, `btn-left`, `btn-right`, `btn-nitro`) HTML body'den cikarilmis.

**Etki:**
- CSS'te `#mobile-controls` stilleri hala tanimli ama HTML elementi yok
- JS'te `setupMobileBtn()` fonksiyonu null kontrolu yapip sessizce donuyor (`if (!btn) return`)
- Mobil/tablet kullanicilar oyunu hic oynayamiyor — dokunmatik kontroller gorunmuyor

**Cozum:** Mobil kontrol HTML blogu `</body>` oncesine, script tag'lerinin hemen ustune yeniden eklendi:
```html
<div id="mobile-controls">
    <div class="mobile-row" style="justify-content:center;">
        <div class="mobile-btn" id="btn-up">...</div>
    </div>
    ...
</div>
```

---

### 3. WebGL / Three.js Hata Yonetimi Eksik

**Dosya:** `index.html`
**Sorun:** Three.js CDN'den yuklenemezse veya tarayici WebGL desteklemiyorsa, sayfa bos beyaz ekran gosteriyordu. Hicbir hata mesaji veya geri bildirim yoktu.

**Etki:**
- Eski tarayicilarda (IE, eski mobil) beyaz ekran
- Internet baglantisi yoksa veya CDN engelliyse beyaz ekran
- Kullanici sorunun ne oldugunu anlayamiyor

**Cozum:** Three.js script tag'inden hemen sonra, oyun kodundan once iki kontrol eklendi:
```javascript
// 1. Three.js yukleme kontrolu
if (!window.THREE) {
    // Turkce hata mesaji goster
    throw new Error('Three.js failed to load');
}

// 2. WebGL destek kontrolu
(function() {
    var gl = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');
    if (!gl) {
        // Turkce hata mesaji goster
        throw new Error('WebGL not supported');
    }
})();
```

---

## ORTA ONCELIKLI SORUNLAR

### 4. Satin Al Butonu Gorunurluk Mantigi Tutarsiz

**Dosya:** `index.html` - `renderShop()` fonksiyonu
**Sorun:** Magaza UI'inda Satin Al butonunun gizlenmesi icin `style.display` kullaniliyordu, ancak butonun baslangic durumu CSS class (`hidden`) ile kontrol ediliyordu. Bu iki yontem birbiriyle catisiyordu.

**Etki:** Buton bazen `display: none` (inline style) bazen `hidden` class ile gizleniyordu. `!important` olan `.hidden` class'i ile inline style arasinda oncelik catismasi olusabiliyordu.

**Cozum:** `style.display` kullanimi kaldirildi, tutarli sekilde `classList.add/remove('hidden')` kullanildi:
```javascript
if (def.owned) {
    buyBtn.classList.add('hidden');
} else {
    buyBtn.classList.remove('hidden');
}
```

---

### 5. Shadow X Aracinin Magaza Rengi Gorunmuyor

**Dosya:** `index.html` - CAR_DEFS
**Sorun:** Shadow X aracinin rengi `0x1a1a1a` (cok koyu gri, neredeyse siyah). Magaza kartlarinda koyu arka plan uzerinde bu renk neredeyse gorunmuyor.

**Etki:** Kullanici magaza ekraninda Shadow X aracinin rengini goremez, bos bir kart gibi gorunur.

**Oneri:** Gelecekte box-shadow veya border ile vurgulanabilir. Suan oynanabilirligi etkilemiyor.

---

## DUSUK ONCELIKLI SORUNLAR / IYILESTIRMELER

### 6. requestAnimationFrame Tab Arkada Kalinca Yavaslar

**Sorun:** Tarayicilar, arka plandaki tab'larda `requestAnimationFrame` cagrilarini saniyede ~1'e dusurur. Kullanici tab degistirip geri donerse, `clock.getDelta()` buyuk bir `dt` degeri doner ve fizik hesaplari bozulabilir.

**Mevcut Onlem:** `const dt = Math.min(clock.getDelta(), 0.05)` ile dt degeri sinirlandirilmis. Bu yeterli bir koruma sagliyor.

---

### 7. localStorage File Protokolunde Calismayabilir

**Sorun:** Bazi tarayicilarda `file://` protokolunde `localStorage` erisimi engellenebilir. Bu durumda altin ve arac kayitlari korunmaz.

**Mevcut Onlem:** `loadSave()` fonksiyonu `try-catch` blogu icinde. Hata olursa varsayilan degerlerle devam eder. Oyun calismaya devam eder, sadece kayit tutulmaz.

---

### 8. CDN Bagimliligi

**Sorun:** Three.js `cdnjs.cloudflare.com` uzerinden yukleniyor. CDN erisimi yoksa oyun calismaz.

**Oneri:** Gelecekte `three.min.js` dosyasi projeye dahil edilebilir (offline calisma icin).

---

## TEST ORTAMI NOTLARI

### Headless Browser Sinirliliklari

Preview ortaminda (headless Chromium) su sinirliliklar tespit edildi:
- **WebGL rendering calismaz** — screenshot alinamiyor
- **requestAnimationFrame tetiklenmez** — oyun dongusu baslamaz
- **Script scope closure'lari eval'dan erisilemez** — `let`/`const` degiskenler global degil

Bu sinirliliklar nedeniyle oyun yalnizca gercek bir tarayicida test edilebilir. Headless ortamda yalnizca:
- Sayfa yuklenmesi (OK)
- Three.js CDN yuklenmesi (OK)
- DOM yapisi (OK)
- Fonksiyon tanimlari (OK)
dogrulanabildi.

---

## DUZELTME OZETI

| # | Hata | Oncelik | Durum |
|---|------|---------|-------|
| 1 | Cift class attribute | Kritik | Duzeltildi |
| 2 | Mobil kontroller eksik | Kritik | Duzeltildi |
| 3 | WebGL/Three.js hata yonetimi | Kritik | Duzeltildi |
| 4 | Buton gorunurluk tutarsizligi | Orta | Duzeltildi |
| 5 | Shadow X renk gorunurlugu | Dusuk | Not edildi |
| 6 | rAF tab arkasi davranisi | Dusuk | Mevcut onlem yeterli |
| 7 | localStorage file:// sorunu | Dusuk | Mevcut onlem yeterli |
| 8 | CDN bagimliligi | Dusuk | Gelecek iyilestirme |
