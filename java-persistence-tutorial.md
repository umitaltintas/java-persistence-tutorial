s# Java Persistence Ekosistemi: Kapsamlı Tutorial

> Bu tutorial, Java'da veri tabanı erişiminin temellerinden ileri düzey pratiklere kadar her şeyi,
> çalışan kod örnekleriyle adım adım öğretir. Thorben Janssen'in anlatımındaki tüm konuları kapsar.

---

## İçindekiler

1. [Ekosisteme Genel Bakış](#1-ekosisteme-genel-bakış)
2. [JDBC — Her Şeyin Temeli](#2-jdbc--her-şeyin-temeli)
3. [JPA (Jakarta Persistence) — ORM Standardı](#3-jpa-jakarta-persistence--orm-standardı)
4. [Hibernate — JPA'nın Motoru](#4-hibernate--jpanın-motoru)
5. [Persistence Context ve Entity Yaşam Döngüsü](#5-persistence-context-ve-entity-yaşam-döngüsü)
6. [Dirty Checking Mekanizması](#6-dirty-checking-mekanizması)
7. [Spring Data JPA — Üst Düzey Soyutlama](#7-spring-data-jpa--üst-düzey-soyutlama)
8. [Spring Data JDBC — Sihirsiz Yaklaşım](#8-spring-data-jdbc--sihirsiz-yaklaşım)
9. [jOOQ — SQL Odaklı Tip Güvenli Sorgular](#9-jooq--sql-odaklı-tip-güvenli-sorgular)
10. [Hibrit Yaklaşım: Hibernate + jOOQ](#10-hibrit-yaklaşım-hibernate--jooq)
11. [R2DBC ve Reaktif Veri Erişimi](#11-r2dbc-ve-reaktif-veri-erişimi)
12. [Jakarta Data — Yeni Stateless Standart](#12-jakarta-data--yeni-stateless-standart)
13. [Hibernate Search ve Tam Metin Arama](#13-hibernate-search-ve-tam-metin-arama)
14. [REST API'lerde PUT ve PATCH Best Practices](#14-rest-apilerde-put-ve-patch-best-practices)
15. [DTO Mapping Araçları: MapStruct, Jackson, JsonNullable](#15-dto-mapping-araçları-mapstruct-jackson-jsonnullable)
16. [Sık Yapılan Hatalar ve Çözümleri](#16-sık-yapılan-hatalar-ve-çözümleri)
17. [Proje Kurulumu (Spring Boot)](#17-proje-kurulumu-spring-boot)

---

## 1. Ekosisteme Genel Bakış

Java'da veri tabanı erişimi tek bir araçtan ibaret değildir. Katmanlı bir yapı söz konusudur:

```
┌─────────────────────────────────────────────────────────┐
│  GELİŞTİRİCİ KODU (Sen burada yazıyorsun)              │
├─────────────────────────────────────────────────────────┤
│  Spring Data JPA / Spring Data JDBC / Spring Data R2DBC │  ← Üst düzey soyutlama
├─────────────────────────────────────────────────────────┤
│  Jakarta Persistence (JPA) / Jakarta Data               │  ← Standartlar (Kurallar)
├─────────────────────────────────────────────────────────┤
│  Hibernate / EclipseLink                                │  ← ORM Motorları (Uygulayıcılar)
├─────────────────────────────────────────────────────────┤
│  JDBC / R2DBC                                           │  ← Düşük seviye DB bağlantısı
├─────────────────────────────────────────────────────────┤
│  PostgreSQL / MySQL / Oracle                            │  ← Veri Tabanı
└─────────────────────────────────────────────────────────┘
```

### Katmanlar Arası İlişki

- **JDBC**: Java'nın veri tabanlarıyla konuşmak için kullandığı en temel API. Tüm ORM araçları
  sonuçta JDBC üzerinden SQL çalıştırır.
- **JPA (Jakarta Persistence)**: Bir **spesifikasyon** (kural seti). Tek başına iş yapmaz.
  "Entity nasıl tanımlanır, sorgu nasıl yazılır" gibi kuralları belirler.
- **Hibernate**: JPA spesifikasyonunu **uygulayan** (implement eden) bir kütüphane. Gerçek işi
  yapan motordur. EclipseLink ise alternatifidir.
- **Spring Data JPA**: Hibernate ve JPA üzerine bir soyutlama katmanıdır. Geliştiriciyi
  tekrarlayan koddan kurtarır.
- **jOOQ**: ORM değildir. SQL'i Java kodu içinde tip güvenli (type-safe) yazmayı sağlar.

---

## 2. JDBC — Her Şeyin Temeli

JDBC (Java Database Connectivity), Java'nın veri tabanına bağlanmak için sunduğu en düşük
seviye API'dir. Her araç sonuçta JDBC kullanır.

### 2.1 JDBC ile Basit Sorgu

```java
import java.sql.*;

public class JdbcExample {

    public static void main(String[] args) throws Exception {
        // 1. Bağlantı kur
        String url = "jdbc:postgresql://localhost:5432/mydb";
        Connection conn = DriverManager.getConnection(url, "user", "password");

        // 2. SQL hazırla (PreparedStatement ile SQL Injection koruması)
        String sql = "SELECT id, name, price FROM products WHERE category = ?";
        PreparedStatement stmt = conn.prepareStatement(sql);
        stmt.setString(1, "ELECTRONICS");

        // 3. Çalıştır ve sonuçları oku
        ResultSet rs = stmt.executeQuery();
        while (rs.next()) {
            long id = rs.getLong("id");
            String name = rs.getString("name");
            BigDecimal price = rs.getBigDecimal("price");
            System.out.println(id + " - " + name + " - " + price);
        }

        // 4. Kaynakları kapat
        rs.close();
        stmt.close();
        conn.close();
    }
}
```

### 2.2 JDBC'nin Sorunları (Neden ORM'e İhtiyaç Duyuldu?)

| Sorun | Açıklama |
|---|---|
| **Boilerplate Kod** | Her sorgu için bağlantı aç, statement hazırla, sonuç oku, kapat — tekrar tekrar |
| **Tip Dönüşümü** | `rs.getString()`, `rs.getLong()` — her sütunu elle dönüştürmen gerekir |
| **Nesne-Tablo Uyumsuzluğu** | SQL tabloları düz (flat), Java nesneleri hiyerarşik — dönüşüm zor |
| **Veritabanı Bağımlılığı** | Farklı DB'lerde SQL sözdizimi farklı olabilir |

Bu sorunlar ORM (Object-Relational Mapping) kavramını doğurdu.

---

## 3. JPA (Jakarta Persistence) — ORM Standardı

JPA bir **spesifikasyondur**, kütüphane değildir. "Java'da ORM nasıl yapılır" sorusunun
kurallarını belirler. Eski adı Java Persistence API, yenisi Jakarta Persistence'dır.

### 3.1 Entity Tanımlama

JPA'da bir Java sınıfını veri tabanı tablosuna eşlemek için anotasyonlar kullanılır:

```java
import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity                          // Bu sınıf bir veri tabanı tablosuna karşılık gelir
@Table(name = "products")       // Tablo adı (yazılmazsa sınıf adı kullanılır)
public class Product {

    @Id                          // Birincil anahtar (Primary Key)
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // DB otomatik artan ID üretir
    private Long id;

    @Column(nullable = false, length = 200)  // NOT NULL, max 200 karakter
    private String name;

    @Column(precision = 10, scale = 2)  // Ondalıklı sayı: 10 basamak, 2'si ondalık
    private BigDecimal price;

    @Column(columnDefinition = "TEXT")   // Uzun metin
    private String description;

    @Column(name = "created_at")         // Sütun adı Java'daki field adından farklıysa
    private LocalDateTime createdAt;

    @Enumerated(EnumType.STRING)         // Enum'u string olarak DB'ye yaz
    private ProductStatus status;

    // Getter ve Setter'lar
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public BigDecimal getPrice() { return price; }
    public void setPrice(BigDecimal price) { this.price = price; }

    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }

    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }

    public ProductStatus getStatus() { return status; }
    public void setStatus(ProductStatus status) { this.status = status; }
}
```

### 3.2 İlişki (Relationship) Tanımlama

```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String status;

    // Bir siparişin birden fazla kalemi olabilir (One-to-Many)
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    // Bir siparişin bir kullanıcısı vardır (Many-to-One)
    @ManyToOne(fetch = FetchType.LAZY)   // LAZY: Kullanıcı bilgisi istenene kadar yükleme
    @JoinColumn(name = "user_id")
    private User user;
}

@Entity
@Table(name = "order_items")
public class OrderItem {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private int quantity;
    private BigDecimal unitPrice;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    private Product product;
}
```

### 3.3 JPQL (Java Persistence Query Language)

JPA kendi sorgu dilini sunar. SQL'e benzer ama **tablo** yerine **Java sınıfı** adlarını kullanır:

```java
// SQL:  SELECT * FROM products WHERE price > 100
// JPQL: SELECT p FROM Product p WHERE p.price > 100

// SQL:  SELECT p.name, COUNT(oi.id)
//       FROM products p JOIN order_items oi ON p.id = oi.product_id
//       GROUP BY p.name
// JPQL:
@Query("SELECT p.name, COUNT(oi) FROM Product p JOIN p.orderItems oi GROUP BY p.name")
List<Object[]> getProductOrderCounts();
```

---

## 4. Hibernate — JPA'nın Motoru

Hibernate, JPA spesifikasyonunu uygulayan en yaygın ORM çerçevesidir. Spring Boot kullandığında
arka planda Hibernate çalışır.

### 4.1 Hibernate'in Yaptığı İş

1. Entity sınıflarını okur → SQL `CREATE TABLE` ifadeleri üretir (geliştirme ortamında)
2. `save()` çağrısını → SQL `INSERT INTO` ifadesine çevirir
3. JPQL sorgularını → Veri tabanına özel SQL'e çevirir
4. Entity değişikliklerini takip eder → Otomatik `UPDATE` üretir (Dirty Checking)
5. İlişkileri (Relationship) yönetir → Gerektiğinde `JOIN` sorguları üretir

### 4.2 EntityManager — JPA'nın Temel Arayüzü

```java
@Service
public class ProductServiceWithEM {

    @PersistenceContext
    private EntityManager entityManager;

    @Transactional
    public void entityManagerExamples() {

        // --- persist(): Yeni kayıt ekler ---
        Product newProduct = new Product();
        newProduct.setName("Mouse");
        newProduct.setPrice(new BigDecimal("29.99"));
        entityManager.persist(newProduct);
        // Bu noktada newProduct MANAGED durumuna geçti
        // Henüz DB'ye INSERT atılmamış olabilir (flush bekliyor)

        // --- find(): ID ile bulur ---
        Product found = entityManager.find(Product.class, 1L);
        // found artık MANAGED. Değişiklikler otomatik kaydedilecek.

        // --- JPQL Query ---
        List<Product> expensive = entityManager
            .createQuery("SELECT p FROM Product p WHERE p.price > :min", Product.class)
            .setParameter("min", new BigDecimal("100"))
            .getResultList();

        // --- remove(): Siler ---
        Product toDelete = entityManager.find(Product.class, 99L);
        if (toDelete != null) {
            entityManager.remove(toDelete);
            // Transaction bittiğinde DELETE sorgusu çalışır
        }

        // --- detach(): Persistence Context'ten çıkarır ---
        entityManager.detach(found);
        // Artık found üzerindeki değişiklikler DB'ye YANSIMAZ

        // --- merge(): Detached nesneyi tekrar Managed yapar ---
        found.setName("Güncellenmiş İsim");
        Product mergedProduct = entityManager.merge(found);
        // mergedProduct MANAGED, found hala DETACHED!
        // Dönen nesneyi kullanmak önemli.

        // --- flush(): Bekleyen SQL'leri hemen DB'ye gönderir ---
        entityManager.flush();
        // Normalde transaction sonunda otomatik yapılır.
        // flush() sadece sıralamanın önemli olduğu durumlarda kullanılır.
    }
}
```

---

## 5. Persistence Context ve Entity Yaşam Döngüsü

Bu konu, JPA (Hibernate) kullanan bir geliştiricinin "Junior" dan "Senior" a geçişindeki
en kritik bilgidir.

### 5.1 Persistence Context Nedir?

Persistence Context, Java kodunuz ile veri tabanı arasında duran bir **tampon bellek** (buffer)
ve **değişiklik takipçisi**dir.

```
┌──────────────┐     ┌─────────────────────────┐     ┌──────────────┐
│  Java Kodu   │ ←→  │  Persistence Context    │ ←→  │  Veri Tabanı │
│  (Uygulama)  │     │  (1. Seviye Ön Bellek)  │     │  (SQL DB)    │
└──────────────┘     └─────────────────────────┘     └──────────────┘
```

İki temel görevi:
1. **Dirty Checking**: Nesnelerin değişip değişmediğini takip eder
2. **Kimlik Garantisi (Identity Map)**: Aynı ID'ye sahip veriyi aynı transaction içinde
   iki kez çekersen, DB'ye gitmez; hafızadaki aynı nesneyi döndürür

### 5.2 Entity'nin 4 Yaşam Evresi

```
    new Product()          persist() / save()
  ┌─────────────┐       ┌─────────────────┐
  │  TRANSIENT  │ ────→ │    MANAGED      │ ←──── find() / query
  │  (Yeni)     │       │  (Yönetilen)    │
  └─────────────┘       └────────┬────────┘
                                 │
                    ┌────────────┼────────────┐
                    │ detach()   │ remove()   │
                    ↓            ↓            │
              ┌───────────┐  ┌──────────┐    │
              │ DETACHED  │  │ REMOVED  │    │
              │ (Kopmuş)  │  │ (Silinm.)│    │
              └─────┬─────┘  └──────────┘    │
                    │                         │
                    └── merge() ──────────────┘
```

#### A. TRANSIENT (Yeni / Geçici)

```java
Product product = new Product();
product.setName("Laptop");
// Şu an TRANSIENT. DB bilmiyor, Hibernate bilmiyor.
// Garbage Collector bu nesneyi istediği zaman silebilir.
```

#### B. MANAGED (Yönetilen) — İşte Tüm Sihir Burada

```java
@Transactional
public void managedExample() {
    // Yol 1: persist ile Managed yapma
    Product newProduct = new Product();
    newProduct.setName("Tablet");
    entityManager.persist(newProduct);  // Artık MANAGED

    // Yol 2: find ile Managed yapma (en yaygın yol)
    Product existing = entityManager.find(Product.class, 1L);  // MANAGED

    // MANAGED nesne üzerindeki değişiklikler OTOMATİK kaydedilir:
    existing.setPrice(new BigDecimal("999.99"));
    // Burada save() veya update() çağrısı YOK!
    // Transaction bittiğinde Hibernate bunu kendi halleder.
}
```

#### C. DETACHED (Kopmuş)

```java
@Transactional
public Product getProduct(Long id) {
    return productRepository.findById(id).orElseThrow();
    // Metot bitip @Transactional kapanınca, dönen nesne DETACHED olur.
}

// Controller'da:
public void updatePrice() {
    Product p = productService.getProduct(1L);
    p.setPrice(new BigDecimal("500"));
    // BU DEĞİŞİKLİK VERİ TABANINA YANSIMAZ!
    // Çünkü product artık DETACHED. Persistence Context kapandı.
}
```

#### D. REMOVED (Silinmiş)

```java
@Transactional
public void deleteProduct(Long id) {
    Product product = entityManager.find(Product.class, id);  // MANAGED
    entityManager.remove(product);  // REMOVED
    // Transaction bittiğinde DELETE sorgusu çalışır.
}
```

### 5.3 Persistence Context'in Kimlik Garantisi

```java
@Transactional
public void identityMapDemo() {
    // İlk çağrı: DB'ye SELECT sorgusu gider
    Product p1 = productRepository.findById(1L).get();

    // İkinci çağrı: DB'ye sorgu GİTMEZ! Persistence Context'ten döner.
    Product p2 = productRepository.findById(1L).get();

    // Aynı nesne mi?
    System.out.println(p1 == p2);  // true — Sadece değer değil, referans da aynı!
}
```

---

## 6. Dirty Checking Mekanizması

### 6.1 Nasıl Çalışır?

1. `findById(1)` çağrılır → Hibernate DB'den veriyi çeker, Java nesnesi oluşturur
2. Bu nesneyi Persistence Context'e koyar
3. **Gizli İşlem**: Nesnenin o anki halinin bir **kopyasını (snapshot)** alır
4. Transaction biterken, mevcut nesneyi snapshot ile **karşılaştırır**
5. Farklılık varsa (nesne "kirli" — dirty), otomatik `UPDATE` SQL'i oluşturur ve çalıştırır

### 6.2 Kod Üzerinde Gösterim

```java
@Service
public class ProductService {

    @Autowired
    private ProductRepository productRepository;

    @Transactional
    public void updateProductPrice(Long id, BigDecimal newPrice) {

        // Adım 1: DB'den oku → Nesne MANAGED oldu
        // Adım 2: Hibernate arka planda snapshot aldı:
        //         snapshot = { name: "Laptop", price: 1000.00, ... }
        Product product = productRepository.findById(id).orElseThrow();

        // Adım 3: Sadece Java nesnesinin değerini değiştir
        product.setPrice(newPrice);

        // Adım 4: Metot biter, @Transactional kapanır
        // Hibernate karşılaştırır:
        //   mevcut  = { name: "Laptop", price: 1500.00, ... }
        //   snapshot = { name: "Laptop", price: 1000.00, ... }
        //   FARK VAR! → UPDATE products SET price = 1500.00 WHERE id = ?

        // DİKKAT: productRepository.save(product) YAZILMAZ!
        // Çünkü nesne zaten MANAGED. save() gereksiz işlem yapar.
    }
}
```

### 6.3 save() Çağrısının Gereksiz Olduğu Durumlar

```java
// ❌ YANLIŞ — Gereksiz save() çağrısı
@Transactional
public void badExample(Long id) {
    Product product = productRepository.findById(id).orElseThrow();
    product.setName("Yeni İsim");
    productRepository.save(product);  // GEREKSİZ! Zaten Managed.
    // Spring Data arka planda:
    // 1. "Bu nesnenin ID'si var mı?" → Evet
    // 2. "O zaman merge() çağırayım" → Ama zaten Managed...
    // 3. Boşuna CPU döngüsü harcandı.
}

// ✅ DOĞRU — Temiz ve verimli
@Transactional
public void goodExample(Long id) {
    Product product = productRepository.findById(id).orElseThrow();
    product.setName("Yeni İsim");
    // Bitti. Hibernate halleder.
}
```

---

## 7. Spring Data JPA — Üst Düzey Soyutlama

Spring Data JPA, Hibernate ve JPA üzerine kurulu bir soyutlama katmanıdır. Repository
arayüzleri tanımlayarak, hiç SQL yazmadan CRUD işlemleri yapmanızı sağlar.

### 7.1 Temel Repository Tanımı

```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.repository.query.Param;

public interface ProductRepository extends JpaRepository<Product, Long> {

    // ─── Türetilmiş Sorgular (Derived Queries) ───
    // Spring Data metot adından SQL üretir

    // SELECT * FROM products WHERE category = ?
    List<Product> findByCategory(String category);

    // SELECT * FROM products WHERE category = ? AND status = ?
    // DİKKAT: En fazla 2 parametre! Daha fazlası okunaksız olur.
    List<Product> findByCategoryAndStatus(String category, ProductStatus status);

    // SELECT * FROM products WHERE price BETWEEN ? AND ?
    List<Product> findByPriceBetween(BigDecimal min, BigDecimal max);

    // SELECT * FROM products WHERE name LIKE '%keyword%'
    List<Product> findByNameContainingIgnoreCase(String keyword);

    // ─── JPQL ile Özel Sorgular ───
    @Query("SELECT p FROM Product p WHERE p.price > :minPrice ORDER BY p.price DESC")
    List<Product> findExpensiveProducts(@Param("minPrice") BigDecimal minPrice);

    // ─── DTO Projection (Sadece gerekli alanları çek) ───
    @Query("SELECT new com.example.dto.ProductSummary(p.name, p.price) " +
           "FROM Product p WHERE p.category = :category")
    List<ProductSummary> findSummaryByCategory(@Param("category") String category);

    // ─── Native SQL (Veri tabanına özel fonksiyonlar gerektiğinde) ───
    @Query(value = "SELECT * FROM products WHERE is_active = true " +
                   "ORDER BY created_at DESC LIMIT 10",
           nativeQuery = true)
    List<Product> findTop10ActiveProducts();

    // ─── Toplu Güncelleme (Bulk Update) ───
    @Modifying
    @Query("UPDATE Product p SET p.price = p.price * :multiplier WHERE p.category = :cat")
    int bulkUpdatePrices(@Param("cat") String category, @Param("multiplier") double multiplier);
}
```

### 7.2 Servis Katmanı Kullanımı

```java
@Service
public class ProductService {

    @Autowired
    private ProductRepository productRepository;

    // Yeni ürün oluştur
    @Transactional
    public Product createProduct(ProductCreateDTO dto) {
        Product product = new Product();
        product.setName(dto.getName());
        product.setPrice(dto.getPrice());
        product.setCategory(dto.getCategory());
        product.setStatus(ProductStatus.ACTIVE);
        product.setCreatedAt(LocalDateTime.now());
        return productRepository.save(product);
        // Burada save() ZORUNLU çünkü nesne yeni (Transient).
        // save() onu Managed yapar ve ID atar.
    }

    // Mevcut ürünü güncelle
    @Transactional
    public void updateProduct(Long id, ProductUpdateDTO dto) {
        Product product = productRepository.findById(id).orElseThrow();
        product.setName(dto.getName());
        product.setPrice(dto.getPrice());
        // save() YOK! Nesne Managed, Dirty Checking yapacak.
    }

    // Silme
    @Transactional
    public void deleteProduct(Long id) {
        Product product = productRepository.findById(id).orElseThrow();
        productRepository.delete(product);
    }

    // Salt okunur sorgu (Read-Only) — Performans optimizasyonu
    @Transactional(readOnly = true)
    public List<Product> getExpensiveProducts(BigDecimal minPrice) {
        return productRepository.findExpensiveProducts(minPrice);
        // readOnly = true: Hibernate snapshot ALMAZ → Bellek tasarrufu
        // Dirty Checking YAPILMAZ → CPU tasarrufu
    }
}
```

---

## 8. Spring Data JDBC — Sihirsiz Yaklaşım

Spring Data JDBC, Hibernate kullanmaz. JDBC'yi doğrudan kullanır. Persistence Context,
Lazy Loading, Dirty Checking gibi "sihirli" mekanizmalar yoktur. Domain Driven Design (DDD)
yaklaşımına daha uygundur.

### 8.1 Temel Farklar

| Özellik | Spring Data JPA | Spring Data JDBC |
|---|---|---|
| ORM Motoru | Hibernate | Yok (Doğrudan JDBC) |
| Persistence Context | Var | Yok |
| Dirty Checking | Var (Otomatik UPDATE) | Yok |
| Lazy Loading | Var | Yok |
| save() Zorunluluğu | Managed ise gereksiz | HER ZAMAN zorunlu |
| Karmaşıklık | Yüksek (sihir çok) | Düşük (açık ve net) |

### 8.2 Aggregate Root Modeli

```java
// Spring Data JDBC — Entity yerine "Aggregate" kullanılır
import org.springframework.data.annotation.Id;
import org.springframework.data.relational.core.mapping.MappedCollection;
import org.springframework.data.relational.core.mapping.Table;

@Table("orders")
public class Order {

    @Id
    private Long id;
    private String status;
    private LocalDateTime createdAt;

    // Aggregate içindeki alt nesneler
    // Order kaydedildiğinde Items da otomatik kaydedilir/silinir
    @MappedCollection(idColumn = "order_id")
    private Set<OrderItem> items = new HashSet<>();

    // İş mantığı metotları doğrudan Entity'de
    public void addItem(Long productId, int quantity, BigDecimal price) {
        items.add(new OrderItem(productId, quantity, price));
    }

    public BigDecimal getTotalAmount() {
        return items.stream()
            .map(item -> item.getUnitPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

### 8.3 Kullanımı

```java
@Service
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;

    @Transactional
    public void updateOrderStatus(Long orderId, String newStatus) {
        Order order = orderRepository.findById(orderId).orElseThrow();

        order.setStatus(newStatus);

        // !! Spring Data JDBC'de save() ZORUNLU !!
        // Dirty Checking yok. Açıkça kaydetmezseniz değişiklik kaybolur.
        orderRepository.save(order);
    }
}
```

---

## 9. jOOQ — SQL Odaklı Tip Güvenli Sorgular

jOOQ (Java Object Oriented Querying) bir ORM değildir. SQL yazmayı sevenler için, SQL'i
Java kodu içinde **tip güvenli (type-safe)** olarak yazmayı sağlar.

### 9.1 Neden jOOQ?

- Hibernate karmaşık sorgularda (çoklu JOIN, alt sorgu, pencere fonksiyonları) zorlanır
- Hibernate'in ürettiği SQL'i kontrol edemezsiniz
- jOOQ'da yazdığınız SQL tam olarak ne üretiyorsa o çalışır
- Tablo yapısı değişirse, kodunuz **derleme aşamasında** hata verir (compile-time safety)

### 9.2 jOOQ ile Karmaşık Sorgu

```java
@Service
public class ReportService {

    @Autowired
    private DSLContext dsl;  // jOOQ'nun ana nesnesi

    public List<SalesSummary> getSalesSummary(int year) {
        return dsl
            .select(
                USERS.FIRST_NAME,
                USERS.LAST_NAME,
                sum(ORDERS.TOTAL_AMOUNT).as("total_sales"),
                count(ORDERS.ID).as("order_count")
            )
            .from(USERS)
            .join(ORDERS).on(USERS.ID.eq(ORDERS.USER_ID))
            .where(
                extract(ORDERS.ORDER_DATE, DatePart.YEAR).eq(year)
                .and(ORDERS.STATUS.eq("COMPLETED"))
            )
            .groupBy(USERS.ID, USERS.FIRST_NAME, USERS.LAST_NAME)
            .having(sum(ORDERS.TOTAL_AMOUNT).gt(BigDecimal.valueOf(1000)))
            .orderBy(sum(ORDERS.TOTAL_AMOUNT).desc())
            .fetchInto(SalesSummary.class);
    }

    // Pencere fonksiyonu (Window Function) — Hibernate ile yapmak çok zor
    public List<RankedProduct> getTopProductsByCategory() {
        return dsl
            .select(
                PRODUCTS.NAME,
                PRODUCTS.CATEGORY,
                PRODUCTS.PRICE,
                rowNumber()
                    .over(partitionBy(PRODUCTS.CATEGORY).orderBy(PRODUCTS.PRICE.desc()))
                    .as("rank")
            )
            .from(PRODUCTS)
            .fetchInto(RankedProduct.class);
    }
}
```

### 9.3 Ne Zaman Hibernate, Ne Zaman jOOQ?

| Senaryo | Araç |
|---|---|
| Basit CRUD (Oluştur, Oku, Güncelle, Sil) | Hibernate / Spring Data JPA |
| Tekil kayıt güncelleme | Hibernate (Dirty Checking) |
| Karmaşık raporlar, dashboard verileri | **jOOQ** |
| Çoklu JOIN + alt sorgu + gruplama | **jOOQ** |
| Pencere fonksiyonları (ROW_NUMBER, RANK) | **jOOQ** |
| Toplu veri güncelleme (Bulk Update) | jOOQ veya @Modifying JPQL |
| İlişki yönetimi (Cascade, Lazy Loading) | Hibernate |

---

## 10. Hibrit Yaklaşım: Hibernate + jOOQ

Aynı projede her iki aracı birlikte kullanmak mümkündür ve önerilir:
- **Yazma (Write) işlemleri**: Hibernate ile (Entity yönetimi, ilişkiler, validasyon)
- **Karmaşık okuma (Read) işlemleri**: jOOQ ile (Raporlar, dashboard, analitik)

### 10.1 Aynı Projede Kullanım

```java
@Service
public class ProductDashboardService {

    @Autowired
    private ProductRepository productRepository;  // Spring Data JPA (Hibernate)

    @Autowired
    private DSLContext dsl;  // jOOQ

    // Yazma işlemi: Hibernate
    @Transactional
    public void createProduct(ProductCreateDTO dto) {
        Product product = new Product();
        product.setName(dto.getName());
        product.setPrice(dto.getPrice());
        productRepository.save(product);
    }

    // Karmaşık okuma: jOOQ
    @Transactional(readOnly = true)
    public DashboardData getDashboard() {
        // Bu tür bir sorguyu Hibernate ile yazmak çok zahmetli olurdu
        return dsl
            .select(
                count().as("total_products"),
                sum(PRODUCTS.PRICE).as("total_value"),
                avg(PRODUCTS.PRICE).as("avg_price"),
                countDistinct(PRODUCTS.CATEGORY).as("category_count")
            )
            .from(PRODUCTS)
            .where(PRODUCTS.STATUS.eq("ACTIVE"))
            .fetchOneInto(DashboardData.class);
    }
}
```

### 10.2 jOOQ ile SQL Üretip Hibernate ile Çalıştırmak

```java
@Service
public class HybridQueryService {

    @PersistenceContext
    private EntityManager entityManager;

    @Autowired
    private DSLContext dsl;

    public List<Product> findComplexProducts() {
        // 1. jOOQ ile tip güvenli SQL stringi ÜRET (çalıştırma)
        SelectConditionStep<?> jooqQuery = dsl
            .selectFrom(PRODUCTS)
            .where(PRODUCTS.STOCK.lt(10))
            .and(PRODUCTS.CATEGORY.eq("ELECTRONICS"));

        String sql = jooqQuery.getSQL();
        // Çıktı: "SELECT * FROM products WHERE stock < ? AND category = ?"

        List<Object> bindValues = jooqQuery.getBindValues();
        // Çıktı: [10, "ELECTRONICS"]

        // 2. Hibernate'in Native Query'sine ver ve çalıştır
        Query query = entityManager.createNativeQuery(sql, Product.class);
        for (int i = 0; i < bindValues.size(); i++) {
            query.setParameter(i + 1, bindValues.get(i));
        }

        return query.getResultList();
        // Sonuç: Hibernate tarafından yönetilen MANAGED Entity'ler döner
    }
}
```

---

## 11. R2DBC ve Reaktif Veri Erişimi

R2DBC (Reactive Relational Database Connectivity), JDBC'nin reaktif (non-blocking, asenkron)
karşılığıdır. Sanal iş parçacıkları (Virtual Threads) öncesinde çok sayıda eşzamanlı
bağlantıyı yönetmek için tasarlanmıştır.

### 11.1 Spring Data R2DBC Kullanımı

```java
// Entity tanımı (JPA anotasyonları KULLANILMAZ)
import org.springframework.data.annotation.Id;
import org.springframework.data.relational.core.mapping.Table;

@Table("products")
public class Product {
    @Id
    private Long id;
    private String name;
    private BigDecimal price;
    // Getter/Setter...
}

// Repository
public interface ProductRepository extends ReactiveCrudRepository<Product, Long> {
    Flux<Product> findByCategory(String category);
    Mono<Product> findByName(String name);
}

// Servis — Reaktif (Non-Blocking)
@Service
public class ProductService {

    @Autowired
    private ProductRepository repository;

    public Flux<Product> getExpensiveProducts(BigDecimal minPrice) {
        return repository.findAll()
            .filter(p -> p.getPrice().compareTo(minPrice) > 0)
            .sort(Comparator.comparing(Product::getPrice).reversed());
    }

    public Mono<Product> createProduct(Product product) {
        return repository.save(product);
    }
}
```

### 11.2 R2DBC Ne Zaman Kullanılır?

- Çok yüksek eşzamanlılık gerektiren uygulamalar (chat, gerçek zamanlı bildirim)
- WebFlux (reaktif web çerçevesi) kullanıyorsanız
- **Dikkat**: Java 21+ ile gelen Virtual Threads, R2DBC'nin çözdüğü problemi büyük ölçüde
  ortadan kaldırdı. Yeni projelerde Virtual Threads + JDBC tercih edilebilir.

---

## 12. Jakarta Data — Yeni Stateless Standart

Jakarta Data, Spring Data'dan ilham alan ama **standart** (vendor-bağımsız) bir
Repository API'sidir. JPA'nın aksine **Stateless** (durumsuz) çalışır — Persistence Context
yoktur. Hibernate 6.6+ tarafından desteklenir.

### 12.1 Temel Kullanım

```java
import jakarta.data.repository.CrudRepository;
import jakarta.data.repository.Find;
import jakarta.data.repository.Query;
import jakarta.data.repository.Repository;

@Repository
public interface ProductRepository extends CrudRepository<Product, Long> {

    @Find
    List<Product> findByCategory(String category);

    @Query("SELECT p FROM Product p WHERE p.price > :minPrice")
    List<Product> findExpensive(BigDecimal minPrice);
}
```

### 12.2 Stateless vs Stateful (JPA ile Fark)

| Özellik | JPA (Stateful) | Jakarta Data (Stateless) |
|---|---|---|
| Persistence Context | Var | Yok |
| Dirty Checking | Var (Otomatik UPDATE) | Yok |
| save() Zorunluluğu | Managed ise gereksiz | HER ZAMAN zorunlu |
| Öğrenme Eğrisi | Daha dik (sihir) | Daha kolay (açık) |

---

## 13. Hibernate Search ve Tam Metin Arama

SQL'deki `LIKE '%kelime%'` sorgusu büyük verilerde yavaştır ve akıllı arama (typo toleransı,
eşanlamlılar) yapamaz. Hibernate Search, Apache Lucene motorunu kullanarak Google benzeri
tam metin arama özelliği sunar.

### 13.1 Entity'ye Arama Desteği Eklemek

```java
import org.hibernate.search.mapper.pojo.mapping.definition.annotation.*;

@Entity
@Indexed  // Bu entity aranabilir!
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @FullTextField(analyzer = "turkish")  // Tam metin arama alanı
    private String name;

    @FullTextField(analyzer = "turkish")
    private String description;

    @KeywordField  // Tam eşleşme (filtreleme için)
    private String category;

    @GenericField(sortable = Sortable.YES)  // Sıralanabilir
    private BigDecimal price;
}
```

### 13.2 Arama Sorgusu

```java
@Service
public class ProductSearchService {

    @PersistenceContext
    private EntityManager entityManager;

    @Transactional(readOnly = true)
    public List<Product> search(String keyword) {
        SearchSession searchSession = Search.session(entityManager);

        SearchResult<Product> result = searchSession.search(Product.class)
            .where(f -> f.match()
                .fields("name", "description")
                .matching(keyword)
                .fuzzy(1))  // 1 karakter hata toleransı ("laptob" → "laptop")
            .sort(f -> f.score())  // Alaka düzeyine göre sırala
            .fetchAll();

        return result.hits();
    }
}
```

---

## 14. REST API'lerde PUT ve PATCH Best Practices

### 14.1 PUT vs PATCH Farkı

| Özellik | PUT (Tam Değiştirme) | PATCH (Kısmi Güncelleme) |
|---|---|---|
| **Anlam** | "Gönderdiğimle tamamen değiştir" | "Sadece gönderdiğim alanları güncelle" |
| **Gelen JSON** | TÜM alanlar gönderilmeli | Sadece değişecek alanlar |
| **Gönderilmeyen alan** | null / varsayılan olur | **Mevcut değeri korur** |

### 14.2 Altın Kural: Entity'yi Controller'da KULLANMA!

```java
// ❌ YANLIŞ — Entity'yi doğrudan JSON'a bağlamak
@PutMapping("/{id}")
public void update(@PathVariable Long id, @RequestBody Product product) {
    // product DETACHED! Persistence Context'te değil!
    // Güvenlik açığı: Kullanıcı ID, createdAt gibi alanları değiştirebilir!
    productRepository.save(product);  // Tehlikeli!
}

// ✅ DOĞRU — DTO kullan
@PutMapping("/{id}")
public ResponseEntity<Void> update(@PathVariable Long id,
                                   @Valid @RequestBody ProductPutDTO dto) {
    productService.updateProduct(id, dto);
    return ResponseEntity.noContent().build();
}
```

### 14.3 PUT İşlemi (Tam Değiştirme)

```java
// DTO
public class ProductPutDTO {
    @NotBlank
    private String name;
    @NotNull
    private BigDecimal price;
    private String description;  // null gelebilir — PUT'ta sorun değil
    @NotNull
    private String category;
    // Getter/Setter...
}

// Servis
@Service
public class ProductService {

    @Autowired
    private ProductRepository repository;

    @Transactional
    public void updateProductPut(Long id, ProductPutDTO dto) {
        // 1. Fetch — Nesne MANAGED olur
        Product product = repository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Ürün bulunamadı: " + id));

        // 2. Tüm alanları ezerek yaz (overwrite)
        product.setName(dto.getName());
        product.setPrice(dto.getPrice());
        product.setDescription(dto.getDescription());  // null geldiyse DB'de de null olur
        product.setCategory(dto.getCategory());

        // 3. save() YOK — Managed Entity, Dirty Checking yapacak
    }
}
```

### 14.4 PATCH İşlemi — Yöntem 1: Map<String, Object>

"Gönderilmeyen alan" ile "null gönderilen alan" ayrımını yapmanın en basit yolu:

```java
// Controller
@PatchMapping("/{id}")
public ResponseEntity<Void> patch(@PathVariable Long id,
                                  @RequestBody Map<String, Object> updates) {
    productService.patchProduct(id, updates);
    return ResponseEntity.noContent().build();
}

// Servis
@Service
public class ProductService {

    @Transactional
    public void patchProduct(Long id, Map<String, Object> updates) {
        Product product = repository.findById(id).orElseThrow();

        // Sadece JSON'da GÖNDERİLEN anahtarları güncelle
        if (updates.containsKey("name")) {
            product.setName((String) updates.get("name"));
        }
        if (updates.containsKey("price")) {
            product.setPrice(new BigDecimal(updates.get("price").toString()));
        }
        if (updates.containsKey("description")) {
            product.setDescription((String) updates.get("description"));
            // null geldiyse: client bilerek temizlemek istiyor → DB'de null olur
        }
        if (updates.containsKey("category")) {
            product.setCategory((String) updates.get("category"));
        }

        // save() YOK — Managed Entity
    }
}
```

### 14.5 PATCH İşlemi — Yöntem 2: DTO + Tip Güvenliği

```java
// PATCH DTO — Tüm alanlar nullable
public class ProductPatchDTO {
    private String name;         // null = gönderilmedi
    private BigDecimal price;    // null = gönderilmedi
    private String description;
    private String category;
    // Getter/Setter...
}

// Servis — null olan alanlara dokunma
@Transactional
public void patchProductWithDto(Long id, ProductPatchDTO dto) {
    Product product = repository.findById(id).orElseThrow();

    if (dto.getName() != null) {
        product.setName(dto.getName());
    }
    if (dto.getPrice() != null) {
        product.setPrice(dto.getPrice());
    }
    if (dto.getDescription() != null) {
        product.setDescription(dto.getDescription());
    }
    if (dto.getCategory() != null) {
        product.setCategory(dto.getCategory());
    }

    // ⚠️ KISITLAMA: Bu yaklaşımda client bir alanı bilerek null yapamaz!
    // Çünkü null = "gönderilmedi" ile null = "temizle" ayrımı yapılamıyor.
}
```

---

## 15. DTO Mapping Araçları: MapStruct, Jackson, JsonNullable

30-40 alanlı Entity'lerde elle `set/get` yazmak sürdürülebilir değildir. Sektör bu
boilerplate'i şu araçlarla çözer:

### 15.1 MapStruct (Sektör Standardı)

MapStruct derleme zamanında (compile-time) çalışır, reflection kullanmaz, çok hızlıdır.

**Maven Bağımlılığı:**
```xml
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.5.5.Final</version>
</dependency>
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct-processor</artifactId>
    <version>1.5.5.Final</version>
    <scope>provided</scope>
</dependency>
```

**Mapper Arayüzü:**
```java
import org.mapstruct.*;

@Mapper(componentModel = "spring")
public interface ProductMapper {

    // Entity → DTO dönüşümü (Okuma için)
    ProductResponseDTO toResponseDto(Product entity);

    // DTO → Entity (Yeni kayıt oluşturma için)
    Product toEntity(ProductCreateDTO dto);

    // PUT: DTO'daki HER ŞEYİ Entity'nin üzerine yazar (null dahil)
    void updateFromPutDto(ProductPutDTO dto, @MappingTarget Product entity);

    // PATCH: DTO'daki null olan alanları ATLA, sadece dolu olanları güncelle
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    void updateFromPatchDto(ProductPatchDTO dto, @MappingTarget Product entity);
}
```

**Servis Kullanımı (Boilerplate Sıfır!):**
```java
@Service
public class ProductService {

    @Autowired private ProductRepository repository;
    @Autowired private ProductMapper mapper;

    // PUT
    @Transactional
    public void putProduct(Long id, ProductPutDTO dto) {
        Product product = repository.findById(id).orElseThrow();
        mapper.updateFromPutDto(dto, product);
        // Tüm alanlar ezildi. Dirty Checking UPDATE atacak.
    }

    // PATCH
    @Transactional
    public void patchProduct(Long id, ProductPatchDTO dto) {
        Product product = repository.findById(id).orElseThrow();
        mapper.updateFromPatchDto(dto, product);
        // Sadece null OLMAYAN alanlar güncellendi.
    }
}
```

### 15.2 Jackson readerForUpdating (DTO'suz PATCH)

```java
@Service
public class ProductService {

    @Autowired private ProductRepository repository;
    @Autowired private ObjectMapper objectMapper;  // Spring Boot otomatik sağlar

    @Transactional
    public void patchProduct(Long id, JsonNode patchJson) throws IOException {
        Product product = repository.findById(id).orElseThrow();

        // JSON'daki alanları doğrudan Entity'nin üzerine "yamala"
        objectMapper.readerForUpdating(product).readValue(patchJson);

        // Dirty Checking devreye girer, UPDATE atar.
    }
}

// Controller
@PatchMapping("/{id}")
public ResponseEntity<Void> patch(@PathVariable Long id,
                                  @RequestBody JsonNode patchJson) throws IOException {
    productService.patchProduct(id, patchJson);
    return ResponseEntity.noContent().build();
}
```

**⚠️ Güvenlik Uyarısı**: Bu yöntemde Entity doğrudan JSON'a açılır. `@JsonIgnore`
veya `@JsonProperty(access = READ_ONLY)` ile hassas alanları (id, createdAt, role)
koruyun.

### 15.3 JsonNullable (Gerçek RESTful PATCH)

"Gönderilmedi" ile "bilerek null yaptı" ayrımını yapar.

**Maven Bağımlılığı:**
```xml
<dependency>
    <groupId>org.openapitools</groupId>
    <artifactId>jackson-databind-nullable</artifactId>
    <version>0.2.6</version>
</dependency>
```

**DTO Tanımı:**
```java
import org.openapitools.jackson.nullable.JsonNullable;

public class ProductPatchDTO {

    // JsonNullable 3 durumu ifade eder:
    // 1. undefined() → JSON'da bu alan YOK (dokunma)
    // 2. of("değer") → JSON'da değer var (güncelle)
    // 3. of(null)    → JSON'da alan var ama değeri null (null yap)
    private JsonNullable<String> name = JsonNullable.undefined();
    private JsonNullable<BigDecimal> price = JsonNullable.undefined();
    private JsonNullable<String> description = JsonNullable.undefined();

    // Getter/Setter...
}
```

**Servis:**
```java
@Transactional
public void patchProduct(Long id, ProductPatchDTO dto) {
    Product product = repository.findById(id).orElseThrow();

    // isPresent(): JSON'da bu anahtar var mı? (null olsa bile true döner)
    if (dto.getName().isPresent()) {
        product.setName(dto.getName().get());  // null da olabilir
    }
    if (dto.getPrice().isPresent()) {
        product.setPrice(dto.getPrice().get());
    }
    if (dto.getDescription().isPresent()) {
        product.setDescription(dto.getDescription().get());
    }
}
```

**JSON Örnekleri:**
```json
// İsmi güncelle, açıklamayı null yap, fiyata dokunma
{ "name": "Yeni İsim", "description": null }
// → name.isPresent() = true, name.get() = "Yeni İsim"
// → description.isPresent() = true, description.get() = null
// → price.isPresent() = false (dokunulmaz)
```

---

## 16. Sık Yapılan Hatalar ve Çözümleri

### 16.1 N+1 Güncelleme Problemi

```java
// ❌ FELAKET — Her kayıt için ayrı UPDATE sorgusu (1000 kayıt = 1000 sorgu)
@Transactional
public void giveRaise(String department) {
    List<Employee> employees = employeeRepository.findByDepartment(department);
    for (Employee emp : employees) {
        emp.setSalary(emp.getSalary().multiply(new BigDecimal("1.10")));
        // Hibernate her biri için ayrı UPDATE atar!
    }
}

// ✅ DOĞRU — Tek sorgu ile toplu güncelleme
@Modifying
@Query("UPDATE Employee e SET e.salary = e.salary * 1.10 WHERE e.department = :dept")
int bulkRaise(@Param("dept") String department);
// 1000 kayıt olsa bile tek bir UPDATE sorgusu çalışır.
```

### 16.2 N+1 SELECT Problemi (Lazy Loading Tuzağı)

```java
// ❌ Entity'de varsayılan Lazy Loading
@OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
private List<OrderItem> items;

// Kodda:
List<Order> orders = orderRepository.findAll();  // 1 SELECT
for (Order order : orders) {
    order.getItems().size();  // Her sipariş için 1 SELECT daha!
    // 100 sipariş = 1 + 100 = 101 sorgu!
}

// ✅ ÇÖZÜM 1: JOIN FETCH ile tek sorguda getir
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.status = :status")
List<Order> findWithItems(@Param("status") String status);

// ✅ ÇÖZÜM 2: @EntityGraph
@EntityGraph(attributePaths = {"items"})
List<Order> findByStatus(String status);

// ✅ ÇÖZÜM 3: DTO Projection — En iyisi (Entity değil, sadece ihtiyacın olanı çek)
@Query("SELECT new com.example.dto.OrderSummary(o.id, o.status, SIZE(o.items)) " +
       "FROM Order o WHERE o.status = :status")
List<OrderSummary> findSummariesByStatus(@Param("status") String status);
```

### 16.3 Büyük Veri Setlerini Managed Olarak Çekmek (Bellek Tükenmesi)

```java
// ❌ 50.000 kaydı Managed olarak çekmek
@Transactional
public void processAllProducts() {
    List<Product> all = productRepository.findAll();
    // 50.000 Entity + 50.000 Snapshot = 100.000 nesne RAM'de!
    // OutOfMemoryError riski!
}

// ✅ ÇÖZÜM 1: Salt okunur işlemler için readOnly
@Transactional(readOnly = true)
public List<Product> getAllProducts() {
    return productRepository.findAll();
    // readOnly = true: Snapshot alınmaz → Bellek yarıya iner
}

// ✅ ÇÖZÜM 2: DTO Projection kullanmak (En iyi çözüm)
@Query("SELECT new com.example.dto.ProductSummary(p.id, p.name, p.price) FROM Product p")
List<ProductSummary> findAllSummaries();
// DTO'lar MANAGED olmaz. Snapshot alınmaz. Minimum bellek kullanır.

// ✅ ÇÖZÜM 3: Sayfalama (Pagination) kullanmak
Page<Product> products = productRepository.findAll(PageRequest.of(0, 50));
```

### 16.4 Detached Entity'yi Güncellemeye Çalışmak

```java
// ❌ Controller'dan gelen Entity DETACHED
@PostMapping("/update")
public void update(@RequestBody Product product) {
    product.setPrice(new BigDecimal("999"));
    // Bu değişiklik DB'ye YANSIMAZ! product Detached.
    // productRepository.save(product) yazsan bile merge() çalışır ki
    // bu da güvenlik açığı olabilir (client tüm alanları kontrol eder).
}

// ✅ DOĞRU — Fetch-Update deseni
@Transactional
public void updatePrice(Long id, BigDecimal newPrice) {
    Product product = productRepository.findById(id).orElseThrow();  // MANAGED
    product.setPrice(newPrice);
    // Dirty Checking → UPDATE
}
```

---

## 17. Proje Kurulumu (Spring Boot)

### 17.1 Maven Bağımlılıkları (pom.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.0</version>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>persistence-tutorial</artifactId>
    <version>1.0.0</version>

    <properties>
        <java.version>21</java.version>
        <mapstruct.version>1.5.5.Final</mapstruct.version>
    </properties>

    <dependencies>
        <!-- Spring Data JPA (Hibernate dahil gelir) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <!-- Spring Web (REST API) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- PostgreSQL Driver -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- MapStruct (DTO Mapping) -->
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
            <version>${mapstruct.version}</version>
        </dependency>

        <!-- JsonNullable (Gerçek PATCH desteği) -->
        <dependency>
            <groupId>org.openapitools</groupId>
            <artifactId>jackson-databind-nullable</artifactId>
            <version>0.2.6</version>
        </dependency>

        <!-- jOOQ (Opsiyonel - Karmaşık sorgular için) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jooq</artifactId>
        </dependency>

        <!-- Hibernate Search (Opsiyonel - Tam metin arama) -->
        <dependency>
            <groupId>org.hibernate.search</groupId>
            <artifactId>hibernate-search-mapper-orm</artifactId>
            <version>7.1.0.Final</version>
        </dependency>
        <dependency>
            <groupId>org.hibernate.search</groupId>
            <artifactId>hibernate-search-backend-lucene</artifactId>
            <version>7.1.0.Final</version>
        </dependency>

        <!-- Validation -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.mapstruct</groupId>
                            <artifactId>mapstruct-processor</artifactId>
                            <version>${mapstruct.version}</version>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### 17.2 application.yml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/tutorial_db
    username: postgres
    password: postgres
    driver-class-name: org.postgresql.Driver

  jpa:
    hibernate:
      ddl-auto: validate  # Prod'da validate, dev'de update kullanılabilir
    show-sql: true         # Geliştirmede SQL'leri gör
    properties:
      hibernate:
        format_sql: true
        default_batch_fetch_size: 16  # N+1 problemi için batch fetch
        jdbc:
          batch_size: 50    # Toplu INSERT/UPDATE optimizasyonu

  # Hibernate Search (Opsiyonel)
  jpa.properties.hibernate.search:
    backend:
      type: lucene
      directory.root: ./data/search-index
```

### 17.3 Tam Çalışan Örnek: Product CRUD API

```java
// === DTO'lar ===

public record ProductCreateDTO(
    @NotBlank String name,
    @NotNull BigDecimal price,
    String description,
    @NotBlank String category
) {}

public record ProductResponseDTO(
    Long id,
    String name,
    BigDecimal price,
    String description,
    String category,
    LocalDateTime createdAt
) {}

public record ProductPutDTO(
    @NotBlank String name,
    @NotNull BigDecimal price,
    String description,
    @NotBlank String category
) {}

// === Controller ===

@RestController
@RequestMapping("/api/products")
public class ProductController {

    @Autowired private ProductService productService;

    @PostMapping
    public ResponseEntity<ProductResponseDTO> create(@Valid @RequestBody ProductCreateDTO dto) {
        ProductResponseDTO created = productService.create(dto);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }

    @GetMapping("/{id}")
    public ProductResponseDTO getById(@PathVariable Long id) {
        return productService.getById(id);
    }

    @PutMapping("/{id}")
    public ResponseEntity<Void> put(@PathVariable Long id,
                                    @Valid @RequestBody ProductPutDTO dto) {
        productService.putProduct(id, dto);
        return ResponseEntity.noContent().build();
    }

    @PatchMapping("/{id}")
    public ResponseEntity<Void> patch(@PathVariable Long id,
                                      @RequestBody Map<String, Object> updates) {
        productService.patchProduct(id, updates);
        return ResponseEntity.noContent().build();
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        productService.deleteProduct(id);
        return ResponseEntity.noContent().build();
    }
}

// === Servis ===

@Service
public class ProductService {

    @Autowired private ProductRepository repository;
    @Autowired private ProductMapper mapper;

    @Transactional
    public ProductResponseDTO create(ProductCreateDTO dto) {
        Product product = mapper.toEntity(dto);
        product.setCreatedAt(LocalDateTime.now());
        product.setStatus(ProductStatus.ACTIVE);
        Product saved = repository.save(product);  // save() ZORUNLU (yeni kayıt)
        return mapper.toResponseDto(saved);
    }

    @Transactional(readOnly = true)
    public ProductResponseDTO getById(Long id) {
        Product product = repository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Ürün bulunamadı: " + id));
        return mapper.toResponseDto(product);
    }

    @Transactional
    public void putProduct(Long id, ProductPutDTO dto) {
        Product product = repository.findById(id).orElseThrow();
        mapper.updateFromPutDto(dto, product);  // Tüm alanlar ezilir
        // save() YOK — Managed Entity
    }

    @Transactional
    public void patchProduct(Long id, Map<String, Object> updates) {
        Product product = repository.findById(id).orElseThrow();

        if (updates.containsKey("name")) {
            product.setName((String) updates.get("name"));
        }
        if (updates.containsKey("price")) {
            product.setPrice(new BigDecimal(updates.get("price").toString()));
        }
        if (updates.containsKey("description")) {
            product.setDescription((String) updates.get("description"));
        }
        if (updates.containsKey("category")) {
            product.setCategory((String) updates.get("category"));
        }
        // save() YOK — Managed Entity
    }

    @Transactional
    public void deleteProduct(Long id) {
        Product product = repository.findById(id).orElseThrow();
        repository.delete(product);
    }
}
```

---

## Özet: Hangi Aracı Ne Zaman Kullan?

| Senaryo | Araç | Sebep |
|---|---|---|
| Basit CRUD | Spring Data JPA | Minimum kod, otomatik sorgular |
| Karmaşık okuma/rapor | jOOQ | Tip güvenli SQL, tam kontrol |
| Toplu güncelleme | @Modifying JPQL veya jOOQ | N+1 problemini önler |
| DDD / Sihirsiz yaklaşım | Spring Data JDBC | Persistence Context yok, açık ve net |
| Tam metin arama | Hibernate Search | Google benzeri fuzzy arama |
| Reaktif uygulama | Spring Data R2DBC | Non-blocking IO |
| DTO Mapping | MapStruct | Derleme zamanı, hızlı, güvenli |
| Gerçek PATCH desteği | JsonNullable | null vs undefined ayrımı |

**Altın Kural:** Fetch → Update → Let Dirty Checking Handle It.
Yani: Getir → Güncelle → Hibernate'e bırak.
