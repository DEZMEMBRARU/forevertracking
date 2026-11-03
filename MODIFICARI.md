# Ce s-a schimbat (optimizare viteză)

## Cauza lentorii
Fiecare ecran se demonta la navigare și, la revenire, **își cerea din nou toate datele de la zero**,
iar interfața rămânea goală până răspundea baza de date. În plus, lista de alimente (`foods`) era
descărcată integral de mai multe ori, sub 3 chei de cache diferite care nu se refolosea niciodată.

## Soluția: cache „stale-while-revalidate" (`src/lib/cache.js`)
- Revii pe un ecran → vezi **instant** ce ai văzut data trecută; datele proaspete vin în fundal.
- Orice scriere în baza de date (insert/update/upsert/delete) **invalidează automat** cache-ul
  (vezi `src/lib/supabase.js`), deci nu apar date vechi după ce salvezi ceva.
- Date rare (lista de alimente, citatul zilei): cache 10 min, zero cereri cât sunt proaspete.
- Cererile identice simultane se unesc într-una singură.

## Pe pagini
- **Acasă:** nu mai descarcă tot tabelul `foods` la deschidere (doar când bifezi un produs din
  lista de cumpărături). Suplimente + antrenamente: cereri în paralel, nu una după alta.
- **Nutriție:** `foods` se cere o singură dată pentru toate tab-urile; suplimentele luate azi vin
  în aceeași rundă cu restul.
- **Sport / Plan:** de la 5–7 cereri una după alta la 2 runde în paralel.
- **Sport / Statistici:** grafice (recharts, ~105 KB gzip) într-un fișier separat, încărcat doar
  când deschizi tab-ul. Înainte se încărca la orice intrare în /sport.
- **Meniul de jos:** un singur Footer, nu se mai recreează la fiecare navigare.
- **Navigare:** paginile Nutriție și Sport se descarcă în fundal după pornire; ecranul vechi rămâne
  vizibil cât se încarcă cel nou (fără flash de „Se încarcă...").
- **Autentificare:** profilul nu se mai cere la fiecare revenire în tab / refresh de token.

## Build
- Scos `@vitejs/plugin-legacy`: genera a doua copie a întregii aplicații + polyfills.
  Precache PWA: 2936 KiB → 1437 KiB (41 → 27 fișiere).
- Biblioteci mari (react, supabase) în fișiere separate, ca un deploy nou să nu le invalideze.
- Fonturile Google nu mai blochează afișarea, și se țin în cache de service worker.
- Corectat calea iconițelor PWA (`/icons/...`).
- Curățenie: scoase 4 fișiere duplicat din rădăcină (App.jsx era o versiune veche, fără Raport și
  Fitness RPG; main.jsx, index.css, manifest.json erau identice cu cele din src/public).

## Bug-uri găsite și reparate pe parcurs
- **Calendar:** ultima zi a lunii nu era inclusă în interogare (`toISOString()` pe dată locală
  o mută cu o zi în urmă în România).
- **Statistici:** graficele luau cele mai VECI 20 de alergări / 60 de antrenamente, deci nu se mai
  actualizau după ce treceai de acest număr. Acum iau cele mai noi.
- **Plan (admin):** numele elevilor din feedback apărea mereu „Elev" (citea state-ul înainte să se
  actualizeze).

## Fișiere modificate
src/lib/cache.js, src/lib/supabase.js, src/App.jsx, src/components/Footer.jsx,
src/pages/Dashboard.jsx, src/pages/Nutritie.jsx, src/pages/Sport.jsx,
src/pages/sport/StatisticiTab.jsx (nou), vite.config.js, index.html, package.json


---

# XP din tot (apă, nutriție, suplimente, antrenamente)

## Ce s-a schimbat
Înainte, XP venea doar din activitățile introduse manual în modulul RPG. Acum se acordă automat și pentru:

| Sursă | XP |
|---|---|
| Apă | 50% din țintă +8 · țintă atinsă +12 |
| Nutriție | masă înregistrată +5 · 3 mese +10 · proteine la țintă +15 · calorii între 85–110% din țintă (la sfârșit de zi) +20 |
| Suplimente | fiecare luat +3 · toate luate +10 |
| Alergare (Sport + Strava) | 5 / km (min 5, max 60) |
| Bicicletă | 2 / km (min 8, max 50) |
| Forță | +20, plus 2 / set (max +20) |
| Alt antrenament / plan bifat | +10 / +10 |
| Zi completă (apă + mese + suplimente + mișcare) | +25 |

Plafonul de 150 XP/zi se aplică doar antrenamentelor (RPG + Sport + Strava). Apa, nutriția și suplimentele nu au plafon.
Atribut nou: **VITALITY** (apă + nutriție + suplimente). Pagina RPG are cardul „Cum câștigi XP".

## Cum funcționează
- Funcția SQL `sync_xp` citește datele reale din baza de date și acordă XP. Clientul nu trimite nicio sumă.
- Fiecare premiu are o cheie unică: se acordă o singură dată, chiar dacă ștergi și reintroduci datele.
- Aplicația o declanșează automat după orice salvare reușită (cu mică întârziere, ca salvările în rafală să fie
  o singură sincronizare), la deschidere (ultimele 3 zile) și la revenirea în aplicație.
- Apare o notificare „+XP" (cu nivel nou, dacă e cazul). Cardul de pe Acasă se actualizează singur.
- Istoricul dinainte de actualizare NU primește XP automat; există butonul „Adaugă XP și pentru ultimele 30 de zile"
  în Fitness RPG → Prezentare → Cum câștigi XP.

## Instalare (obligatoriu)
1. Supabase → SQL Editor → rulează `supabase/migrations/202611030001_xp_everything.sql` (după migrarea 202611020001).
   Se poate rula de mai multe ori fără probleme.
2. Până rulezi migrarea, aplicația funcționează normal, iar pagina Fitness RPG arată un avertisment.

## Repară și o eroare veche
`award_activity_xp` dădea eroare „column reference total_xp is ambiguous" la a doua apelare pe aceeași activitate.

## Fișiere
Nou: supabase/migrations/202611030001_xp_everything.sql, src/lib/xp.js, src/components/XpToast.jsx
Modificat: src/lib/supabase.js, src/lib/cache.js, src/lib/fitness/rpg.js, src/App.jsx,
src/pages/fitness/FitnessRpg.jsx, src/components/FitnessDashboardHero.jsx, src/components/StreakCard.jsx
(StreakCard: folosea data UTC; între 00:00 și 02:00/03:00 „azi" era ieri și streak-ul se rupea)
