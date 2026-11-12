# Arhitektura sistema za več salonov

**Faza:** 1 – analiza in arhitektura  
**Stanje:** predlog za pregled; aplikacija in migracije še niso izdelane  
**Časovni pas salonov:** `Europe/Ljubljana`; vse časovne točke se v bazi hranijo kot `timestamptz`.

## 1. Ciljna arhitektura

En Next.js App Router sistem streže vsem salonom. Zahteva se razvrsti v enega od treh kontekstov:

1. **Javna stran salona** – domena iz `Host` glave se normalizira in poišče v `salon_domains`; najdeni aktivni salon poda vsebino, temo in nastavitve. Neznane oziroma neaktivne domene se zavrnejo.
2. **Administracija salona** – skupna prijavna pot (`/admin/login`) in administracija za potrjene uporabnike. Strežnik preveri Supabase sejo, članstvo, `salon_id` in vlogo pri vsaki operaciji. Salon v URL-ju sam po sebi ni dokaz dostopa.
3. **Platformna administracija** – namenski `platform_admin` rolni dostop za upravljanje salonov, domen, predlog in stanja aktivacije. Ločen od članstev v salonu.

Večnajemniška meja je PostgreSQL: vse najemniku pripadajoče vrstice imajo `salon_id`, tuji ključi ga vključujejo, RLS pa preverja članstvo. Strežniška plast ponovi preverjanje pravic; service-role ključ se ne pošlje brskalniku. Javne rezervacije se izvajajo prek omejene strežniške funkcije/RPC, ne prek neposrednega anonimnega vstavljanja v tabele.

### Sestavni deli

- **Next.js + TypeScript + Tailwind CSS**: strežniško upodabljanje javnih strani in administracije, Route Handlers oziroma Server Actions za mutacije.
- **Supabase**: PostgreSQL, Auth in Storage; RLS na vseh tabelah v izpostavljenih shemah. `@supabase/ssr` za piškotne seje na strežniku in v odjemalcu, različice paketov zaklenjene v lockfile.
- **PostgreSQL**: transakcije in omejitve za konsistentnost; območja terminov ter GiST izključitvena omejitev preprečijo prekrivanje aktivnih rezervacij istega zaposlenega.
- **E-pošta**: ponudnik, kot je Resend, za asinhrona obvestila; ključi samo v okolju strežnika. Pošiljanje naj bo prek outbox/čakalne vrste z idempotentnim obdelovalcem.
- **Gostovanje**: Vercel ali Cloudflare, izbran po preverjanju podpore za Next.js runtime, regije, funkcije in domeno po meri. DNS in TLS za vsako domeno; razmestitve prek GitHuba.

Uradna dokumentacija potrjuje App Router kot priporočen Next.js model, Supabase SSR z uporabo piškotkov, potrebo po eksplicitnih dovoljenjih poleg RLS pravil in zmožnost PostgreSQL exclusion/GiST omejitev za preprečevanje prekrivanja. To so vzorci za zasnovo; konkretne združljivosti paketov in gostitelja preverimo ob postavitvi projekta.

## 2. Struktura projekta (ciljna)

```text
src/
  app/
    (public)/                 # javni tenant renderer
      page.tsx
      booking/
    (auth)/admin/login/
    admin/[salonSlug]/        # salon panel
    platform/                 # platformni admin
    api/                      # omejeni webhooki in route handlerji
  components/
    booking/                  # ponovna uporaba korakov rezervacije
    admin/
    themes/                   # ločene kompozicije predlog
  features/
    bookings/                 # domenska pravila, use-cases, sheme
    salons/
    employees/
    services/
    scheduling/
    notifications/
  lib/
    auth/                      # sejni in RBAC pripomočki
    tenants/                   # domena -> salon kontekst
    supabase/                  # browser/server klienta; brez service ključa v browserju
    validation/
  styles/
supabase/
  migrations/
  seed.sql                    # samo razvojni podatki, jasno ločeni od produkcije
  tests/                      # RLS in DB integracijski testi
public/
  fonts/
```

