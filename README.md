# Salon multi-tenant platform

Začetna Next.js App Router osnova za več salonov. Javna stran je odzivna in ima tri kompozicijsko različne predloge. Lokalni predogled uporablja jasno označeno demo vsebino. Rezervacije trenutno niso vključene: stran pošteno pove, da se termin ne shrani, in ponudi klic/e-pošto.

## Lokalni zagon

1. Namestite Node.js in npm, nato v tej mapi zaženite `npm install`.
2. Kopirajte `.env.example` v `.env.local`.
3. Za lokalni prikaz brez Supabase pustite Supabase spremenljivki prazni. Za domene po meri pripravite Supabase projekt, izvedite SQL migracijo in nastavite URL ter anon ključ.
4. Zaženite `npm run dev` in odprite `http://localhost:3000`.

Predloge si lahko ogledate na `/predogled/noir`, `/predogled/atelier` in `/predogled/afterdark`. Vsaka domena v produkciji se razreši prek preverjene vrstice v `salon_domains`.

## Pomembno pred uporabo

- `supabase/migrations/202611120001_initial_schema.sql` je začetna shema/RLS osnova, ne varnostno pregledana produkcijska migracija. Najprej jo je treba preizkusiti na ločenem Supabase projektu; javna rezervacijska RPC namerno še ni dodana.
- V tej izvedbi ni prostih terminov, ustvarjanja rezervacij, pošiljanja e-pošte, urejevalnika salona ali platformnega nadzornega panela. Te funkcije ne smejo biti predstavljene kot delujoče.
- `.env.local` skrivnosti ne shranjujte v Git. Service-role ključ se ne sme uporabljati v brskalniku; pripravljeno okolje ga namenoma ne potrebuje.
- Primer imen, lokacije, cen, urnika in fotografij na lokalnem predogledu je predstavitvena vsebina, ne dejanski salon ali ponudba.

## Naslednja faza

Najprej povežite ločen razvojni Supabase projekt, pregledajte in popravite migracijo ter dodajte teste izolacije RLS. Nato zgradite strežniško preverjanje terminov in transakcijski rezervacijski RPC, preden se rezervacijski obrazec aktivira.

