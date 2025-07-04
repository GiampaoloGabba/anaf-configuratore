# Documentazione Sistema Configuratore Estintori

## Panoramica Generale

Il sistema configuratore estintori è un'applicazione web ASP.NET MVC che permette di creare preventivi personalizzati per estintori attraverso un processo di selezione guidata. Il sistema gestisce clienti, prodotti, certificazioni e opzioni con un complesso sistema di calcolo prezzi e regole di esclusione.

## Struttura Dati Principale

### Entità Core
- **Cliente**: Anagrafica con cluster geografico, settore e sconto associato
- **Prodotto**: Modello estintore con prezzo base e costo
- **Certificazione**: Standard di qualità che determina classi fuoco compatibili
- **Opzione**: Componenti aggiuntivi organizzati in gruppi con prezzi/costi
- **Coefficiente**: Moltiplicatori globali per adeguamento prezzi

### Relazioni Chiave
```
Cliente → ClusterNazioni (VariazionePerc)
Cliente → Settori (VariazionePerc)
Prodotto → ProdottiCombinazioni → Coefficienti specifici
Prodotto → ClassiFuocoCertProd → Certificazioni
Opzioni → OpzioniGruppi (ordinamento e raggruppamento)
OpzioniProdottiEsclusioni → Regole di incompatibilità
```

## Flusso Utente Completo

### 1. Selezione Cliente
**Azione**: L'utente seleziona un cliente dall'elenco dropdown
**Dati Caricati**:
- `ClusterId` (cluster geografico di appartenenza)
- `SettoreId` (settore di attività)  
- `Sconto` cliente (percentuale fissa)
- `ClusterNazioni.VariazionePerc` (variazione percentuale cluster)
- `Settori.VariazionePerc` (variazione percentuale settore)

**Calcolo Variazione Totale**:
```javascript
variazionePercTotale = variazionePercCluster + variazionePercSettore
```

### 2. Filtro Prodotti Disponibili
Il sistema filtra i prodotti in base al cliente selezionato:
```csharp
// Solo prodotti attivi E (non solo combinazioni OR ha combinazioni specifiche per cliente)
prodotti = contesto.Prodotti.Where(x => 
    x.Disattivato == false && 
    (x.SoloCombinazioni == false || 
     x.ProdottiCombinazioni.Any(y => y.NazioneId == cliente.ClusterId && 
                                     y.SettoreId == cliente.SettoreId)))
```

### 3. Selezione Prodotto
**Azione**: L'utente seleziona un prodotto dalla lista filtrata
**Dati Caricati**:
- `PrezzoNav` (prezzo base navigazione)
- `CostoNav` (costo base navigazione)
- Lista opzioni disponibili per il prodotto
- Coefficiente applicabile (logica priorità)

**Logica Coefficiente** (ConfiguratoreController.cs:32-43):
```csharp
// Default: coefficiente globale
ViewBag.Coefficiente = _contesto.Coefficienti.FirstOrDefault()?.Valore ?? 0;

// Override: se esiste combinazione specifica prodotto+nazione+settore
var combinazione = _contesto.ProdottiCombinazioni.FirstOrDefault(x => 
    x.ProdottoId == Id && 
    x.NazioneId == cliente.ClusterId && 
    x.SettoreId == cliente.SettoreId);

if (combinazione != null && combinazione.Coefficiente > 0) {
    ViewBag.Coefficiente = combinazione.Coefficiente;
}
```

### 4. Selezione Certificazione
**Azione**: L'utente seleziona una certificazione
**Effetto**: Filtra le classi fuoco compatibili
```csharp
return PartialView("Partials/_ClassiFuoco", 
    _contesto.ClassiFuoco.FirstOrDefault(x =>
        x.ClassiFuocoCertProd.Any(y => 
            y.CertificazioneId == Id && 
            y.ProdottoId == prodottoId)));
```