Predloge niso samo barvne teme. Vsaka predloga ima svoj sestav sekcij in kompozicijo (npr. črno-zlata uredniška stran s širokim hero blokom; svetla bež stran s kompaktno navigacijo in asimetrično galerijo; temna sodobna stran z močno tipografijo in poudarjeno kartico za rezervacijo). Podatki o vsebini in oblikovalske nastavitve ostanejo ločeni od komponent. Nastavitve vsebujejo `template_key`, žetone barv/pisav, vrstni red in vidnost sekcij ter različico sheme, da je mogoče konfiguracijo varno nadgrajevati.

## 3. Podatkovni model

Vse primarne ključe predlagam kot UUID. Vse tabele, ki pripadajo salonu, imajo `salon_id NOT NULL`; večina tujih ključev se definira sestavljeno (`(salon_id, referenced_id)`) in referencira enak tenant, da napaka v aplikacijski plasti ne more povezati podatkov različnih salonov.

| Tabela | Namen in glavna polja |
| --- | --- |
| `salons` | `id`, `name`, `status` (`active/suspended`), `timezone` privzeto `Europe/Ljubljana`, `created_at` |
| `salon_domains` | `id`, `salon_id`, normaliziran `hostname` UNIQUE, `is_primary`, `verified_at`; ne hraniti poti ali protokola |
| `user_profiles` | `user_id` (FK `auth.users`), ime; aplikacijski profil ni kopija gesla |
| `salon_memberships` | `salon_id`, `user_id`, `role` (`owner/manager/staff`), `employee_id` opcijsko; UNIQUE `(salon_id,user_id)` |
| `platform_admins` | `user_id` in platformna vloga; dostop ločen od tenant članstva |
| `employees` | `id`, `salon_id`, ime, aktivnost, javni prikaz, opcijski uporabniški profil |
| `services` | `id`, `salon_id`, ime/opis, trajanje, cena v celičnih centih (`price_minor`), valuta, aktivnost |
| `employee_services` | sestavljeni ključi salona, zaposlenega in storitve; določa usposobljenost |
| `working_hours` | salon, opcijski `employee_id` (NULL pomeni osnovni urnik), dan tedna, lokalni začetek/konec, veljavnost; model za ponavljajoč urnik |
| `breaks` | urnik/zaposleni, dan ali ponavljajoči lokalni interval, čas začetka/konca |
| `holidays` | salon in opcijski zaposleni, celodnevni ali časovni razpon, opis; pokrije dopuste, praznike in dela proste dni |
| `customers` | `id`, `salon_id`, ime, telefon, e-pošta, čas ustvarjanja; PII minimizirati in omejiti dostop |
| `bookings` | `id`, `salon_id`, `employee_id`, `service_id`, `customer_id`, `starts_at`, `ends_at`, status, cena/trajanje ob rezervaciji, izvor, opombe |
| `booking_actions` | enkratni zgoščeni tokeni za spremembo/preklic, namen, rok, poraba; surovega tokena ne hraniti |
| `salon_settings` | en zapis na salon: kontakt, naslov, asset reference, booking pravila, lead time, predloga/tema in konfiguracija sekcij |
| `email_outbox` | dogodek, prejemnik, varen payload/reference, status, poskusi, idempotency ključ |

### Omejitve in indeksi

