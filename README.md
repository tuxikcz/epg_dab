# DAB EPG Lab – automatický program vysílání pro DAB+

**DAB EPG Lab** je nástroj, který převádí program rozhlasové stanice z webových stránek na **EPG** (elektronického programového průvodce) vysílaného přímo v DAB+ multiplexu.

Díky tomu si posluchač může na kompatibilním přijímači zobrazit, co právě hraje, co bude následovat a případně i popis pořadu. **Připojení přijímače k internetu není potřeba** – informace cestují přímo v DAB+ vysílání.

Projekt vznikl pro **HEY Radio** v multiplexu **COLOR DAB+**, ale jeho jednotlivé části lze postupně rozšířit i pro další rádia.

> **Stav k 17. listopadu 2026:** Vysílání EPG ze statického souboru bylo úspěšně vyzkoušeno na skutečném COLOR DAB+ a zobrazilo se v AbracaDABře. Automatické stahování a generování už funguje na serveru s Debianem 11 v odděleném **staging režimu**. Přepnutí produkčního multiplexeru na automatický FIFO vstup zatím **neproběhlo**.

## Jak to funguje

Celý postup si lze představit jako jednoduchou výrobní linku:

```text
Web HEY Radia (týdenní program a stránky pořadů)
                    |
                    v
       Stažení a zpracování HTML
                    |
                    v
     Program s konkrétními daty a časy
                    |
                    v
  Doplnění mezer + názvy, popisy, moderátoři
                    |
                    v
       Kontrola úplnosti a správnosti
                    |
                    v
       SPI / MOT / DAB datové pakety
                    |
                    v
                 epg.dat
                    |
            +-------+-------+
            |               |
            v               v
     Staging (test)    FIFO feeder (plán)
                            |
                            v
                       ODR-DabMux
                            |
                            v
                    COLOR DAB+ -> rádio
```

### 1. Stažení programu z webu

