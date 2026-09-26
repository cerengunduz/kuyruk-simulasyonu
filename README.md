# Hava Savunma Tehdit Saptama Kuyruk Simülasyonu (SimPy)

Bu proje, bir hava savunma sisteminde tehditlerin (uçak, füze, İHA vb.) algılanıp
sınıflandırılması ve imha edilmesi sürecini **çok aşamalı bir kuyruk ağı (tandem
queueing network)** olarak modelleyen, Python + [SimPy](https://simpy.readthedocs.io/)
tabanlı bir kesikli olay simülasyonudur.

Proje, klasik "Arena" tarzı bir simülasyon çalışmasının (varış süreci, kaynaklar,
kuyruklar, öncelikler, servis süreleri, istatistik toplama) Python ile nasıl
kurulacağını gösterir ve akademik/ders projesi olarak doğrudan kullanılabilir.

> ⚠️ Bu proje yalnızca **kuyruk teorisi ve olay-tabanlı simülasyon** amaçlıdır.
> Gerçek bir silah sistemi, sensör donanımı veya operasyonel karar destek aracı
> değildir; sayılar ve parametreler tamamen varsayımsal örnek değerlerdir.

## Sistem Modeli

Bir tehdit (threat) sisteme girdiğinde üç aşamadan geçer:

```
        λ (varış oranı)
           │
           ▼
   ┌───────────────┐     ┌──────────────────┐     ┌────────────────┐
   │  1) RADAR /    │ --> │ 2) SINIFLANDIRMA │ --> │ 3) ANGAJMAN /   │ --> Sistemden çıkış
   │  TESPİT KUYRUĞU│     │   (Tehdit Analizi)│     │  İMHA KUYRUĞU  │
   └───────────────┘     └──────────────────┘     └────────────────┘
   N_radar sunucu          N_classifier sunucu       N_battery sunucu
```

- **Varışlar**: Poisson süreci (üstel geliş aralıkları), her tehdide rastgele bir
  **öncelik seviyesi** atanır (Yüksek / Orta / Düşük). Yüksek öncelikli tehditler
  kuyrukta öne alınır (`simpy.PriorityResource`).
- **Aşama 1 – Radar/Tespit**: Sınırlı sayıda radar kaynağı tehdidi tespit eder.
- **Aşama 2 – Sınıflandırma**: Sınırlı sayıda operatör/analiz istasyonu tehdidi
  değerlendirir (dost/düşman, tehlike derecesi).
- **Aşama 3 – Angajman/İmha**: Sınırlı sayıda "batarya" kaynağı tehdidi etkisiz
  hale getirir.
- **Sabırsızlık (reneging)**: Bir tehdit, tespit kuyruğunda belirli bir "reaksiyon
  penceresi" süresinden fazla beklerse sistemden **kayıp (undetected)** olarak
  çıkar — gerçek sistemlerde tepki süresi sınırlı olduğu için bu önemli bir
  performans göstergesidir.

### Toplanan Metrikler

- Her aşamada ortalama/95. yüzdelik bekleme süresi
- Sistemde toplam kalış süresi (sojourn time)
- Kaynak kullanım oranları (utilization) — radar, sınıflandırma, batarya
- Kuyruk uzunluğu zaman serisi
- Kayıp (reneged / undetected) tehdit oranı
- Önceliğe göre kırılım (Yüksek/Orta/Düşük tehditler için ayrı istatistikler)

## Klasör Yapısı

```
threat-detection-sim/
├── README.md
├── requirements.txt
├── LICENSE
├── .gitignore
├── config.yaml                 # Varsayılan simülasyon parametreleri
├── src/
│   ├── __init__.py
│   ├── config.py                # Config yükleme / dataclass
│   ├── entities.py              # Threat veri sınıfı + istatistik toplayıcı
│   ├── simulation.py             # SimPy simülasyon modeli (çekirdek)
│   ├── run_experiment.py         # CLI: simülasyonu çalıştır, CSV/özet üret
│   └── visualize.py              # matplotlib grafikleri
├── tests/
│   └── test_simulation.py       # pytest ile temel doğrulama testleri
├── docs/
│   └── model.md                 # Matematiksel model ve varsayımlar
└── results/
    └── .gitkeep
```

## Kurulum

```bash
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

## Çalıştırma

Varsayılan parametrelerle tek bir simülasyon koşusu:

```bash
python -m src.run_experiment
```

Parametreleri komut satırından değiştirme:

```bash
python -m src.run_experiment \
    --sim-time 480 \
    --arrival-rate 0.5 \
    --num-radars 2 \
    --num-classifiers 2 \
    --num-batteries 1 \
    --seed 42 \
    --output results/run1.csv
```

Grafikleri üretmek için:

```bash
python -m src.visualize results/run1.csv
```

Çoklu senaryo karşılaştırması (ör. farklı radar sayıları için kayıp oranı):

```bash
python -m src.run_experiment --sweep num_radars=1,2,3,4 --replications 10
```

Testleri çalıştırma:

```bash
pytest tests/ -v
```

## Parametreler (`config.yaml`)

| Parametre           | Açıklama                                          | Varsayılan |
|---------------------|----------------------------------------------------|-----------|
| `arrival_rate`       | Tehdit varış oranı (adet/dakika, λ)                | 0.4       |
| `num_radars`         | Radar/tespit kaynağı sayısı                        | 2         |
| `detect_mean_time`   | Ortalama tespit süresi (dk)                        | 2.0       |
| `num_classifiers`    | Sınıflandırma istasyonu sayısı                      | 2         |
| `classify_mean_time` | Ortalama sınıflandırma süresi (dk)                 | 1.5       |
| `num_batteries`      | Angajman/imha bataryası sayısı                     | 1         |
| `engage_mean_time`   | Ortalama angajman süresi (dk)                      | 3.0       |
| `reaction_window`    | Tespit kuyruğunda azami bekleme (dk) — aşılırsa kayıp | 5.0    |
| `sim_time`           | Toplam simülasyon süresi (dk)                      | 480       |
| `seed`               | Rastgelelik tohumu (tekrarlanabilirlik için)        | 42        |

## Neden SimPy? (Arena ile ilişkisi)

Arena'daki *Create, Process, Seize/Delay/Release, Decide* blokları burada doğrudan
karşılık bulur:

| Arena bloğu     | SimPy karşılığı                                   |
|-----------------|----------------------------------------------------|
| Create          | `env.process(threat_generator(env, ...))`          |
| Seize/Delay/Release | `with resource.request() as req: yield req; yield env.timeout(...)` |
| Decide          | `if/else` + `random.random()`                       |
| Queue istatistiği | `simpy.Resource` içindeki `.queue` uzunluğu + özel `StatsCollector` |
| Replication (10 tekrar) | `for rep in range(replications): ...` döngüsü + farklı `seed` |

Bu sayede Arena dersinde/ödevinde kurulan modelin birebir açık kaynak, kod-tabanlı,
tekrarlanabilir (reproducible) bir versiyonu elde edilir.

## Lisans

MIT — bkz. `LICENSE`.
