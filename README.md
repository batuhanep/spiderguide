# Web Scraping & Spider Guide (Python + Scrapy)

> **Amaç:** Sıfırdan başlayıp, HTML’i okuyabilen, XPath/CSS seçicilerle veri çıkaran, linkleri takip edip sayfalar arası gezebilen (crawl) bir **Scrapy** örümceği (spider) kurmak. Bu rehberi doğrudan GitHub repoya **`WEB_SCRAPING_SPIDER_GUIDE.md`** olarak koyabilirsin.

---

## İçindekiler
1. Ön Koşullar ve Kurulum
2. HTML’e Hızlı Giriş
3. XPath Seçiciler: Temelden İleriye
4. CSS Seçiciler ve XPath ↔ CSS Dönüşümleri
5. Attribute ve Metin (Text) Çıkarma
6. Scrapy Selector ile Çalışma (Standalone)
7. Response Nesnesiyle Çalışma ve Link Takibi
8. İlk Örümceğini Yaz: CrawlerProcess ile Çalıştırma
9. Crawl: Linkleri Çek, Sayfaları Gez, Veri Topla
10. Mini Capstone: Kurs → Bölüm Başlıkları Sözlüğü
11. Kontrol Listesi (Debug / İpucu)
12. Sık Karşılaşılan Hatalar ve Çözümler
13. Ek Ödevler ve Geliştirmeler

---

## Ön Koşullar ve Kurulum

### 1) Python ve Sanal Ortam
```bash
python -V          # 3.10+ önerilir
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
```

### 2) Scrapy Kurulumu
```bash
pip install scrapy requests parsel
```
> `parsel`, Scrapy’nin altında da kullanılan selektör kütüphanesidir; HTML üzerinde XPath ve CSS kullanacağız.

### 3) PyCharm İpucu
- Yeni bir **Virtualenv** proje oluştur; **Interpreter** olarak `.venv`’i seç.
- Run/Debug konfigürasyonunda çalıştıracağın dosyayı seç (ör. `spider_dc.py`).

---

## HTML’e Hızlı Giriş
HTML ağaç yapısında etiketler (tags) içerir. En temel etiketlerden bazıları: `html`, `body`, `div`, `p`, `a`. Elementlerin **attribute**’ları vardır: `id` (benzersiz olmalı), `class` (birden fazla elemana verilebilir), `href` (link hedefi), `src` (görsel kaynağı) vb.

**Öğrendiklerimiz:**
- Ağaç mantığıyla ebeveyn/çocuk (parent/child) ve kardeş (sibling) kavramları.
- `<a>` bağlantı etiketi ve `href`.
- `id` benzersiz, `class` tekrar edebilir.

> Bu bölümdeki kavramlar ileride XPath/CSS ile hedef seçiminde kritik olacak.

---

## XPath Seçiciler: Temelden İleriye

### Temel Semboller
- `/` : Bir alt nesile git (tek adım)
- `//` : Aşağı doğru tüm nesillerde ara
- `[]` : Kardeşler arasından indeksleme veya filtreleme (`p[2]` gibi)
- `*` : Joker (herhangi bir tag)
- `@attr` : Attribute erişimi (`@href`, `@class`, `@id`)
- `contains(@attr, "parça")` : Attribute içinde parça eşleştirme

### Örnekler
```xpath
/html/body/p             # body altındaki tüm p'ler
/html/body/div/p[2]      # body>div altındaki ikinci p
//p                      # sayfadaki tüm p'ler
/html/body/*             # body altındaki tüm çocuklar
//*[@id="uid"]          # id'si uid olan herhangi bir element
//div[@id="uid"]/p[2]   # id=uid div'i altındaki ikinci p
//*[contains(@class,"class-1")] # class içinde class-1 geçenler
//div[@id="uid"]/a/@href       # href attribute değerleri
```

**Not:** `'/html/body'` ile `'/html[1]/body[1]'` aynı hedefi verir. XPath’ta indeksler 1’den başlar.

---

## CSS Seçiciler ve XPath ↔ CSS Dönüşümleri

### Hızlı Eşleştirmeler
- `XPath /`  → `CSS >` (ilk karakter hariç)
- `XPath //` → `CSS`’te araya boşluk (descendant) veya `>` (child) kullan
- `XPath [N]` → `CSS :nth-of-type(N)`

**Örnek dönüşümler**
```
XPath: /html/body//div/p[2]
CSS  : html > body div > p:nth-of-type(2)
```

### Sık Kullanılan CSS Kalıpları
```css
p.class-1                /* class'ı class-1 olan p'ler */
div#uid                  /* id'si uid olan div */
.class1                  /* class1 sınıfına sahip tüm elemanlar */
div#uid > p.class1       /* id=uid div'inin altındaki p.class1 */
```