Program se načítá především ze stránky [týdenního programu HEY Radia](https://heyradio.cz/showpage.php?name=tyden). Aplikace navíc stahuje dostupné stránky jednotlivých pořadů a pokouší se k nim doplnit popisy a moderátory.

Týdenní tabulka je **pravidelné schéma**, nikoliv záruka, že například o státním svátku nepoběží mimořádný pořad. Takové změny zatím automaticky neověřujeme. Originální stažené stránky se ukládají do archivu pro případnou kontrolu.

### 2. Doplnění chybějících hodin

V rozpisu nemusí být vyplněná každá hodina. Aby v EPG nevznikaly prázdné úseky, mezery se automaticky doplní:

- **Pořad:** `Ten nejlepsi Rock na Hey radiu`
- **Moderátor:** `Němý DJ`

Existující pořady se mají zachovat. Generátor zároveň kontroluje časové mezery, překryvy a správné datum. Pracuje s časovým pásmem **Europe/Prague**, včetně přechodů mezi letním a zimním časem.

### 3. Názvy, popisy a moderátoři

Projekt si udržuje **katalog pořadů**, aby se už jednou získané anotace nemusely pokaždé hledat znovu. Párování je opatrné: používá název pořadu a odpovídající úplný odkaz, aby nepřiřadil špatný popis.

Součástí projektu je i katalog starších anotací z původního XMLTV. Tyto údaje jsou označené jako **historické a dnes neověřené**. Pokud není shoda jistá, popis se raději vynechá, než aby se v EPG zobrazila nepřesná informace.

### 4. Výroba souboru `epg.dat`

Zkontrolovaný program se převede do formátu **SPI** (informace o službě a pořadech), zabalí se do **MOT** objektů a následně do DAB datových paketů. Výsledkem je binární soubor **`epg.dat`**, který dokáže zpracovat ODR-DabMux.

Před publikováním se výsledek znovu kontroluje: programy, časy, identifikátory stanice a integrita dat. Pokud stahování nebo generování selže, nová neplatná verze nemá nahradit poslední funkční soubor.

### 5. Jak se EPG dostane do éteru

EPG využívá **samostatný datový subkanál** DAB+. Nejde o slideshow obrázků skladeb, které rádio dostává přes PAD v audio streamu. Audio HEY Radia tedy kvůli EPG nemusíme měnit.

V současnosti bylo v COLOR DAB+ úspěšně ověřeno vysílání **ze statického souboru**. Připravený automatický režim používá **FIFO** – speciální soubor, do kterého samostatný proces (*feeder*) opakovaně posílá EPG. Při aktualizaci dokončí celý cyklus a pak začne posílat novou verzi.

**Důležité:** první přechod ze statického souboru na FIFO vyžaduje změnu vstupu v konfiguraci a u používaného **ODR-DabMuxu 4.0.0** také řízené znovuotevření vstupu, typicky restart multiplexeru. To může krátce přerušit všechny služby. Proto tato změna ještě nebyla nasazena automaticky.

## Konkrétní nastavení COLOR DAB+

| Položka | Hodnota |
|---|---|
| Multiplex | COLOR DAB+ |
| DAB kanál | 6A |
| Ensemble ID (EID) | `0x2117` |
| ECC | `0xE2` |
| Stanice | HEY RADIO |
| Service ID (SID) | `0x24F8` |
| SCIdS | `0` |
| SPI bearer | `dab:2e2.2117.24f8.0` |
| Datový subkanál | `30` |
| Packet address | `1` |
| Datový tok EPG | `16 kbps`, EEP 3-A |
| ODR-DabMux | `4.0.0` |

EPG v multiplexu zabírá **12 CU**. AUDIO služby i EDI výstupy zůstávají oddělené od jeho obsahu.

## Co už bylo ověřeno

- Binární SPI/MOT generátor vytvořil EPG, které AbracaDABra skutečně zobrazila z vysílání COLOR DAB+ (názvy, časy, popisy a češtinu).
- Automatický import a čtyřdenní generování proběhly úspěšně na skutečném serveru s **Debianem 11**.
- Staging vytvořil 17. 11. 2026 **62 programových bloků**, z toho **10 výplňových**, v pěti MOT objektech. Výsledný soubor měl **52 032 bajtů**.
- Integrační a regresní testy proběhly s Pythonem **3.9, 3.11 a 3.13**; FIFO bylo testováno s izolovanou instancí ODR-DabMuxu 4.0.0.
- Samostatný systemd staging timer je aktivní a ruční spuštění jeho služby na produkčním serveru dokončilo generování úspěšně.

**Co ještě není ověřeno v produkčním vysílání:** nová FIFO cesta, dlouhodobý provoz feederu, skutečné automatické přepínání carouselu v éteru a příjem přes tuto cestu. Fungující statické EPG a fungující staging jsou dvě rozdílné věci.

## Instalace a vyzkoušení na Debianu

Projekt je určený zejména pro Debian 11 (Python 3.9), ale testuje se i na Debianu 12 a 13. Nepotřebuje měnit systémový Python ani ODR-DabMux.

### Instalace programu

Ze složky se zdrojovým kódem:

```bash
cd /home/tuxik/epg_dab
sh scripts/install-debian.sh --apply
```

Instalátor vytvoří samostatné prostředí v `/opt/dab-epg/.venv`. **Nespouští služby, neupravuje mux.conf a nepřepisuje vysílaný EPG soubor.** Pokud už v tomto umístění existuje virtuální prostředí, odmítne ho přepsat.

### Bezpečný staging test

```bash
install -d -o tuxik -g tuxik -m 0750 /home/tuxik/dab-epg-stage-state

sudo -u tuxik /opt/dab-epg/.venv/bin/dab-epg-stage \
  --state-dir /home/tuxik/dab-epg-stage-state

/opt/dab-epg/.venv/bin/dab-epg-status \
  --state-dir /home/tuxik/dab-epg-stage-state
```

Staging zapisuje pouze do určeného testovacího adresáře. **Není propojený s produkčním vstupem muxu.** Program umí volitelně porovnat svůj výstup s explicitně zadaným statickým EPG pomocí `--compare /cesta/k/testovacimu/epg.dat` (referenční soubor se jen čte).

Výsledky najdeš například zde:

```text
/home/tuxik/dab-epg-stage-state/
├── staging.json              # Shrnutí generování a případného porovnání
├── staging.log               # Historie pokusů o generování
├── update.json               # Stav importu a vytvořených dat
├── metadata-resolution.json  # Odkud se vzaly popisy pořadů
└── ...                       # Stažené podklady, katalog a archiv
```

### Automatický staging přes systemd

Na serveru COLOR DAB+ je pro testovací generování připravená služba `dab-epg-stage-auto.service` a timer `dab-epg-stage-auto.timer`.

Rychlá kontrola:

```bash
systemctl status dab-epg-stage-auto.service --no-pager
systemctl list-timers --all dab-epg-stage-auto.timer
journalctl -u dab-epg-stage-auto.service -n 30 --no-pager
```

Ruční spuštění mimo plánovaný čas:

```bash
systemctl start dab-epg-stage-auto.service
```

Služba typu `oneshot` po úspěšném dokončení běžně ukazuje `inactive (dead)` a `status=0/SUCCESS`. **To je v pořádku.** Timer se spouští přibližně každé čtyři hodiny (s náhodným zpožděním až 15 minut) a má zapnutý `Persistent=true`.

Automatický staging **jen připravuje soubory** – nic sám neposílá do vysílání.

## Produkční automatizace: co je připraveno a co zatím není zapnuté

V projektu jsou připravené následující jednotky:

| Soubor | Úkol |
|---|---|
| `deploy/systemd/dab-epg-update.service` | Stáhne program, ověří jej a připraví nový binární soubor |
| `deploy/systemd/dab-epg-update.timer` | Pravidelně spouští aktualizaci |
| `deploy/systemd/dab-epg-feeder.service` | Opakuje celý carousel do FIFO pro multiplexer |

Jejich výchozí cesta `/var/lib/dab-epg/epg.dat` je stejná jako cesta používaná současným **statickým EPG**. Proto **tyto produkční jednotky nezapínej naslepo**, dokud není připravené a schválené přepnutí. Samotný `scripts/install-debian.sh` je pouze instalace programu, ne aktivace vysílání.

Při budoucím přechodu na FIFO je nutné předem ověřit přístupová práva, stav feederu, kompletní zálohu konfigurace a aktuální obsah EPG. Přesný postup je popsán v `docs/deployment-color-checklist.md`.

### Návrat při problému

Návrat z FIFO na statický vstup znamená obnovit **zálohovanou konfiguraci i platný statický soubor** a znovu otevřít vstup muxu. Může to vyžadovat restart a krátký výpadek. Skript `scripts/rollback-carousel.sh` pouze vrací předchozí validovanou verzi carouselu ve feederu – **není to návrat celé konfigurace muxu na statický vstup**.

## Jak poznat, že něco nefunguje

| Projev | Co zkontrolovat |
|---|---|
| EPG se neaktualizuje | `staging.log`, `update.json`, poslední úspěšné stažení webu |
| Generování selže | Dostupnost webu, změnu HTML stránky, pokrytí programu, validaci |
| Chybí popisy pořadů | `metadata-resolution.json`; popis mohl být záměrně vynechán kvůli nejisté shodě |
| `feeder: unknown_or_stale` ve stagingu | Běžné, protože staging nespouští produkční feeder |
| EPG není v AbracaDABře | Ověřit správný `epg.dat`, EID `0x2117`, SID `0x24F8`, cache přijímače a přenos SPI/MOT |
| Mux ukazuje starý program | Ověřit SHA256 souboru i to, který otevřený soubor mux ve skutečnosti čte |
| Timer běží, ale program je starý | Zkontrolovat `coverage_start`, `coverage_end_exclusive` a zda web poskytuje stále stejnou týdenní šablonu |

U současné verze ODR-DabMuxu je důležité vědět, že při atomickém nahrazení souboru může proces dále číst původní otevřený soubor. Právě proto je připravovaný FIFO feeder.

## Co projekt zatím neumí nebo negarantuje

- Týdenní HTML samo o sobě **nepotvrzuje mimořádné změny denního programu**.
- Starší katalogové popisy nemusí odpovídat dnešnímu vysílání.
- Připravený generátor pracuje aktuálně s **jednou stanicí**; více stanic je další možný krok.
- RadioDNS přes internet není pro tento projekt potřeba a zatím není předmětem automatizace.
- Úspěšný zápis do FIFO neznamená, že každý přijímač již sestavil kompletní EPG.
- Webová administrační stránka je plánovaná, ale zatím není součástí hotového řešení.

## Kde hledat podrobnosti

Pokud chceš technické detaily, postupy a výsledky testování, najdeš je v repozitáři:

- `docs/phase5.md` – automatický import a FIFO feeder.
- `docs/phase51-staging.md` – katalog pořadů a testování na Debianu 11.
- `docs/deployment-color-checklist.md` – bezpečné přepnutí multiplexu a návrat.
- `CURRENT_HANDOFF.md`, `CURRENT_TODO.md`, `CHANGELOG.md` – aktuální stav a historie projektu.

---

**Stručně:** DAB EPG Lab si umí sám přečíst program z webu, doplnit mezery, připravit čtyřdenní EPG a zkontrolovat jeho správnost. Statické vysílání EPG už fungovalo v COLOR DAB+. Nyní běží automatické generování v bezpečném stagingu; posledním krokem k plné automatizaci je schválené přepnutí produkčního muxu na FIFO.