- `CHECK` za pozitivno trajanje, ceno ≥ 0, delovni konec po začetku, veljavne statuse in logične meje `lead_days`.
- UNIQUE na domeni, članstvu, povezavi zaposlenega/storitve in nastavitvah salona.
- Indeksi po `salon_id` in pogostih filtrih: rezervacije `(salon_id, starts_at)`, `(salon_id, employee_id, starts_at)`, storitve/zaposleni `(salon_id, active)`.
- Aktivne rezervacije: GiST exclusion na `(salon_id WITH =, employee_id WITH =, tstzrange(starts_at, ends_at, '[)') WITH &&)`, omejeno na statuse, ki zasedajo termin. `btree_gist` se vključi v migraciji, če ga Supabase okolje podpira. Interval `[)` omogoča zaporedna termina brez lažnega trka.
- V bazi hranimo trenutke v UTC (`timestamptz`); urniki in izjeme so lokalni čas/koledarski datum v časovnem pasu salona. Pretvorba upošteva prehode poletnega časa. Neveljaven ali dvoumen lokalni čas je treba obravnavati eksplicitno.
- Cena in trajanje ob rezervaciji sta posnetek takratnih vrednosti, tako da kasnejša sprememba cenika ne spremeni zgodovine.

### RLS in dostop

- RLS vključimo na vseh tenant tabelah, tudi na povezovalnih in pomožnih tabelah; anon/authenticated pravice damo samo, kjer so potrebne. Politike same ne nadomestijo SQL `GRANT` omejitev.
- `authenticated` uporabnik bere/piše samo vrstice salonov, kjer ima članstvo z ustrezno vlogo. Vse politike vključujejo preverjanje tenant identitete; `staff` dobi samo dovoljene podatke in opravila.
- Javni anon dostop je omejen na javne podatke aktivnega salona (npr. objavljene storitve, javni zaposleni in javni urnik). PII strank in administrativne tabele niso javne.
- Anonimna rezervacija gre prek strežniškega endpointa in transakcijskega RPC-ja z validacijo salon/storitev/zaposlenega, rate limitom in zaščito pred avtomatizacijo. RPC ima določen `search_path`, minimalne pravice in preveri vse argumente. Ni neposrednega `INSERT` za anon.
- `service_role` je samo v strežniških skrivnostih, nikoli v `NEXT_PUBLIC_*`, klientu ali dnevnikih. Za redne zahteve se uporablja uporabniška Supabase seja, da RLS ohrani kontekst.
- Brisanje/anonimizacija stranke mora določiti, kako ravnati z rezervacijami in morebitnimi zakonskimi roki hrambe; to je odprta pravna in produktna odločitev, ne trditev o skladnosti z GDPR.

## 4. Tok rezervacije in preprečevanje dvojnih terminov

1. Strežnik razreši domeno, naloži aktivni salon in javna pravila.
2. Razpoložljivi termini se izračunajo na strežniku iz storitvenega trajanja, dodelitve storitve, urnika zaposlenega (individualni prepiše/zoži osnovnega po dogovorjenem pravilu), odmorov, izjem/dopusta in aktivnih rezervacij. Brskalnik samo prikaže rezultat.
3. Potrditev pošlje izbrane podatke na strežniški endpoint; strežnik znova preveri salon, storitev, zaposlenega, časovno okno, podatke stranke in omejitve rezerviranja.
4. Ena DB transakcija izbere zaposlenega (pri »katerikoli« po seznamu usposobljenih), preveri urnik/izjeme ter vstavi rezervacijo in outbox dogodek. Za kandidata ali nabor zaposlenih se uporabi zaklepanje, kjer je potrebno.
5. Exclusion constraint je zadnja avtoriteta tudi ob dveh sočasnih zahtevah. Konflikt se vrne kot razumljiv »termin je bil pravkar zaseden«; pri »katerikoli« lahko strežnik poskusi naslednjega upravičenega zaposlenega.
6. Outbox pošlje potrditev stranki in obvestilo salonu; dostava ni del transakcije rezervacije. Idempotency key prepreči podvojene e-pošte ob ponovnem poskusu.
7. Sprememba/preklic prek povezave preveri enkratni, časovno omejen, zgoščeni token, ponovno uporabi enaka pravila konfliktov in zapiše dogodek.

## 5. Varovala aplikacije