### CSS ile Attribute/Metin
```css
a::attr(href)            /* a etiketlerinin href değeri */
p#p-example::text        /* p'nin sadece doğrudan metin çocukları */
p#p-example ::text       /* p ve altındaki tüm metin düğümleri */
```

---

## Attribute ve Metin (Text) Çıkarma

### XPath ile Attribute
```xpath
//div[@id='uid']/a/@href
```

### CSS ile Attribute
```css
div#uid > a::attr(href)
```

### Metin Çıkarma
- XPath `text()` yalnızca **doğrudan** metinleri döndürür.
- `//text()` torun metinleri de kapsar.
- CSS’te `::text` doğrudan metin; ` ::text` (boşlukla) torunları da içerir.

```python
sel.xpath("//p[@id='p-example']/text()").extract()
sel.xpath("//p[@id='p-example']//text()").extract()
sel.css("p#p-example::text").extract()
sel.css("p#p-example ::text").extract()
```

---

## Scrapy Selector ile Çalışma (Standalone)
Bazen tam proje kurmadan da seçici pratikleri yapmak isteyebilirsin.

```python
from scrapy import Selector
html = """
<html>
  <body>
    <div class="hello datacamp">
      <p>Hello World!</p>
    </div>
    <p>Enjoy DataCamp!</p>
  </body>
</html>
"""
sel = Selector(text=html)

# XPath
sel.xpath("//p")                # SelectorList döner
sel.xpath("//p").extract()      # ['<p>Hello World!</p>', '<p>Enjoy DataCamp!</p>']
sel.xpath("//p").extract_first()# '<p>Hello World!</p>'

# CSS
sel.css("div > p").extract()    # ['<p>Hello World!</p>']

# requests ile gerçek sayfa -> Selector
import requests
url = "https://en.wikipedia.org/wiki/Web_scraping"
html = requests.get(url).content
sel = Selector(text=html)
```

> **İpucu:** `SelectorList` indekslenebilir (`ps[1]` gibi). `extract()` HTML string döndürür; yalnızca metin istiyorsan önce `::text`/`text()` ile node’ları hedefle.

---

## Response Nesnesiyle Çalışma ve Link Takibi
Scrapy spider içinde `response` nesnesi **Selector** gibi davranır: `response.xpath(...)`, `response.css(...)`, ardından `extract()` / `extract_first()` ile veri alırsın.

Ek avantajlar:
- **`response.url`**: Şu anki sayfanın URL’i.
- **`response.follow(next_url, callback=...)`**: İlgili linki takip edip sonucu başka bir parse fonksiyonuna gönderir.

```python
# zincirleme kullanım
data = response.xpath('//div').css('span.bio').extract()
# link takip
yield response.follow(url=next_url, callback=self.parse_item)
```

---

## İlk Örümceğini Yaz: `CrawlerProcess` ile Çalıştırma
Aşağıdaki örümcek, bir sayfayı isteyip HTML’ini dosyaya yazar. Tek dosyalık çalıştırmalar için idealdir.

```python
# file: spider_dc.py
import scrapy
from scrapy.crawler import CrawlerProcess

class DCSpider(scrapy.Spider):
    name = "dc_spider"

    def start_requests(self):
        url = "https://www.datacamp.com/courses/all"
        yield scrapy.Request(url=url, callback=self.parse)

    def parse(self, response):
        with open("DC_courses.html", "wb") as fout:
            fout.write(response.body)

if __name__ == "__main__":
    process = CrawlerProcess()
    process.crawl(DCSpider)
    process.start()
```

> `start_requests` en az bir `scrapy.Request` üretmeli; `callback` hangi fonksiyonun `response`’u işleyeceğini belirler.

---

## Crawl: Linkleri Çek, Sayfaları Gez, Veri Topla
Aynı sayfadaki kurs kartlarından linkleri toplayıp CSV’ye yazalım; ardından her linki **follow** edip ikinci parse fonksiyonuna gönderelim.

```python
# file: spider_links.py
import scrapy
from scrapy.crawler import CrawlerProcess

class DCLinksSpider(scrapy.Spider):
    name = "dc_links"

    def start_requests(self):
        yield scrapy.Request(
            url="https://www.datacamp.com/courses/all",
            callback=self.parse
        )

    def parse(self, response):
        # tek adımda
        links = response.css('div.course-block > a::attr(href)').extract()

        # veya adım adım
        # blocks = response.css('div.course-block')
        # hrefs = blocks.xpath('./a/@href')
        # links = hrefs.extract()

        # dosyaya yaz
        with open("DC_links.csv", "w", encoding="utf-8") as f:
            for link in links:
                f.write(link + "\n")

        # her linki takip et
        for link in links:
            yield response.follow(url=link, callback=self.parse_course)

    def parse_course(self, response):
        # burada her kurs sayfasını parse edebilirsin
        self.logger.info("Parsed course page: %s", response.url)

if __name__ == "__main__":
    process = CrawlerProcess()
    process.crawl(DCLinksSpider)
    process.start()
```

