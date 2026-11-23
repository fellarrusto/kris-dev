---
name: kris-dev
description: Usa per creare, migrare o modificare siti con Kris CMS, in locale o tramite Kris Agent MCP: conversioni di siti esistenti, template, schema, contenuti, indirizzi e redirect, plugin, staging e deploy.
---

# Sviluppo con Kris CMS

Kris e' un CMS PHP senza build: i dati sono documenti JSON (in `data/`, o in un database SQLite/MySQL), il sito pubblico e' HTML, CSS e JS reso da PHP. Questa skill vale per la 2.1. Il riferimento completo e' `AGENTS.md` nella root del CMS (repo: https://github.com/fellarrusto/kris-cms): in locale leggilo prima di iniziare e, se contraddice questa skill, vale il codice della versione installata (`kris/VERSION`). Con l'MCP la versione del sito e le operazioni valide sono nella descrizione degli strumenti.

## 1. Scegli la modalita'

- **Sito gia' online con contenuti reali:** usa l'MCP. Non sovrascrivere mai `data/`, `config/` o `assets/uploads/` del sito live.
- **Sito nuovo o migrazione:** lavora in locale, consegna con il primo deploy, poi passa all'MCP.
- **Cartella con un vecchio sito:** non modificare gli originali. Spostali in `_sorgente/`, installa Kris accanto e converti.

Confini dell'MCP: tocca template, CSS, JS, SVG, contenuti, schema, pagine e indirizzi (`register_page`, `set_page`, `set_sitemap`), reindirizzamenti (`set_redirect`), upload, font, i siti ammessi per gli embed e le impostazioni dei plugin; chiede l'installazione di un plugin. Non tocca `.htaccess`, `robots.txt`, `base_url`, lingue, PHP o `kris/`, non accende plugin e non pubblica: queste cose le fa l'admin dall'editor (o via FTP) e vanno elencate all'utente.

## 2. Modellare i contenuti

Prima di scrivere, fai una mappa: pagina -> template -> tipo -> campi. Una cosa sta in un solo posto.

- **Tipo proprio** per ogni cosa con vita autonoma (avviso, orari, promo, progetti): non infilarla nella homepage. Progetti, news e articoli con pagina di dettaglio sono un tipo con i suoi contenuti, non un elenco; un blog e' un tipo in Posts (`set_post_collection`).
- **Dati comuni** (logo, menu, footer, contatti, WhatsApp): tipo `site` a contenuto unico.
- **Header e footer:** sempre `k-component`, mai copiati tra pagine.
- **Proprieta' del sito** (`kris_site`): nome, modello del titolo, descrizione e immagine di condivisione predefinite, verifiche di Google e Bing, favicon. Un tipo con i campi `seo_title`, `seo_description`, `seo_image` li usa per le sue pagine.
- **404:** `kris_notfound` e la sezione dell'editor. Non creare la raccolta `not_found`.
- **Lavori in corso:** `kris_wip`.
- **Privacy e cookie:** `kris_legal`. Non inventare dati societari o cookie.

Tipi di campo: `text`, `richtext`, `media`, `select` (con `options`, eventualmente `multiple`), `boolean`, `list` (con `of`) ed `embed`. Tutti tranne `list` hanno `localized`: `true` un valore per lingua (`{"it": "...", "en": "..."}`), `false` uno solo. Nomi in `snake_case`; mai `id` o `language`; prefissi `kris_` e `plg_` riservati. Il testo con HTML richiede `richtext`, che esce filtrato.

- **Embed** (video, post, mappe di altri siti): un campo `embed`, mai un iframe in un `richtext` (il filtro lo toglie). Il valore e' l'indirizzo della pagina (YouTube, Vimeo, Instagram, TikTok, Google Maps, Spotify) o il codice di incorporamento; `{{campo}}` diventa l'`<iframe>`. Altri siti: `set_embed_domains`.
- **Immagini nei richtext:** scrivi `width` e `height`; Kris aggiunge `loading="lazy"`.

Ordine dell'editor, obbligatorio per ogni campo nuovo:

- assegna `section` a ogni campo: i campi senza sezione finiscono in testa e confondono;
- inserisci il campo vicino ai suoi simili con `move_field`, non lasciarlo in fondo;
- scrivi una `description` utile al cliente e aggiornala con `describe_field` quando il significato cambia;
- dai a ogni tipo `label`, `description` e, se ha un solo contenuto, `single: true` (`set_collection_info`).