### 5. Configurazione Opzioni
**Processo**: Per ogni gruppo di opzioni (ordinato per `Ordinamento`):
- L'utente seleziona un'opzione dal dropdown
- Il sistema applica le regole di esclusione
- Calcola il prezzo parziale dell'opzione

**Dati Opzioni**:
- `Opzioni.Prezzo` (costo aggiuntivo)
- `Opzioni.Costo` (costo interno)
- Coefficiente applicabile (stesso del prodotto)

## Calcolo Prezzi - Formula Completa

### Step 1: Calcolo Prezzo Singola Opzione
Per ogni opzione selezionata:
```javascript
// Recupero prezzo base dall'attributo data-prezzo
var prezzo = $this.find("option:selected").data("prezzo");
var costo = $this.find("option:selected").data("costo");

// Conversione stringa → numero
prezzo = prezzo.replace(",",".")/1;
costo = costo.replace(",",".")/1;

// Applicazione variazione percentuale totale (cluster + settore)
var variazioneTotale = prezzo * variazionePercTotale / 100;
prezzo = prezzo + variazioneTotale;
```

### Step 2: Somma Prezzi Parziali
```javascript
prezzoTotale = 0.00;
costoTotale = 0.00;

$(".tot-parziale").each(function(){
    var prezzo = $(this).find(".prezzo-valore").text().replace(",",".");
    var costo = $(this).find(".costo-valore").text().replace(",",".");
    
    if (prezzo != "" && prezzo != "-" && costo != "" && costo != "-") {
        prezzoTotale = prezzoTotale + (prezzo/1);
        costoTotale = costoTotale + (costo/1);
    }
});
```

### Step 3: Applicazione Sconto Cliente
```javascript
var scontoClienteTotale = prezzoTotale * scontoCliente / 100;
prezzoTotale = prezzoTotale - scontoClienteTotale;
```

### Step 4: Applicazione Sconto Manuale
```javascript
var scontoManuale = 0;
var valore = $scontoManualeInput.val();

if (valore != "") {
    var perc = valore.indexOf('%') > -1;
    valore = valore.replace(",",".").replace("%","").trim();
    
    if ($.isNumeric(valore)) {
        if (perc === true) {
            // Sconto percentuale
            scontoManuale = (prezzoTotale * valore / 100).toFixed(2)/1;
        } else {
            // Sconto valore fisso
            scontoManuale = valore/1;
        }
        prezzoTotale = prezzoTotale - scontoManuale;
    }
}
```

### Step 5: Validazione Margine Minimo
```javascript
if (prezzoTotale < costoTotale) {
    $risultato.addClass("errore"); // Evidenzia errore visivamente
} else {
    $risultato.removeClass("errore");
}
```

## Sistema Esclusioni

### Struttura Regole JSON
Le regole di esclusione sono definite nel controller `EsclusioneOpzioni`:
```csharp
foreach (var esclusione in _contesto.OpzioniProdottiEsclusioni.Where(x => 
    x.ProdottoId == id && x.QueryEsclusione != "")) {
    
    model.Opzioni.Add(new EsclusioniOpzioniModel {
        Id = esclusione.Id,
        Gruppo = esclusione.Opzioni.OpzioneGruppoId ?? 0,
        OpzDaEliminare = esclusione.OpzioneSelezionataId,
        Query = esclusione.QueryEsclusione
    });
}
```

### Logica di Esclusione (esclusioni.js)
```javascript
function eseguiEsclusioni() {
    // Reset: riabilita tutte le opzioni
    $("select option:disabled").css("color","").removeAttr("disabled");
    
    esclusioniOpzioni.forEach(esclusione => {
        var $daNascondere = $("#gruppo_" + esclusione.Gruppo + 
                           " > option[data-id='" + esclusione.OpzDaEliminare + "']");
        
        if ($daNascondere.length > 0) {
            var esclusioni = toEsclusioni(esclusione.Query);
            
            // Verifica se le condizioni di esclusione sono soddisfatte
            if (controllaRegole(esclusioni.rules, esclusioni.condition, null)) {
                // Disabilita l'opzione
                $daNascondere.prop("selected", false)
                            .prop('disabled','disabled')
                            .css("color","#b3b3b3");
                
                // Aggiorna il totale
                selectAggiornaTotale($daNascondere.parent());
            }
        }
    });
}
```

