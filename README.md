# Immo_Webscraper: Ein automatisierter Webscraper für ImmobilienScout24 unter Nutzung von Pythons Playwright
Schreibt einen auf Pythons Playwright basierenden Bot, um Daten zu zum Verkauf stehenden Häusern von [ImmobilienScout24](https://www.immobilienscout24.de/) zu sammeln. Es sei darauf hingewiesen, dass der Code nur zur Veranschaulichung dient und nicht aktiv genutzt werden sollte, da der Webseiten-Betreiber Richtlinien gegen Webscraping haben könnte.

## Grundlage
Wir schreiben Code, um Pythons Paket `playwright` zu nutzen, um einen Chrome-Browser manuell durch die Suchliste von ImmobilienScout24 zu navigieren, Seiten zu Immobilien nacheinander zu öffnen und die Verkaufsdaten in einem `pandas` dataframe zu speichern. Als Beispiel nutzen wir hier die Suchergebnisse der zum Verkauf stehenden Häuser im Kreis Paderborn in Nordrhein-Westfalen (sortiert von den neuesten Einträgen zu den ältesten), jedoch kann der Code theoretisch für andere Suchen genutzt werden.

## Hinweise vorweg
ImmobilienScout24 nutzt Anti-Bot-Software, um botgesteuerte Browser zu erkennen und Scraping-Versuche entweder zu verlangsamen oder auch mit CAPTCHAs zu unterbrechen. Aus diesem Grund nutzen wir einen durch `seleniumbase` gestarteten Stealth-Browser, den wir dann mit `playwright` steuern. Dadurch tauchen keine reCAPTCHAs bei Ansteuern der Seite auf. Es ist anzumerken, dass manche Werbseiten in regelmäßigen Abständen HTML-Tags verändern, um dadurch zusätzlich Webscraping vorzubeugen. Daher ist der folgende Code nur eine Momentaufnahme und muss ggf. von Zeit zu Zeit angepasst werden.

## Code

### Imports

```
from playwright.sync_api import sync_playwright
from seleniumbase import sb_cdp
from datetime import datetime 
import pandas as pd
from numpy.random import uniform as unif
from playwright.sync_api import TimeoutError as PlaywrightTimeoutError
```

### Hilfsfunktionen

Dies sind Hilfsfunktionen, um die einzelnen Größen von der Immobilienwebseite abzufangen und zu speichern. Globale Listen werden anhand dieser aktualisiert. Eine weitere Hiulfsfunktion startet den Browser neu, was hilfreich sein kann, um Timeouts zu vermeiden.

```
# Kaufpreis gesamt
def sammle_kaufpreis(page, inp_list):
    if page.locator('xpath = //span[contains(@class, "preis-value")]').count() > 0:
        inp_list.append(page.locator('xpath = //span[contains(@class, "preis-value")]').inner_text())
    else:
        inp_list.append('nan')

def sammle_adresse(page, inp_list):
    sub = page.locator('xpath=//div[contains(@data-qa, "address")]//span')
    if sub.count() > 0:
        inp_list.append(sub.first.inner_text())
    else:
        inp_list.append('nan')

# Kaufpreis pro Quadratmeter    
def sammle_preis_pro_qm(page, inp_list):
    sub = page.locator('xpath=//div[@class = "is24qa-maincriteria-purchaseprice-label-main-label is24-label font-s"]//span')
    if sub.count() > 0:
        inp_list.append(sub.inner_text())
    else:
        inp_list.append('nan')
# Wohnflaeche in Quadratmetern
def sammle_wohn_qm(page, inp_list):
    sub = page.locator('xpath=//div[contains(@class, "maincriteria-livingspace-label-main")]')
    if sub.count() > 0:
        inp_list.append(sub.first.inner_text())
    else:
        inp_list.append('nan')
# Grundstuecksflaeche in Quadratmetern
def sammle_grundst_qm(page, inp_list):
    sub = page.locator('xpath=//div[contains(@class, "maincriteria-plotareashort-label-main")]')
    if sub.count() > 0:
        inp_list.append(sub.first.inner_text())
    else:
        inp_list.append('nan')
# Keller vorhanden?
def sammle_keller_existiert(page, inp_list):
    if page.locator('xpath=//span[contains(@class, "cellar-label")]').count() > 0:
        keller_existiert = 'ja'
    else:
        keller_existiert = 'nein'
    inp_list.append(keller_existiert)
# Anzahl Zimmer
def sammle_anzahl_zimmer(page, inp_list):
    sub = page.locator('xpath=//div[contains(@class, "maincriteria-numberofrooms-label-main")]')
    if sub.count() > 0:
        inp_list.append(sub.first.inner_text())
    else:
        inp_list.append('nan')
# Anzahl der Baeder
def sammle_anzahl_bad(page, inp_list):
    sub = page.locator('xpath=//dd[contains(@class, "badezimmer")]')
    if sub.count() > 0:
        inp_list.append(sub.inner_text())
    else:
        inp_list.append('nan')
def sammle_anzahl_etagen(page, inp_list):
    sub = page.locator('xpath=//dd[contains(@class, "is24qa-etagenanzahl grid-item")]')
    if sub.count() > 0:
        inp_list.append(sub.inner_text())
    else:
        inp_list.append('nan')
def sammle_anzahl_schlafzimmer(page, inp_list):
    sub = page.locator('xpath=//dd[contains(@class, "is24qa-schlafzimmer grid-item")]')
    if sub.count() > 0:
        inp_list.append(sub.inner_text())
    else:
        inp_list.append('nan')
# Anzahl der Garagen, Stellplaetze, etc.
def sammle_anzahl_garage(page, inp_list):
    sub = page.locator('xpath=//dd[contains(@class, "garage-stellplatz")]')
    if sub.count() > 0:
        inp_list.append(sub.first.inner_text())
    else:
        inp_list.append('nan')
# Haustyp
def sammle_haustyp(page, inp_list):
    sub = page.locator('xpath=//dd[contains(@class, "is24qa-typ grid-item")]')
    if sub.count() > 0:
        inp_list.append(sub.inner_text())
    else:
        inp_list.append('nan')
# Marklerprovision
def sammle_maklerprovision(page, inp_list):
    sub = page.locator('xpath=//dd[contains(@class, "is24qa-provision grid-item")]')
    if sub.count() > 0:
        inp_list.append(sub.inner_text())
    else:
        inp_list.append('nan')
# Baujahr des Hauses
def sammle_baujahr(page, inp_list):
    sub = page.locator('xpath=//dd[contains(@class, "is24qa-baujahr grid-item")]')
    if sub.count() > 0:
        inp_list.append(sub.inner_text())
    else:
        inp_list.append('nan')
# Zustand des Hauses
def sammle_objektzustand(page, inp_list):
    sub = page.locator('xpath=//dd[contains(@class, "is24qa-objektzustand grid-item")]')
    if sub.count() > 0:
        inp_list.append(sub.inner_text())
    else:
        inp_list.append('nan')
# Qualitaet der Ausstattung
def sammle_quali_ausstattung(page, inp_list):
    sub = page.locator('xpath=//dd[contains(@class, "is24qa-qualitaet-der-ausstattung grid-item")]')
    if sub.count() > 0:
        inp_list.append(sub.inner_text())
    else:
        inp_list.append('nan')
# Heizungsart
def sammle_heizungsart(page, inp_list):
    sub = page.locator('xpath=//dd[contains(@class, "is24qa-heizungsart grid-item")]')
    if sub.count() > 0:
        inp_list.append(sub.inner_text())
    else:
        inp_list.append('nan')
# Wesentliche Energietraeger
def sammle_energietraeger(page, inp_list):
    sub = page.locator('xpath=//dd[contains(@class, "is24qa-wesentliche-energietraeger grid-item")]')
    if sub.count() > 0:
        inp_list.append(sub.inner_text())
    else:
        inp_list.append('nan')
# Liegt Energieausweis vor?
def sammle_energieausweis_existiert(page, inp_list):
    sub = page.locator('xpath=//dd[contains(@class, "is24qa-energieausweis grid-item")]')
    if sub.count() > 0:
        inp_list.append('ja') if sub.inner_text() == 'liegt vor' else inp_list.append('nein')
    else:
        inp_list.append('nan')
# Typ des Energieausweises
def sammle_energieausweistyp(page, inp_list):
    sub = page.locator('xpath=//dd[contains(@class, "is24qa-energieausweistyp grid-item")]')
    if sub.count() > 0:
        inp_list.append(sub.inner_text())
    else:
        inp_list.append('nan')
# Endenergiebedarf
def sammle_endenergiebedarf(page, inp_list):
    sub = page.locator('xpath=//dd[contains(@class, "is24qa-endenergiebedarf grid-item")]')
    if sub.count() > 0:
        inp_list.append(sub.inner_text())
    else:
        inp_list.append('nan')
# Energieklasse
def sammle_energieklasse(page, inp_list):
    sub = page.locator('xpath=//span[@class = "energy-efficiency-class"]//img')
    if sub.count() > 0:
        inp_list.append(sub.get_attribute('src').split('/')[-1:][0].split('.')[0])
    else:
        inp_list.append('nan')
# Beschreibender Text zur Ausstattung
def sammle_text_ausstattung(page, inp_list):
    block = page.locator("div.expose-description-block").filter(has = page.locator("span.expose-description-h3", has_text = "Ausstattung"))
    if block.count() > 0:
        inp_list.append(block.first.locator('xpath=//span[@class="expose-description-body"]').inner_text())
    else:
        inp_list.append('nan')
# Beschreibender Text zur Lage des Hauses
def sammle_text_lage(page, inp_list):
    block = page.locator("span.expose-location-description")
    if block.count() > 0:
        inp_list.append(block.inner_text())
    else:
        inp_list.append('nan')
# Allgemeine Objektbeschreibung
def sammle_text_objektbeschreibung(page, inp_list):
    block = page.locator("div.expose-description-block").filter(has = page.locator("span.expose-description-h3", has_text = "Objektbeschreibung"))
    if block.count() > 0:
        inp_list.append(block.first.locator('xpath=//span[@class="expose-description-body"]').inner_text())
    else:
        inp_list.append('nan')
# Exposé-Titel    
def sammle_expose_titel(page, inp_list):
    subs = page.locator('xpath=//h1[@data-qa = "expose-title"]')
    if subs.count() > 0:
        print(subs.inner_text())
        inp_list.append(subs.inner_text())
    else:
        print('nan')
        inp_list.append('nan')
# Veröffentlichungsdatum
def sammle_veroeffentlichungsdatum(page, inp_list):
    if page.locator('xpath=//div[contains(@class, "premium-stats-item grid-item")]').count() > 0:
        inp_list.append(page.locator('xpath=//div[contains(@class, "premium-stats-item grid-item")]').nth(1).inner_text())
    else:
        inp_list.append('nan')
# Sammle und aktualisiere alle Größen
def sammle_alle_groessen(page, titel, adresse, kaufpreis_gesamt, kaufpreis_pro_qm,
                         wohnflaeche_in_qm, grundstuecksflaeche_in_qm,
                         keller_existiert, anzahl_zimmer, anzahl_badezimmer,
                         anzahl_schlafzimmer, anzahl_etagen,
                         autostellplaetze, haustyp, maklerprovision,
                         baujahr, objektzustand, ausstattungsquali,
                         heizungsart, energietraeger,
                         energieausweis_vorliegend, typ_energieausweis,
                         endenergiebedarf, energieklasse,
                         text_objektbeschreibung, text_ausstattung,
                         text_lage, veroeffentlichungsdatum,
                         download_zeit):
    sammle_expose_titel(page, titel)
    sammle_adresse(page, adresse)
    sammle_kaufpreis(page, kaufpreis_gesamt)
    sammle_preis_pro_qm(page, kaufpreis_pro_qm)
    sammle_wohn_qm(page, wohnflaeche_in_qm)
    sammle_grundst_qm(page, grundstuecksflaeche_in_qm)
    sammle_keller_existiert(page, keller_existiert)
    sammle_anzahl_zimmer(page, anzahl_zimmer)
    sammle_anzahl_bad(page, anzahl_badezimmer)
    sammle_anzahl_schlafzimmer(page, anzahl_schlafzimmer)
    sammle_anzahl_etagen(page, anzahl_etagen)
    sammle_anzahl_garage(page, autostellplaetze)
    sammle_haustyp(page, haustyp)
    sammle_maklerprovision(page, maklerprovision)
    sammle_baujahr(page, baujahr)
    sammle_objektzustand(page, objektzustand)
    sammle_quali_ausstattung(page, ausstattungsquali)
    sammle_heizungsart(page, heizungsart)
    sammle_energietraeger(page, energietraeger)
    sammle_energieausweis_existiert(page, energieausweis_vorliegend)
    sammle_energieausweistyp(page, typ_energieausweis)
    sammle_endenergiebedarf(page, endenergiebedarf)
    sammle_energieklasse(page, energieklasse)
    sammle_text_objektbeschreibung(page, text_objektbeschreibung)
    sammle_text_ausstattung(page, text_ausstattung)
    sammle_text_lage(page, text_lage)
    sammle_veroeffentlichungsdatum(page, veroeffentlichungsdatum)
    download_zeit.append(datetime.now().strftime("%y-%m-%d %H:%M"))

# Sammle auf Suchseite die URLs der einzelnen Objekte
def neubau_ind(page_card):
    if page_card.locator('xpath=.//div[@class="indicator indicator--brand-white indicator--elevation"]//span[text() = "Haus bauen"]').count() > 0:
        return True
    else:
        return False
# Sammle Neubauprojekt-Indikator und gegenteiligen Indikator
def sammle_SuchURLs_Einzelobjekt(page, url_sammlung, neubau_sammlung):
    page_results = page.locator("xpath=//div[@data-elementtype = 'hybridViewCardContainer']")
    page_cards = page_results.locator("xpath=.//div[contains(@class, 'listing-card card-listing-') and not(contains(@class, 'stacked-cards'))]")
    neubau_idx = [neubau_ind(pc) for pc in page_cards.all()]    
    url_new = [card.locator('xpath=.//a[@data-exp-referrer="HYBRID_VIEW_LISTING"]').first.get_attribute('href') for card in page_cards.all()]
    url_sammlung.append(url_new)
    neubau_neu = ['kein hausbau'] * len(url_new)
    for pos in range(0, len(url_new)):
        if neubau_idx[pos]:
            neubau_neu[pos] = 'hausbau'
    neubau_sammlung.append(neubau_neu)
    
def sammle_SuchURLs_Mehrprojekte(page, url_sammlung, neubau_sammlung):
    page_results = page.locator("xpath=//div[@data-elementtype = 'hybridViewCardContainer']")
    page_cards = page_results.locator("xpath=.//div[contains(@class, 'listing-card card-listing-') and contains(@class, 'stacked-cards')]") 
    sub_cards = page_cards.locator("xpath=.//div[@class='slick-list']//div[contains(@class, 'slick-slide')]//a[@data-exp-referrer='HYBRID_VIEW_LISTING']").all()
    url_new = [card.first.get_attribute('href') for card in sub_cards]
    url_sammlung.append(url_new)
    neubau_sammlung.append(['hausbau'] * len(url_new))

# Sammle die gesammelten Such-URLs
def sammle_SuchURLs(page, url_sammlung, neubau_sammlung):
    sammle_SuchURLs_Einzelobjekt(page, url_sammlung, neubau_sammlung)
    sammle_SuchURLs_Mehrprojekte(page, url_sammlung, neubau_sammlung)

# Starte Browser neu
def neustart_browser(page, base_url, pw, sb):
    page.close()
    sb.sleep(60)
    # Restart browser to avoid timeout
    sb = sb_cdp.Chrome()
    endpoint_url = sb.get_endpoint_url()
    browser = pw.chromium.connect_over_cdp(endpoint_url)
    context = browser.contexts[0]
    context.set_default_timeout(30000)
    page = context.pages[0]
    page.goto(base_url, timeout=30000, wait_until="domcontentloaded")
    sb.sleep(5)
    page.get_by_test_id("pagination-button-next").click()
    sb.sleep(2)
    cookie_button = page.get_by_role("button", name="Alle akzeptieren")
    if cookie_button.count() > 0:
        cookie_button.click()
        sb.sleep(2)
    return page
```

### Browser initialisieren & Startseite laden

```
sb = sb_cdp.Chrome()
endpoint_url = sb.get_endpoint_url()
# Aktualisiere den Dateipfad
file_path = 'C:/Users/UserName/Documents/Immobiliendaten/Datensatz_ImmoScout24_KreisPaderborn20260801'
base_url = 'https://www.immobilienscout24.de/Suche/de/nordrhein-westfalen/paderborn-kreis/haus-kaufen?enteredFrom=one_step_search'
```

Durch diesen Ansatz wird kein reCAPCTHA zu lösen sein.

### Leere Listen

```
# Leere Listen, die zu füllen sind
URLS = []
neubau = []
titel = []
adresse = []
kaufpreis_gesamt = []
kaufpreis_pro_qm = []
wohnflaeche_in_qm = []
grundstuecksflaeche_in_qm = []
keller_existiert = []
anzahl_zimmer = []
anzahl_badezimmer = []
anzahl_schlafzimmer = []
anzahl_etagen = []
autostellplaetze = []
haustyp = []
maklerprovision = []
baujahr = []
objektzustand = []
ausstattungsquali = []
heizungsart = []
energietraeger = []
energieausweis_vorliegend = []
typ_energieausweis = []
endenergiebedarf = []
energieklasse = []
text_objektbeschreibung = []
text_ausstattung = []
text_lage = []
veroeffentlichungsdatum = []
download_zeit = []
```

### Herunterladen der Daten

```
i = 0
with sync_playwright() as p:
    # Starte Browser
    browser = p.chromium.connect_over_cdp(endpoint_url)
    context = browser.contexts[0]
    context.set_default_timeout(30000)
    # Lade Seite
    page = context.pages[0]
    page.goto(base_url, timeout=30000, wait_until="domcontentloaded")
    sb.sleep(5)
    page.get_by_test_id("pagination-button-next").click()
    sb.sleep(2)
    # Klicke den Cookie-Button
    cookie_button = page.get_by_role("button", name="Alle akzeptieren")
    if cookie_button.count() > 0:
        cookie_button.click()  # Cookie-Fenster schließen
    sb.sleep(1)
    page.get_by_test_id("pagination-button-prev").click()
    sb.sleep(1)
    # Sammle URLs und Infos zu Neubau
    while i == 0:
        sammle_SuchURLs(page, URLS, neubau)    # Erweitert URLS-Objekt
        next_button_disab = page.locator("xpath=//button[@data-testid='pagination-button-next']").get_attribute("aria-disabled")
        if next_button_disab == "true":
            print('URLs erfolgreich gesammelt')
            break      
        page.get_by_test_id("pagination-button-next").click()
        #sb.sleep(1 + unif(1, 2, 1).item())
        sb.sleep(1)
        
    # Passe URLs an, behalte nur einzigartige URLs, und entferne auch entsprechend
    # Hausbau-Einträge bei URL-Dopplungen    
    URLS = ['https://www.immobilienscout24.de' + url for sublist in URLS for url in sublist]
    neubau = [item for sublist in neubau for item in sublist]

    URLS_final = list()
    neubau_final = list()

    for s_url, s_nb in zip(URLS, neubau):
        if s_url not in URLS_final:
            URLS_final.append(s_url)
            neubau_final.append(s_nb)
    
    try:   # Starte Browser neu
        page = neustart_browser(page = page, base_url = base_url, pw = p, sb = sb)
    except:
        pass
    
    
    # Gehe durch alle gespeicherten URLs und sammle Daten
    j = 0
    k = 1
    
    for url in URLS_final:
        j += 1
        success = False
        retry_count = 0

        # Mit Failback-While-Schleife, um erneut zu versuchen
        while not success and retry_count < 3:
            # Versuche, zu Seite zu gehen und Daten zu sammeln
            try:
                page.goto(url, timeout=30000, wait_until="domcontentloaded")
                cookie_button = page.get_by_role("button", name="Alle akzeptieren")
                if cookie_button.count() > 0:
                    cookie_button.click()
                print(j)
                sb.sleep(1.5)
                #sb.sleep(1.5 + unif(0, 1, 1).item())   # Anpassen, z.B. bei langsamerer Internetverbindung
                #zusatz_warten = unif(0, 1, 1).item()
                #if (zusatz_warten < 0.05):
                #    sb.sleep(3)
                sammle_alle_groessen(page, titel, adresse, kaufpreis_gesamt, kaufpreis_pro_qm,
                                 wohnflaeche_in_qm, grundstuecksflaeche_in_qm,
                                 keller_existiert, anzahl_zimmer, anzahl_badezimmer,
                                 anzahl_schlafzimmer, anzahl_etagen,
                                 autostellplaetze, haustyp, maklerprovision,
                                 baujahr, objektzustand, ausstattungsquali,
                                 heizungsart, energietraeger,
                                 energieausweis_vorliegend, typ_energieausweis,
                                 endenergiebedarf, energieklasse,
                                 text_objektbeschreibung, text_ausstattung,
                                 text_lage, veroeffentlichungsdatum,
                                 download_zeit)
                success = True
            
            except PlaywrightTimeoutError:   # Bei Timeout, bersuche erneut
                retry_count += 1
                
                try:   # Lade dafür Browser neu zur Sicherheit
                    page = neustart_browser(page = page, base_url = base_url, pw = p, sb = sb)
                except:
                    pass
                next
            
            if j % 300 == 0:    # Nach 300 Datenpunkten, speichere bisherige Daten und starte Browser neu

                df = pd.DataFrame({
                    'Titel': titel,
                    'Adresse': adresse,
                    'Kaufpreis_Gesamt': kaufpreis_gesamt,
                    'Kaufpreis_pro_QM': kaufpreis_pro_qm,
                    'Wohnflaeche_in_QM': wohnflaeche_in_qm,
                    'Grundstuecksflaeche_in_QM': grundstuecksflaeche_in_qm,
                    'Keller_existiert': keller_existiert,
                    'Anzahl_Zimmer': anzahl_zimmer,
                    'Anzahl_Badezimmer': anzahl_badezimmer,
                    'Anzahl_Schlafzimmer': anzahl_schlafzimmer,
                    'Anzahl_Etagen': anzahl_etagen,
                    'Anzahl_Autostellplaetze': autostellplaetze,
                    'Haustyp': haustyp,
                    'Maklerprovision': maklerprovision,
                    'Baujahr': baujahr,
                    'Objektzustand': objektzustand,
                    'Ausstattungsqualitaet': ausstattungsquali,
                    'Heizungsart': heizungsart,
                    'Wesentliche_Energietraeger': energietraeger,
                    'Energieausweis_vorliegend': energieausweis_vorliegend,
                    'Typ_Energieausweis': typ_energieausweis,
                    'Endenergiebedarf': endenergiebedarf,
                    'Energieklasse': energieklasse,
                    'Text_Objektbeschreibung': text_objektbeschreibung,
                    'Text_Ausstattung': text_ausstattung,
                    'Text_Lage': text_lage,            
                    'URL': URLS_final[0:j],
                    'Bauprojekt': neubau_final[0:j],
                    'Veroeffentlichungsdatum': veroeffentlichungsdatum,
                    'Download_Zeit': download_zeit
                })

                file_nam = file_path + 'Set' + str(k) + '.csv'
                df.to_csv(file_nam, sep = '|', index = False)
                print('Datensatz gespeichert bis Index ' + str(j) + '!')
                k += 1
                page = neustart_browser(page = page, base_url = base_url, pw = p, sb = sb)
    page.close()   # Schließe Browser final
    try:
        browser.close()
    except:
        pass
```

### Speichern des Datensatzes

```
df = pd.DataFrame({
        'Titel': titel,
        'Adresse': adresse,
        'Kaufpreis_Gesamt': kaufpreis_gesamt,
        'Kaufpreis_pro_QM': kaufpreis_pro_qm,
        'Wohnflaeche_in_QM': wohnflaeche_in_qm,
        'Grundstuecksflaeche_in_QM': grundstuecksflaeche_in_qm,
        'Keller_existiert': keller_existiert,
        'Anzahl_Zimmer': anzahl_zimmer,
        'Anzahl_Badezimmer': anzahl_badezimmer,
        'Anzahl_Schlafzimmer': anzahl_schlafzimmer,
        'Anzahl_Etagen': anzahl_etagen,
        'Anzahl_Autostellplaetze': autostellplaetze,
        'Haustyp': haustyp,
        'Maklerprovision': maklerprovision,
        'Baujahr': baujahr,
        'Objektzustand': objektzustand,
        'Ausstattungsqualitaet': ausstattungsquali,
        'Heizungsart': heizungsart,
        'Wesentliche_Energietraeger': energietraeger,
        'Energieausweis_vorliegend': energieausweis_vorliegend,
        'Typ_Energieausweis': typ_energieausweis,
        'Endenergiebedarf': endenergiebedarf,
        'Energieklasse': energieklasse,
        'Text_Objektbeschreibung': text_objektbeschreibung,
        'Text_Ausstattung': text_ausstattung,
        'Text_Lage': text_lage,            
        'URL': URLS_final,
        'Bauprojekt': neubau_final,
        'Veroeffentlichungsdatum': veroeffentlichungsdatum,
        'Download_Zeit': download_zeit
    })

file_nam = file_path + 'Total.csv'
df.to_csv(file_nam, sep = '|', index = False)
print('Vollständiger Datensatz gespeichert!')
```