## 3. Template

Sintassi: `{{campo}}`, `{{language}}`, `{{id}}`, `{{tipo.campo}}` (il primo contenuto di un altro tipo), `{{#if a == "b"}}...{{#elif}}...{{#else}}...{{/if}}`, operatori `== != > < >= <= has`, `k-array` + `k-template`, `k-component` + `k-template` + `k-index`. Niente filtri, `parent.x` o loop.

- Il segnaposto `k-component` viene sostituito: tag, classi e attributi vanno nel frammento.
- Il contenitore `k-array` resta nel DOM e deve essere vuoto. `k-array="progetti"` su un tipo che non ha quell'elenco rende i contenuti del tipo `progetti`.
- Dentro un frammento vale solo il contenuto del frammento: il padre non e' accessibile.
- Un frammento di lista deve iniziare con un tag. Niente condizioni dentro il tag di apertura del frammento: usale nel contenuto, e controlla il risultato renderizzato.
- Ogni testo, URL, numero o immagine che il cliente puo' cambiare viene da un campo.
- **Link:** `index.php?page=x&amp;key=y&amp;id=3&amp;ln={{language}}`. Kris li trasforma da solo negli indirizzi leggibili (`/progetto/casa-rossi`, `/en/menu`): non scrivere a mano i percorsi leggibili. Una pagina con contenuto fisso basta `index.php?page=x&amp;ln={{language}}`; i parametri in piu' restano in coda (`/preventivo?servizio=social`).
- **Selettore della lingua:** `<a href="?ln=en" hreflang="en" data-kris-lang="en">EN</a>`: porta alla stessa pagina nell'altra lingua. Niente `onclick`: per i filtri `data-kris-filter` (vedi `kris/core/scripts/script.js`).
- **Head:** Kris aggiunge canonical, hreflang, description, Open Graph, title dal modello e favicon, tag per tag, solo dove il template non li ha. Nel template scrivi solo i tag che cambiano (per esempio `og:image` con la copertina del progetto); tutto cio' che serve a ogni pagina (dati strutturati, script globali) va in `template/kris_head.html`.
- Asset: percorsi `assets/...` senza `/` iniziale, cosi' funzionano in sottocartella e sotto `/en/`. Niente `?v=` a mano.
- **Header fisso:** fuori dalla hero, in `position: sticky` o `fixed` con spazio compensato. L'ancora di ritorno `#top` va su un elemento in cima alla pagina.
- **Font:** self-hosted in `.woff2` (massimo 2 MB). Kris salva il nome in minuscolo: `Archivo-Variable.woff2` diventa `archivo-variable.woff2`, e il `@font-face` deve usare il nome finale.
- **JS:** solo ASCII negli script, usa gli escape (`·`, non il carattere). Niente valori interpolati dentro il codice: passali con attributi `data-*`.
- Niente redirect in JS o nella 404: vedi la sezione 5.
- I nomi dei file del tema accettano solo minuscole, cifre, `-` e `_`.

## 4. Lavorare con l'MCP

Orientamento: `list_sites`, poi `get_site_overview` (lingue, pagine, tipi, contenuti, file del tema, plugin) e `get_site_config` (pagine, SEO, redirect, sitemap, embed ammessi). Non indovinare le operazioni: l'elenco valido e' nella descrizione di `apply_operation`, e l'operazione `help` non esiste. I tipi si creano con `create_collection` (`name`, `fields`), mai `add_collection`.

**Ordine obbligatorio per una pagina nuova:**

1. `create_collection`, con tutti i campi gia' definiti;
2. `create_entity`, `set_collection_info`;
3. template e CSS;
4. `register_page` con `template`, `type`, `item` (un contenuto fisso, o `null` per una pagina per contenuto), `slug` e `sitemap`: fallisce se il template non esiste. Per cambiare una pagina esistente `set_page`;
5. contenuti ed elenchi;
6. `get_staging_status`: leggi gli avvisi e le `preview_pages`.

Usa `apply_operations` per gruppi di modifiche: tutte o nessuna.

**Scrivere i contenuti:**

