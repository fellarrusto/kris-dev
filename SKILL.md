---
name: kris-dev
description: Usa per creare, migrare o modificare siti con Kris CMS, in locale o tramite Kris Agent MCP: conversioni di siti esistenti, template, schema, contenuti, staging e deploy.
---

# Sviluppo con Kris CMS

Kris e' un CMS PHP a file JSON, senza database e senza build. Il riferimento completo e' `AGENTS.md` nella root del CMS (repo: https://github.com/fellarrusto/kris-cms): in locale leggilo prima di iniziare e, se contraddice questa skill, vale il codice della versione installata (`kris/VERSION`).

## 1. Scegli la modalita'

- **Sito gia' online con contenuti reali:** usa l'MCP. Non sovrascrivere mai `data/`, `config/` o `assets/uploads/` del sito live.
- **Sito nuovo o migrazione:** lavora in locale, consegna con il primo deploy, poi passa all'MCP.
- **Cartella con un vecchio sito:** non modificare gli originali. Spostali in `_sorgente/`, installa Kris accanto e converti.

Confini dell'MCP: tocca template, CSS, JS, SVG, contenuti, schema, pagine, upload e font. Non tocca `.htaccess`, `robots.txt`, `config/seo.json` (salvo sitemap via `register_page`), `base_url`, PHP o `kris/`, e non pubblica. Queste cose vanno elencate all'utente per l'FTP o la pubblicazione.

## 2. Modellare i contenuti

Prima di scrivere, fai una mappa: pagina -> template -> raccolta -> campi. Una cosa sta in un solo posto.

- **Raccolta propria** per ogni cosa con vita autonoma (avviso, orari, promo): non infilarla nella homepage.
- **Dati comuni** (logo, menu, footer, contatti, WhatsApp, immagine social): raccolta `site` a contenuto unico.
- **Header e footer:** sempre `k-component`, mai copiati tra pagine.
- **404:** usa `kris_notfound` e la sezione dell'editor. Non creare la raccolta `not_found`: se esiste, vince sulla sezione del framework e duplica i testi (e' solo compatibilita' con i siti precedenti alla 1.8.0). Se `kris_notfound` manca, il sito non ha ancora applicato la migrazione: controlla la versione e apri l'editor.
- **Favicon:** `kris_site.favicon`, impostata in tutte le lingue attive; nessun tag nel template.
- **Privacy e cookie:** `kris_legal`. Non inventare dati societari o cookie.

Tipi disponibili: `plain`, `text`, `richtext`, `image`, `array`. Nomi in `snake_case`; mai `id` o `language`; prefisso `kris_` riservato. Il testo con HTML richiede `richtext`.

Ordine dell'editor, obbligatorio per ogni campo nuovo:

- assegna `section` a ogni campo: i campi senza sezione finiscono in testa e confondono;
- inserisci il campo vicino ai suoi simili con `move_field`, non lasciarlo in fondo;
- scrivi una `description` utile al cliente e aggiornala con `describe_field` quando il significato cambia;
- dai a ogni raccolta `label`, `description` e, se ha un solo contenuto, `single: true`.

## 3. Template

Sintassi: `{{campo}}`, `{{language}}`, `{{id}}`, `{{#if a == "b"}}...{{#elif}}...{{#else}}...{{/if}}`, operatori `== != > < >= <= has`, `k-array` + `k-template`, `k-component` + `k-template` + `k-index`. Niente filtri, `parent.x` o loop.

- Il segnaposto `k-component` viene sostituito: tag, classi e attributi vanno nel frammento.
- Il contenitore `k-array` resta nel DOM e deve essere vuoto.
- Dentro un frammento vale solo l'entita' del frammento: il padre non e' accessibile.
- Un frammento di lista deve iniziare con un tag: se si apre con una condizione, Kris non lo mostra. Metti l'`{{#if}}` dentro l'elemento radice.
- Niente condizioni dentro il tag di apertura del frammento: vengono ignorate. Usale nel contenuto, e controlla il risultato renderizzato.
- Ogni testo, URL, numero o immagine che il cliente puo' cambiare viene da un campo. Niente indirizzi, telefoni o immagini social scritti nel template.
- Link: `index.php?page=x&amp;key=y&amp;id=0&amp;ln={{language}}`. Le pagine usano query string: Kris non ha URL puliti.
- Asset: percorsi `assets/...` senza `/` iniziale, cosi' funzionano in sottocartella. Niente `?v=` a mano.
- **Header fisso:** deve stare fuori dalla hero, in `position: sticky` o `fixed` con spazio compensato. L'ancora di ritorno `#top` va su un elemento in cima alla pagina, non sull'header fisso.
- **Font:** self-hosted in `.woff2`. Kris salva il nome in minuscolo: `Archivo-Variable.woff2` diventa `archivo-variable.woff2`, e il `@font-face` deve usare il nome finale (massimo 2 MB).
- **JS:** solo ASCII negli script, usa gli escape (`·`, non `·`). Niente valori interpolati dentro il codice: passali con attributi `data-*`.
- Niente redirect in JS o nella 404: i vecchi indirizzi si gestiscono con un 301 in `.htaccess`.
- I nomi dei file del tema accettano solo minuscole, cifre, `-` e `_`.

## 4. Lavorare con l'MCP

Orientamento: `list_sites`, poi `get_site_overview`. Non indovinare le operazioni: l'elenco valido e' nella descrizione di `apply_operation`, e l'operazione `help` non esiste. Le raccolte si creano con `create_collection` (`name`, `fields`), mai `add_collection`.

**Ordine obbligatorio:**

1. `create_collection`, con tutti i campi gia' definiti;
2. `create_entity`, `set_collection_info`;
3. template e CSS;
4. `register_page`: fallisce se il template non esiste;
5. contenuti e liste;
6. `get_staging_status`.

**Scrivere i contenuti:**

- `add_item` accetta i valori solo nella chiave `fields`. Qualsiasi altra chiave viene ignorata senza errore e crea un elemento vuoto.
- I campi `array` dentro `fields` vengono scartati senza errore: crea il padre, prendi `assigned_id`, poi aggiungi i figli con `add_item` e `path`.
- Una stringa semplice in un campo per lingua scrive solo la lingua principale. Passa sempre `{"it": "...", "en": "..."}`.
- Dopo il primo inserimento di una serie, controlla con `get_entity` che i valori ci siano, poi prosegui. La risposta "registrata" non prova che i dati siano stati scritti.
- I valori `image` devono essere file gia' caricati (`assets/uploads/...`).

**File:**

- `read_file`, poi una sola modifica per volta con l'hash ricevuto.
- Non lanciare mai in parallelo piu' scritture sullo stesso file: la seconda viene rifiutata. Rileggi per avere il nuovo hash.
- `edit_file`: copia `old` dal testo appena letto, non a memoria; deve comparire una sola volta.
- Per modifiche ampie usa `write_file` una volta sola.

**Upload:** carica solo file definitivi, mai di prova. L'MCP non cancella gli upload: un file di troppo resta fino all'FTP. Usa `create_upload_slot` e `curl -T`; riserva `upload_image` a file piccoli.

**Anteprima:** gli script nuovi non girano finche' l'admin non li approva. Non dichiarare testate le interazioni senza averle viste. Prova il JS in una copia locale, oppure chiedi all'utente di approvare e controllare. La 404 si vede su un indirizzo inesistente con `?kris_preview=1`.

`discard_changes` solo su richiesta esplicita dell'utente.

## 5. Sviluppo locale

```sh
git clone --depth 1 https://github.com/fellarrusto/kris-cms .
rm -rf .git
php -S 127.0.0.1:8000 -t .
```

Apri `/editor/` per il setup e le migrazioni. Non serve MySQL, Composer o Node.

- Modifica `template/`, `assets/`, `data/k_model.json`, `data/k_data.json`, `config/allowed_pages.json`, `config/seo.json`.
- Sostituisci i contenuti demo: template, schema e dati non usati non devono restare.
- `k_data.json` e' una lista di entita'. Ogni campo ha `name`, `type`, `value`; `text`, `richtext` e `image` hanno valori per lingua, `plain` uno scalare.
- Se il sito vive gia' online, ricorda che `tools/staging.php` prova le stesse operazioni dell'MCP (`op`, `status`, `read`, `edit`) su una copia locale.
- Non modificare `kris/` e gli stub: un aggiornamento li sostituisce.

## 6. Migrare un vecchio sito

1. **Inventario:** elenca pagine, URL attuali, lingue, immagini, PDF, font, form, script terzi, cookie, testi SEO (title, description, og). Mostra la mappa all'utente prima di costruire.
2. **Conserva il design:** stessa identita', testi fedeli. Non riscrivere i contenuti se non richiesto.
3. **Tema:** estrai header e footer in componenti, le ripetizioni in frammenti `k-array`, i valori ricorrenti in variabili CSS.
4. **Contenuti:** trasferisci testi e immagini nei JSON; gli upload vanno in `assets/uploads/` con nomi leggibili; nessun link verso il vecchio dominio o file locali.
5. **Cosa Kris non fa:** form, invio email, prenotazioni, pagamenti e ricerca server non esistono. Elencali e chiedi come gestirli: non fingere che funzionino.
6. **Redirect:** ogni vecchio URL con traffico ottiene un 301 in `.htaccess`, prima della regola 404 di Kris:

```apache
RewriteRule ^chi-siamo\.html$ /?page=chi-siamo&key=chi_siamo [R=301,L]

RewriteCond %{QUERY_STRING} ^page=forno$
RewriteRule ^$ /?page=pane [R=301,L]
```

In sottocartella aggiungi il prefisso alla destinazione. Prova ogni regola su Apache reale: il server PHP locale ignora `.htaccess`.

7. **SEO:** registra ogni tipo di pagina in `config/seo.json`, imposta `base_url`, crea `robots.txt` con `Disallow: /editor/` e l'URL della sitemap. Non bloccare `/kris/` o `/assets/`.
8. **Cookie:** script di terzi bloccati con `type="text/plain"` e `data-kris-consent-script`, banner collegato a `kris_legal`.

## 7. Verifica prima di consegnare

Guarda le pagine renderizzate con un browser, non il codice.

- Larghezze 375, 768, 1280 e 1440: testi lunghi, titoli con font larghi, nessuno sforamento dalla colonna.
- Tutte le lingue, testi vuoti, liste vuote, immagini mancanti.
- Header che resta visibile allo scroll e logo che riporta in cima; menu, tastiera, focus.
- Console senza errori, nessun 404 su asset e font (controlla la scheda rete).
- 404 con stato 404, redirect 301, `sitemap.php`, canonical, title e description su ogni pagina.
- Editor: modifica un testo, un'immagine e l'ordine di una lista e controlla il sito. Ogni campo in sezione, con descrizione attuale, nessun campo inutilizzato.
- Controlli rapidi:

```sh
grep -nP '[^\x00-\x7F]' assets/js/*.js
grep -rnE 'https?://|wa\.me|tel:' template/
grep -n 'url(' assets/css/*.css
node --check assets/js/*.js
```

## 8. Deploy

**Primo rilascio via FTP:** tutto il sito, esclusi `.git/`, `_sorgente/`, `tests/`, `tools/`, documenti di sviluppo e `config/auth.php` locale. `data/` e `assets/uploads/` devono essere scrivibili da PHP. Non lasciare il setup aperto: crea subito l'account. Controlla che `config/` e `data/` non siano scaricabili.

**Rilasci successivi:** via MCP, oppure solo template e asset del tema. Mai `data/`, `config/` o upload dall'ambiente locale.

**Consegna:** con l'MCP dai il link di anteprima, ricorda che va aperto da admin con accesso all'editor, che gli script vanno approvati e che si pubblica da Editor > Da pubblicare. Dichiara sempre cosa hai verificato, cosa no, e l'elenco delle azioni manuali (FTP, `.htaccess`, file da rimuovere).