### Operatori Supportati
- `equal`: L'opzione deve essere selezionata
- `not-equal`: L'opzione NON deve essere selezionata
- Condizioni multiple con `AND`/`OR`

### Esempio Regola Esclusione
```json
{
    "condition": "AND",
    "rules": [
        {
            "id": "gruppo_3",
            "operator": "equal", 
            "value": "15"
        },
        {
            "id": "gruppo_5",
            "operator": "not-equal",
            "value": "23"
        }
    ]
}
```
**Significato**: Disabilita l'opzione se gruppo_3 ha selezionato opzione 15 E gruppo_5 NON ha selezionato opzione 23.

## Regole di Business Implicite

### 1. Priorità Coefficienti
- **Globale**: Coefficiente da tabella `Coefficienti` (default)
- **Specifico**: Coefficiente da `ProdottiCombinazioni` per prodotto+cluster+settore (override)

### 2. Ordine Operazioni Calcolo
1. Applicazione coefficiente su prezzo base
2. Applicazione variazione percentuale (cluster + settore) 
3. Somma di tutti i prezzi parziali
4. Applicazione sconto cliente (percentuale su totale)
5. Applicazione sconto manuale (valore fisso o percentuale)
6. Controllo margine minimo

### 3. Margine di Sicurezza
Il prezzo finale non può mai essere inferiore al costo totale. Questo controllo impedisce vendite in perdita.

### 4. Gestione Arrotondamenti
- Prezzi visualizzati con 2 decimali (`toFixed(2)`)
- Calcoli intermedi mantengono precisione completa
- Controllo margine su valori arrotondati

## Flussi di Aggiornamento

### Cambio Cliente
1. Ricarica pagina con nuovo `clienteSel`
2. Filtra prodotti disponibili per nuovo cliente
3. Reset selezioni prodotto e opzioni

### Cambio Prodotto
1. Carica dati prodotto tramite AJAX (`/configuratore/Prodotto`)
2. Carica opzioni disponibili (`/configuratore/ProdottoOpzioni`)
3. Carica regole esclusioni (`/configuratore/esclusioneopzioni`)
4. Reset selezioni certificazione e opzioni

### Cambio Certificazione
1. Filtra classi fuoco compatibili (`/configuratore/ClassiFuoco`)
2. Aggiorna opzioni disponibili

### Cambio Opzione
1. Ricalcola prezzo parziale opzione
2. Esegue regole di esclusione
3. Ricalcola totale generale

## Autorizzazioni e Visibilità

### Controlli Visualizzazione (basati su `UtenteLoggato`)
- `VisualizzaPrezzi`: Mostra/nasconde prezzi nelle opzioni
- `VisualizzaCosto`: Mostra/nasconde costi nelle opzioni  
- `VisualizzaListino`: Modalità listino (prezzi senza sconti)
- `ScontoManuale`: Abilita/disabilita campo sconto manuale

### Ruoli
- Controller accessibile solo a ruolo `Admin` (`[LocalizedAuthorize(Roles = "Admin")]`)

## Output del Sistema

### Campi Calcolati Finali
- `totale`: Prezzo finale dopo tutti gli sconti
- `totaleNoSconto`: Prezzo prima degli sconti
- `costo`: Costo totale (prodotto + opzioni)
- `sconto`: Valore sconto cliente applicato
- `scontoManuale`: Valore sconto manuale applicato

### Validazione Stato
- Configurazione **valida**: Tutte le opzioni obbligatorie selezionate, `prezzo >= costo`
- Configurazione **non valida**: Opzioni mancanti o margine insufficiente (classe CSS `errore`)

Questo sistema implementa un configuratore prodotto completo con gestione dinamica di prezzi, sconti, compatibilità e validazioni per garantire configurazioni commercialmente sostenibili.