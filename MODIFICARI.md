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