- `add_item` e `create_entity` accettano i valori solo in `fields` e rifiutano chiavi sconosciute. Un elenco annidato si passa in `fields` come lista di elementi.
- Un campo per lingua con una stringa sola scrive la lingua principale e svuota le altre: passa sempre `{"it": "...", "en": "..."}` o `"lang"`. Leggi i `warnings` della risposta.
- **Gli id non sono l'ordine:** `get_entity` da' a ogni elemento `position` (0 = il primo mostrato), ma le operazioni vogliono l'`id`. Prima di modificare un elemento controlla l'id, e dopo il valore `before` nella risposta.
- I valori `media` devono essere file gia' caricati (`assets/uploads/...`).

**File:**

- `read_file` (anche a pezzi o con `search`) e `search_theme` per trovare il testo; poi una modifica per volta.
- Non lanciare mai in parallelo piu' scritture sullo stesso file: la seconda viene rifiutata.
- `edit_file`: copia `old` dal testo appena letto, non a memoria; deve comparire una sola volta.
- Per modifiche ampie usa `write_file` una volta sola. `delete_file` rifiuta il template che una pagina usa: prima `set_page` o `unregister_page`.
- `render_page` restituisce l'HTML finito di una pagina con le modifiche in attesa: usalo per controllare head, link e liste.

**Upload:** carica solo file definitivi. `create_upload_slot` da' un indirizzo sul dominio del sito: manda il file con `curl -T`; riserva `upload_image` ai file piccoli. Riduci le foto alla larghezza d'uso prima di caricarle (Kris crea comunque le varianti 480/960/1600 alla pubblicazione). `delete_upload` toglie un file in attesa; `list_unused_uploads` elenca quelli che nessuno usa.

