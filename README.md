# Immo_Webscraper: Ein automatisierter Webscraper für ImmobilienScout24 unter Nutzung von Pythons Selenium
Schreibt einen auf Pythons Selenium basierenden Bot, um Daten zu zum Verkauf stehenden Häusern von [ImmobilienScout24](https://www.immobilienscout24.de/) zu sammeln. Es sei darauf hingewiesen, dass der Code nur zur Veranschaulichung dient und nicht aktiv genutzt werden sollte, da der Webseiten-Betreiber Richtlinien gegen Webscraping haben könnte.

## Grundlage
Wir schreiben Code, um Pythons Paket `selenium` zu nutzen, um einen Firefox-Browser manuell durch die Suchliste von ImmobilienScout24 zu navigieren, Seiten zu Immobilien nacheinander zu öffnen und die Verkaufsdaten in einem `pandas` dataframe zu speichern. Als Beispiel nutzen wir hier die Suchergebnisse der zum Verkauf stehenden Häuser im Kreis Paderborn in Nordrhein-Westfalen (sortiert von den neuesten Einträgen zu den ältesten), jedoch kann der Code theoretisch für andere Suchen genutzt werden.

## Hinweise vorweg
ImmobilienScout24 nutzt Anti-Bot-Software, um botgesteuerte Browser zu erkennen und Scraping-Versuche entweder zu verlangsamen oder auch mit CAPTCHAs zu unterbrechen. Schon beim ersten Öffnen der Seite könnte man mit einem CAPTCHA konfrontiert werden. Daher nutzen wir hier `selenium`. Dies ermöglicht uns, mit dem ferngesteuerten Browser manuell interagieren zu können, sollten wir auf Hindernisse wie CAPTCHAs treffen. Dem aktuellen Stand nach vom 27.06.2026 könnte man beim ersten Öffnen der Seite auf ein CAPTCHA treffen sowie jeweils nach dem Scrapen von ca. 450 Einträgen. Auch Tools wie `undetected_chromedriver` und `selenium_stealth` helfen derzeit nicht mehr weiter.

## Code

### Imports

```
from selenium import webdriver               # zur Browser-Steuerung
from selenium.webdriver.common.by import By  # für XPATH
from selenium.webdriver.support.ui import WebDriverWait  # um den Browser auf das Laden von Elementen warten zu lassen
from selenium.webdriver.support import expected_conditions as EC   # um den Browser auf das Laden von Elementen warten zu lassen
import time                                  # für Wartezeiten
import pandas as pd                          # für Datensatzerstellung
from datetime import datetime                # um Zeitpunkt des Downloads zu speichern
from numpy.random import uniform as unif     # für variable Wartezeiten
```

### Hilfsfunktionen

```

# Kaufpreis gesamt
def sammle_kaufpreis(driver):
    try:
        return driver.find_element(By.XPATH, '//span[contains(@class, "preis-value")]').text
    except:
        return 'nan'

def sammle_adresse(driver):
    try:
        return driver.find_element(By.XPATH, '//div[@class="address-block"]//span[@class="zip-region-and-country"]').text
    except:
        return 'nan'

# Kaufpreis pro Quadratmeter    
def sammle_preis_pro_qm(driver):
    try:
        return driver.find_element(By.XPATH, '//div[@class = "is24qa-maincriteria-purchaseprice-label-main-label is24-label font-s"]//span').text
    except:
        return 'nan'
# Wohnflaeche in Quadratmetern
def sammle_wohn_qm(driver):
    try:
        return driver.find_element(By.XPATH, '//div[contains(@class, "maincriteria-livingspace-label-main")]').text
    except:
        return 'nan'
# Grundstuecksflaeche in Quadratmetern
def sammle_grundst_qm(driver):
    try:
        return driver.find_element(By.XPATH, '//div[contains(@class, "maincriteria-plotareashort-label-main")]').text
    except:
        return 'nan'
# Keller vorhanden?
def sammle_keller_existiert(driver):
    try:
        keller_existiert = driver.find_element(By.XPATH, '//span[contains(@class, "keller-label")]')
        keller_existiert = 'ja'
    except:
        keller_existiert = 'nein'
    return keller_existiert
# Anzahl Zimmer
def sammle_anzahl_zimmer(driver):
    try:
        return float(driver.find_element(By.XPATH, '//div[contains(@class, "maincriteria-numberofrooms-label-main")]').text)
    except:
        return float('nan')
# Anzahl der Baeder
def sammle_anzahl_bad(driver):
    try:
        return float(driver.find_element(By.XPATH, '//dd[contains(@class, "badezimmer")]').text)
    except:
        return float('nan')
def sammle_anzahl_etagen(driver):
    try:
        return float(driver.find_element(By.XPATH, '//dd[contains(@class, "is24qa-etagenanzahl grid-item")]').text)
    except:
        float('nan')
def sammle_anzahl_schlafzimmer(driver):
    try:
        return float(driver.find_element(By.XPATH, '//dd[contains(@class, "is24qa-schlafzimmer grid-item")]').text)
    except:
        float('nan')
# Anzahl der Garagen, Stellplaetze, etc.
def sammle_anzahl_garage(driver):
    try:
        return driver.find_element(By.XPATH, '//dd[contains(@class, "garage-stellplatz")]').text
    except:
        return 'nan'
# Haustyp
def sammle_haustyp(driver):
    try:
        return driver.find_element(By.XPATH, '//dd[contains(@class, "is24qa-typ grid-item")]').text
    except:
        return 'nan'
# Marklerprovision
def sammle_marklerprovision(driver):
    try:
        return driver.find_element(By.XPATH, '//dd[contains(@class, "is24qa-provision grid-item")]').text.split(' %')[0] + '%'
    except:
        return 'nan'
# Baujahr des Hauses
def sammle_baujahr(driver):
    try:
        return driver.find_element(By.XPATH, '//dd[contains(@class, "is24qa-baujahr grid-item")]').text
    except:
        return 'nan'
# Zustand des Hauses
def sammle_objektzustand(driver):
    try:
        return driver.find_element(By.XPATH, '//dd[contains(@class, "is24qa-objektzustand grid-item")]').text
    except:
        return 'nan'
# Qualitaet der Ausstattung
def sammle_quali_ausstattung(driver):
    try:
        return driver.find_element(By.XPATH, '//dd[contains(@class, "is24qa-qualitaet-der-ausstattung grid-item")]').text
    except:
        return 'nan'
# Heizungsart
def sammle_heizungsart(driver):
    try:
        return driver.find_element(By.XPATH, '//dd[contains(@class, "is24qa-heizungsart grid-item")]').text
    except:
        return 'nan'
# Wesentliche Energietraeger
def sammle_energietraeger(driver):
    try:
        return driver.find_element(By.XPATH, '//dd[contains(@class, "is24qa-wesentliche-energietraeger grid-item")]').text
    except:
        return 'nan'
# Liegt Energieausweis vor?
def sammle_energieausweis_existiert(driver):
    try:
        return 'ja' if driver.find_element(By.XPATH, '//dd[contains(@class, "is24qa-energieausweis grid-item")]').text == 'liegt vor' else 'nein'
    except:
        return 'nan'
# Typ des Energieausweises
def sammle_energieausweistyp(driver):
    try:
        return driver.find_element(By.XPATH, '//dd[contains(@class, "is24qa-energieausweistyp grid-item")]').text
    except:
        return 'nan'
# Endenergiebedarf
def sammle_endenergiebedarf(driver):
    try:
        return driver.find_element(By.XPATH, '//dd[contains(@class, "is24qa-endenergiebedarf grid-item")]').text
    except:
        return 'nan'
# Energieklasse
def sammle_energieklasse(driver):
    try:
        return driver.find_element(By.XPATH, '//span[@class = "energy-efficiency-class"]//img').get_attribute('src').split('/')[-1:][0].split('.')[0]
    except:
        return 'nan'
# Beschreibender Text zur Ausstattung
def sammle_text_ausstattung(driver):
    try:
        return driver.find_element(By.XPATH, '//pre[@class="is24qa-ausstattung text-content short-text"]').text
    except:
        return 'nan'
# Beschreibender Text zur Lage des Hauses
def sammle_text_lage(driver):
    try:
        return driver. find_element(By.XPATH, '//pre[@class="is24qa-lage text-content short-text"]').text
    except:
        return 'nan'
# Allgemeine Objektbeschreibung
def sammle_text_objektbeschreibung(driver):
    try:
        return driver.find_element(By.XPATH, '//pre[@class="is24qa-objektbeschreibung text-content full-text"]').text
    except:
        return 'nan'
# Exposé-Titel    
def sammle_expose_titel(driver):
    try:
        return driver.find_element(By.XPATH, '//h1[@id = "expose-title"]').text
    except:
        return 'nan'
def sammle_veroeffentlichungsdatum(driver):
    try:
        return driver.find_elements(By.XPATH, '//div[contains(@class, "premium-stats-item grid-item")]')[1].text
    except:
        return 'nan'
def neubau_sammler(page_card):
    try:
        return page_card.find_element(By.XPATH, './/div[@class = "indicator indicator--brand-white indicator--elevation"]').text
    except:
        return 'nan'
def sammle_neubau_bestand(driver):
    page_results = driver.find_element(By.XPATH, "//div[@data-elementtype = 'hybridViewCardContainer']")
    page_cards = page_results.find_elements(By.XPATH, ".//div[contains(@class, 'listing-card card-listing-')]")
    return [neubau_sammler(card) for card in page_cards]
def sammle_seiteninformationen(driver, url):
    return pd.DataFrame({
            'Titel': [sammle_expose_titel(driver)],
            'Adresse': [sammle_adresse(driver)],
            'Kaufpreis_Gesamt': [sammle_kaufpreis(driver)],
            'Kaufpreis_pro_QM': [sammle_preis_pro_qm(driver)],
            'Wohnflaeche_in_QM': [sammle_wohn_qm(driver)],
            'Grundstuecksflaeche_in_QM': [sammle_grundst_qm(driver)],
            'Keller_existiert': [sammle_keller_existiert(driver)],
            'Anzahl_Zimmer': [sammle_anzahl_zimmer(driver)],
            'Anzahl_Badezimmer': [sammle_anzahl_bad(driver)],
            'Anzahl_Schlafzimmer': [sammle_anzahl_schlafzimmer(driver)],
            'Anzahl_Etagen': [sammle_anzahl_etagen(driver)],
            'Anzahl_Autostellplaetze': [sammle_anzahl_garage(driver)],
            'Haustyp': [sammle_haustyp(driver)],
            'Maklerprovision': [sammle_marklerprovision(driver)],
            'Baujahr': [sammle_baujahr(driver)],
            'Objektzustand': [sammle_objektzustand(driver)],
            'Ausstattungsqualitaet': [sammle_quali_ausstattung(driver)],
            'Heizungsart': [sammle_heizungsart(driver)],
            'Wesentliche_Energietraeger': [sammle_energietraeger(driver)],
            'Energieausweis_vorliegend': [sammle_energieausweis_existiert(driver)],
            'Typ_Energieausweis': [sammle_energieausweistyp(driver)],
            'Endenergiebedarf': [sammle_endenergiebedarf(driver)],
            'Energieklasse': [sammle_energieklasse(driver)],
            'Text_Objektbeschreibung': [sammle_text_objektbeschreibung(driver)],
            'Text_Ausstattung': [sammle_text_ausstattung(driver)],
            'Text_Lage': [sammle_text_lage(driver)],            
            'URL': [url],
            'Bauprojekt': [bauprojekt],
            'Veroeffentlichungsdatum': [sammle_veroeffentlichungsdatum(driver)],
            'Download_Zeit': [datetime.now().strftime("%y-%m-%d %H:%M")]
        })

# Sammle auf Suchseite die URLs der einzelnen Objekte
def sammle_SuchURLs(driver):
    page_results = driver.find_element(By.XPATH, "//div[@data-elementtype = 'hybridViewCardContainer']")
    page_cards = page_results.find_elements(By.XPATH, ".//div[contains(@class, 'listing-card card-listing-')]")
    return [card.find_element(By.XPATH, './/a[@data-exp-referrer="HYBRID_VIEW_LISTING"]').get_attribute('href') for card in page_cards]

# Loope durch Suchseiten-URLs und fuege die entsprechenden
# detaillierten Objektinformationen zu dem data frame hinzu
def fuege_suchseiteninfos_hinzu(driver, df):
    URLs = sammle_SuchURLs(driver)
    Bauprojekte = sammle_neubau_bestand(driver)
    for URL, Bauprojekt in zip(URLs, Bauprojekte):
        try:
            driver.get(URL)
            time.sleep(1 + unif(1, 2))
            element = WebDriverWait(driver, 10).until(
                    EC.presence_of_element_located((By.XPATH, '//dd[contains(@class, "is24qa-kaufpreis grid-item")]'))
            )    # Als Indikator, um zu warten, bis Element geladen ist
            new_row = sammle_seiteninformationen(driver, URL, Bauprojekt)
            df = pd.concat([df, new_row], ignore_index = True).drop_duplicates(
                subset = ['Titel', 'Adresse', 'Kaufpreis_Gesamt', 'Kaufpreis_pro_QM', 'Wohnflaeche_in_QM',
                       'Grundstuecksflaeche_in_QM', 'Keller_existiert', 'Anzahl_Zimmer',
                       'Anzahl_Badezimmer', 'Anzahl_Autostellplaetze', 'Haustyp',
                       'Maklerprovision', 'Baujahr', 'Objektzustand', 'Ausstattungsqualitaet',
                       'Heizungsart', 'Wesentliche_Energietraeger',
                       'Energieausweis_vorliegend', 'Typ_Energieausweis', 'Endenergiebedarf',
                       'Energieklasse', 'Text_Objektbeschreibung', 'Text_Ausstattung',
                       'Text_Lage', 'URL'])
        except:
            next
    return df
```

### Browser Initialisieren & Startseite Laden

```
driver = webdriver.Firefox()
driver.maximize_window()
driver.set_page_load_timeout(30)
base_url = "https://www.immobilienscout24.de/Suche/de/nordrhein-westfalen/paderborn-kreis/haus-kaufen?sorting=2&enteredFrom=result_list"
driver.get(base_url)
```

Nach `driver.get(base_url)` müsste vermutlich im Browser die CAPTCHA manuell gelöst werden. In dem sich anschließend öffnenden Fenster sollte vermutlich das Cookie-Fenster manuell geschlossen werden. Erst dann sollte mit dem folgenden Code fortgefahren werden.

### Herunterladen der Daten

```
# Leerer Dataframe
df = pd.DataFrame(columns = ['Titel', 'Adresse', 'Kaufpreis_Gesamt', 'Kaufpreis_pro_QM', 'Wohnflaeche_in_QM',
       'Grundstuecksflaeche_in_QM', 'Keller_existiert', 'Anzahl_Zimmer',
       'Anzahl_Badezimmer', 'Anzahl_Schlafzimmer', 'Anzahl_Etagen', 
       'Anzahl_Autostellplaetze', 'Haustyp',
       'Maklerprovision', 'Baujahr', 'Objektzustand', 'Ausstattungsqualitaet',
       'Heizungsart', 'Wesentliche_Energietraeger',
       'Energieausweis_vorliegend', 'Typ_Energieausweis', 'Endenergiebedarf',
       'Energieklasse', 'Text_Objektbeschreibung', 'Text_Ausstattung',
       'Text_Lage', 'URL', 'Bauprojekt', 'Veroeffentlichungsdatum', 'Download_Zeit'])

# Fülle den Dataframe
i = 0
j = 0
while i < 1:
    j += 1
    print('Seite ' + str(j) + '\n')
    curr_url = driver.current_url
    try:
        df = fuege_suchseiteninfos_hinzu(driver, df)
        driver.get(curr_url)
        time.sleep(1 + unif(1, 2))
        nxt_button = WebDriverWait(driver, 10).until(
            EC.presence_of_element_located((By.XPATH, "//button[@data-testid='pagination-button-next']"))
        )
        next_button_disab = driver.find_element(By.XPATH, "//button[@data-testid='pagination-button-next']").get_attribute("aria-disabled")
    except:
        print('Prozess vorzeitig gestoppt')
        break
    if next_button_disab == "true":
        print('Prozess regulär gestoppt')
        break
    nxt_button.click()
    time.sleep(10 + unif(1, 2))
    elems = WebDriverWait(driver, 10).until(
        EC.presence_of_all_elements_located((By.XPATH, "//div[@data-elementtype = 'hybridViewCardContainer']//div[contains(@class, 'listing-card card-listing-')]"))
    )
```

### Speichern des Datensatzes

```
path_for_saving = ''  # Ersetze durch den Pfad, an dem die Daten gespeichert werden sollen
df.to_csv(path_for_saving, sep = '|', index = False)
```