> **Not:** Sunumda `"/n"` kaçış dizisi yer alıyordu; Python’da satır sonu `"\n"` olmalıdır.

---

## Mini Capstone: Kurs → Bölüm Başlıkları Sözlüğü
Bu örnekte, ön sayfadan kurs linklerini toplayıp her kurs sayfasında **başlık** ve **bölüm (chapter) başlıklarını** alıp bir sözlüğe kaydediyoruz.

```python
# file: spider_capstone.py
import json
import scrapy
from scrapy.crawler import CrawlerProcess

dc_dict = {}

class DCChapterSpider(scrapy.Spider):
    name = "dc_chapter_spider"

    def start_requests(self):
        url = "https://www.datacamp.com/courses/all"
        yield scrapy.Request(url=url, callback=self.parse_front)

    def parse_front(self, response):
        course_blocks = response.css('div.course-block')
        course_links = course_blocks.xpath('./a/@href')
        for url in course_links.extract():
            yield response.follow(url=url, callback=self.parse_pages)

    def parse_pages(self, response):
        # Kurs başlığı
        title_sel = response.xpath('//h1[contains(@class, "title")]/text()')
        course_title = (title_sel.extract_first() or '').strip()

        # Chapter başlıkları
        ch_titles = response.css('h4.chapter__title::text')
        chapters = [t.strip() for t in ch_titles.extract()]

        if course_title:
            dc_dict[course_title] = chapters

    def closed(self, reason):
        with open('dc_chapters.json', 'w', encoding='utf-8') as f:
            json.dump(dc_dict, f, ensure_ascii=False, indent=2)

if __name__ == "__main__":
    process = CrawlerProcess()
    process.crawl(DCChapterSpider)
    process.start()
```

> `closed(self, reason)` spider bittiğinde çağrılır; toplanan sözlüğü JSON’a yazdırmak için ideal.

---

## Kontrol Listesi (Debug / İpucu)
- **Elementleri incele:** Tarayıcıda “Inspect Element” ile gerçek HTML’ye bak.
- **Selector test et:** `scrapy shell URL` ile interaktif denemeler yap (`response.css(...)`, `response.xpath(...)`).
- **Kademeli geliştir:** Önce doğru elementi seç; sonra attribute/metin çıkar; sonra listeyi döngüye sok; en son dosyaya yaz.
- **Bağıl (relative) vs mutlak XPath:** `./a/@href` mevcut node’un çocuğunu hedefler; en başa `//` koyarsan belki istenmeyen çok sonuç gelir.
- **Index’ler 1’den başlar:** XPath `p[1]`, `p[2]` …
- **`::text` vs `//text()` farkı:** Doğrudan metin mi tüm alt metinler mi istediğine göre seç.

---

## Sık Karşılaşılan Hatalar ve Çözümler
- **Boş liste/None gelmesi:** Seçici yanlış; sayfa dinamik olabilir. Önce HTML’nin geldiğini `response.text[:500]` ile teyit et; gerekirse `selenium` ya da API yoluna bak.
- **Göreli linkler:** `response.follow` çoğu zaman göreli linkleri otomatik çözer. Kendi başına işliyorsan `urljoin` kullan.
- **Encode/escape sorunları:** Dosyaya yazarken `encoding='utf-8'` kullan. Satır sonu için `"\n"`.

---

## Ek Ödevler ve Geliştirmeler
- Çektiğin linklerden her kurs sayfasında eğitmen adlarını/etiketlerini de topla.
- Sitede sayfalama varsa `next` linkini `response.follow` ile takip ederek çok sayfalı crawl yap.
- Çıkardığın veriyi **Pipelines** üzerinden CSV/JSON/SQLite’e aktaracak mini bir Scrapy projesi kur.

---

### Proje Yapı Önerisi (tek dosyalık örümceklerle)
```
repo/
├─ WEB_SCRAPING_SPIDER_GUIDE.md  # bu rehber
├─ spider_dc.py                   # ilk HTML yazdırma örneği
├─ spider_links.py                # link çek + follow + CSV
└─ spider_capstone.py             # kurs -> bölümler sözlüğü (JSON)
```

> Daha büyük projelerde `scrapy startproject projename` komutu ile klasörlü yapı (spiders, items, pipelines, settings) tercih edilir.

---

**Mutlu kazımalar!** 👋