**Plugin:** `list_plugins` (con `catalog: true` anche quelli installabili) dice cosa e' installato e acceso e cosa aggiunge alle pagine: controllalo prima di scrivere a mano analytics, banner o tag nel head. `install_plugin` lo installa alla pubblicazione, spento: lo accende l'admin. Le impostazioni con `set_plugin_settings` (dopo l'installazione).

**Anteprima:** gli script nuovi non girano finche' l'admin non li approva. Non dichiarare testate le interazioni senza averle viste. La 404 si vede su un indirizzo inesistente con `?kris_preview=1`.

`discard_changes` solo su richiesta esplicita dell'utente.

## 5. Indirizzi e redirect

- Gli indirizzi sono leggibili: `/`, `/menu`, `/progetto/3`, `/en/menu`. Con `slug` su una pagina per contenuto l'indirizzo viene da un campo di testo (`/progetto/casa-rossi`).
- **Mai redirect nel `.htaccess`.** Kris li gestisce dai dati: `set_redirect` (`from`, `to`; `to` vuoto lo toglie), e `/vecchio/*` vale per tutto quello che c'era sotto. Nell'editor: Indirizzi > Reindirizzamenti.
- Cambiando uno slug, Kris aggiunge da solo il 301 dal vecchio indirizzo al nuovo, in ogni lingua.
- Togliendo una pagina: `unregister_page` con `redirect_to`, e i suoi vecchi indirizzi portano li'.
- Il `.htaccess` contiene solo la regola di Kris (e, se servono, https e www). Non usare il flag `QSD`: su alcuni hosting manda in errore 500 tutto il sito. Una regola che rimanda `/pagina` a `?page=pagina` crea un giro infinito con gli indirizzi leggibili: va tolta.
- Le intestazioni di sicurezza (nosniff, Referrer-Policy, X-Frame-Options, HSTS) le manda Kris: non aggiungerle al `.htaccess`.

## 6. Sviluppo locale

```sh
git clone --depth 1 https://github.com/fellarrusto/kris-cms .
rm -rf .git
php -S 127.0.0.1:8000 -t .
```

Apri `/kris-admin/` per il setup (serve il codice di primo avvio in `config/setup_code.php`). Non serve Composer, Node o un database; serve PHP 8.3.

- Modifica `template/`, `assets/` e i dati del formato 2: `data/schema.json` (tipi e campi, pagine in `pages`, `redirects`), `data/content.json` (lista dei contenuti con i valori per nome), `data/settings.json` (lingue, caricamenti, `seo`, `embed`). Il formato e' in `AGENTS.md`.
- Sostituisci i contenuti demo: template, schema e dati non usati non devono restare.
- `tools/staging.php` prova le stesse operazioni dell'MCP (`op`, `status`, `read`, `edit`) su una copia locale.
- Non modificare `kris/` e gli stub: un aggiornamento li sostituisce.

## 7. Migrare un vecchio sito

1. **Inventario:** pagine, URL attuali, lingue, immagini, PDF, video, embed, font, form, script terzi, cookie, testi SEO (title, description, og). Mostra la mappa all'utente prima di costruire.
2. **Conserva il design:** stessa identita', testi fedeli. Non riscrivere i contenuti se non richiesto.
3. **Tema:** header e footer in componenti, le ripetizioni in frammenti `k-array`, i valori ricorrenti in variabili CSS.
4. **Contenuti:** testi e immagini nei dati; upload in `assets/uploads/` con nomi leggibili; embed nei campi `embed`; nessun link verso il vecchio dominio o file locali.
5. **Cosa Kris non fa da solo:** form, invio email, prenotazioni, pagamenti e ricerca server. Controlla prima i plugin (`list_plugins` con `catalog: true`); se non ci sono, elencali e chiedi come gestirli: non fingere che funzionino.
6. **Redirect:** ogni vecchio URL con traffico ottiene un 301 con `set_redirect` (o in `schema.redirects` in locale), anche con `/*` per intere sezioni (`/project-detail/*`). Controlla che il vecchio `.htaccess` non abbia regole verso `?page=`.
7. **SEO:** ogni pagina in `schema.pages` con `"sitemap": true` dove serve; `base_url` e proprieta' del sito compilati dall'admin. Il `robots.txt` lo genera Kris: un file fisico vince, e se c'e' deve escludere `/kris-admin/` (non `/editor/`) e citare la sitemap. Non bloccare `/kris/` o `/assets/`.
8. **Cookie:** script di terzi bloccati con `type="text/plain"` e `data-kris-consent-script`, banner collegato a `kris_legal`; gli embed caricano contenuti di terzi: vanno nella cookie policy.

## 8. Verifica prima di consegnare

Guarda le pagine renderizzate con un browser, non il codice.

- Larghezze 375, 768, 1280 e 1440: testi lunghi, titoli con font larghi, nessuno sforamento dalla colonna.
- Tutte le lingue (il selettore resta sulla stessa pagina), testi vuoti, liste vuote, immagini mancanti.
- Header che resta visibile allo scroll e logo che riporta in cima; menu, tastiera, focus.
- Console senza errori, nessun 404 su asset e font (scheda rete).
- 404 con stato 404, redirect 301, `sitemap.php` con gli indirizzi leggibili, canonical, title, description e og su ogni pagina (`render_page`).
- PageSpeed da mobile su una pagina con molte immagini: dimensioni esplicite e foto non piu' grandi dell'uso.
- Editor: modifica un testo, un'immagine e l'ordine di una lista e controlla il sito. Ogni campo in sezione, con descrizione attuale, nessun campo inutilizzato.
- Controlli rapidi:

```sh
grep -nP '[^\x00-\x7F]' assets/js/*.js
grep -rnE 'https?://|wa\.me|tel:|onclick' template/
grep -n 'url(' assets/css/*.css
node --check assets/js/*.js
```

## 9. Deploy

**Primo rilascio via FTP:** lo zip lo prepara `php tools/package.php --first` (tema, dati iniziali, upload, `.htaccess`). `data/` e `assets/uploads/` devono essere scrivibili da PHP. Non lasciare il setup aperto: crea subito l'account. Controlla che `config/` e `data/` non siano scaricabili.

**Rilasci successivi:** via MCP. Kris si aggiorna dall'editor (Impostazioni > Versione di Kris), che segnala anche i problemi del sito (regole del `.htaccess`, `robots.txt`, embed nei testi). Mai `data/`, `config/` o upload dall'ambiente locale.

**Consegna:** con l'MCP dai i link di anteprima (`preview_pages`), ricorda che vanno aperti da admin con accesso all'editor, che gli script vanno approvati, che i plugin installati vanno accesi e che si pubblica da Editor > Da pubblicare. Dichiara sempre cosa hai verificato, cosa no, e l'elenco delle azioni manuali.