- Validacija strežniških vhodov s shemami; izhodne podatke za javni prikaz oblikujemo po principu najmanjših pravic.
- Omejevanje zahtevkov po IP/domeni in poslovnem ključu; uporabiti zanesljivo skupno shrambo limiterja pri več instancah. Ne zanašati se le na pomnilnik procesa.
- CSRF zaščita za mutacije s piškotki, varni piškotki, preverjanje izvora in CSP; omejiti in validirati upload logotipov/fotografij.
- Noben PII ali skrivnost v URL-jih, analitiki ali dnevnikih. Podatke in retencijo določiti po načelu najmanjšega obsega.
- E-poštni webhooki podpisani in idempotentni; datoteke v Storage imajo tenant prefiks in politike, javni dostop samo za namerno javne assete.
- Centralna administracija zahteva močnejši nadzor (MFA), revizijski zapis sprememb domen, članstev, statusov in nastavitev.

## 6. Fazni načrt izvedbe

1. **Arhitektura (ta dokument):** potrditi meje tenantov, booking pravila, domensko usmerjanje in nejasnosti spodaj.
2. **Osnova Next.js:** App Router, TS, Tailwind, lint/format, okolja in CI; brez produkcijskih skrivnosti v repozitoriju.
3. **Supabase povezava:** ločena server/browser klienta, Auth SSR in lokalni razvojni projekt.
4. **Shema in RLS:** migracije, indeksi, omejitve in RLS/GRANT politike; avtomatski testi izolacije.
5. **Avtentikacija/pravice:** vabila uporabnikov, članstva, vloge, platformni admin, strežniško preverjanje.
6. **Upravljanje salonov:** salon, domena, nastavitve, media upload in audit.
7. **Rezervacijski API:** urniki, prosti termini, transakcijska rezervacija, dvojni sočasni poskus.
8. **Administracija:** pregled koledarja, ročne rezervacije, blokade, stranke, cenik, zaposleni in urniki.
9. **Predloge:** tri ločene kompozicije, konfiguracije in mobilni prikazi.
10. **Več domen:** preverjanje domen, DNS/TLS, neznan tenant in deaktivacija.
11. **E-pošta:** outbox, predloge sporočil, varne povezave, retry in nadzor napak.
12. **Zanesljivost in varnost:** testni scenariji, RLS pregled, obremenitev terminov, backup/restore, monitoring, politika zasebnosti in pravna presoja pred trditvami o skladnosti.

Po vsaki fazi se izvedejo ustrezni testi in pregled; ta dokument sam po sebi ne pomeni, da so funkcije ali varnost že izvedene.

## 7. Odločitve pred implementacijo

Pred začetkom faze 2 je treba določiti ali potrditi:

- Ali salon lahko aktivira rezervacijo šele po ročni potrditvi ali so vse rezervacije takoj potrjene?
- Pravilo za »katerikoli zaposleni« pri izbiri kandidata (prvi prosti, kroženje, uravnoteženje obremenitve).
- Politika roka odpovedi/spremembe in ali jo vsak salon nastavlja sam.
- Ali se stranke lahko združujejo po e-pošti/telefonu ali se ustvarijo ločeno za vsako rezervacijo; predlagano je tenant-ločeno ujemanje po normaliziranem telefonu/e-pošti.
- Ali zaposleni vidijo vse stranke/rezervacije ali samo svoje; predlagano je omejiti po vlogi in salonu.
- Izbira gostitelja (Vercel/Cloudflare), Supabase regija in e-poštni ponudnik/pošiljateljska domena.
- Platformni admin račun bo ustvarjen varno prek ročnega bootstrap postopka, ne s privzetim geslom ali javno registracijo.

## Viri

- [Next.js App Router](https://nextjs.org/docs/app)
- [Supabase Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [Supabase SSR Auth](https://supabase.com/docs/guides/auth/server-side)
- [PostgreSQL range types in exclusion constraints](https://www.postgresql.org/docs/15/rangetypes.html)
