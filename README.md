# SQLi

# Penetrační test: SQL Injection Write-up

Tento report dokumentuje průběh Black Box penetračního testu webového vyhledávače zaměstnanců na adrese `http://[192.168.135.10]:5000`. Cílem bylo ověřit zranitelnost vůči SQL Injection (SQLi) a exfiltrovat skrytou vlajku z databáze.

## 1. Průzkum (Zjištění zranitelnosti)
Nejprve bylo nutné ověřit, zda vyhledávací pole dostatečně filtruje uživatelský vstup. 

Do pole "Jméno uživatele" byl vložen testovací znak – jednoduchý apostrof (`'`). 
Aplikace na tento vstup reagovala vypsáním chybové hlášky přímo do uživatelského rozhraní:
`SQL Chyba: unrecognized token: "'"`

**Závěr průzkumu:** Aplikace je zranitelná vůči SQL Injection. Vložený apostrof narušil syntaxi původního SQL dotazu na backendu. Znění chybové hlášky (*unrecognized token*) navíc s vysokou pravděpodobností prozradilo, že použitým databázovým systémem je **SQLite**.

## 2. Zjištění struktury databáze
Pro úspěšnou exfiltraci dat pomocí techniky `UNION Based SQLi` bylo nezbytné nejprve zmapovat strukturu původního dotazu a samotné databáze.

### 2.1 Zjištění počtu sloupců
Aby bylo možné použít operátor `UNION`, musí mít injektovaný dotaz shodný počet sloupců jako dotaz původní. K určení tohoto počtu byla využita klauzule `ORDER BY`, která řadí výsledky podle indexu sloupce. Postupně byly zadávány tyto payloady (znak `--` slouží v SQLite jako komentář pro zahození zbytku dotazu):

1. `' ORDER BY 1--` (proběhlo bez chyby)
2. `' ORDER BY 2--` (proběhlo bez chyby)
3. `' ORDER BY 3--` (následovala chyba)

Při pokusu o seřazení podle 3. sloupce databáze vrátila chybu: `SQL Chyba: 1st ORDER BY term out of range - should be between 1 and 2`. 
**Závěr:** Původní dotaz vrací přesně **2 sloupce**.

### 2.2 Vylistování názvů tabulek
S vědomím, že dotaz používá 2 sloupce, bylo možné přistoupit k výpisu existujících tabulek. V systému SQLite se metadata uchovávají v systémové tabulce `sqlite_master`. Byl použit následující payload:

`neexistuje' UNION SELECT 1, group_concat(name) FROM sqlite_master WHERE type='table'--`

*Vysvětlení:* Řetězec `neexistuje` zajistil, že původní dotaz nevrátí žádné legitimní uživatele a uvolní místo na obrazovce. Na pozici prvního sloupce byla vložena statická hodnota `1` a na pozici druhého sloupce (který se viditelně vypisuje v aplikaci) byla použita funkce `group_concat(name)`, která vypsala všechny názvy tabulek do jednoho řádku.
**Výsledek:** Databáze vypsala tabulky: `users, config`.

### 2.3 Zjištění struktury cílové tabulky
Vzhledem k zadání (hledání konfiguračních údajů) byla jako cíl vybrána tabulka `config`. Pro zjištění jejích sloupců byl vypsán DDL příkaz, kterým byla tabulka vytvořena (sloupec `sql` v `sqlite_master`):

`neexistuje' UNION SELECT 1, sql FROM sqlite_master WHERE type='table' AND name='config'--`

**Výsledek:** Databáze vrátila strukturu `CREATE TABLE config (key TEXT, value TEXT)`. Tabulka tedy obsahuje sloupce `key` a `value`.

## 3. Exfiltrace dat (Získání vlajky)
Vzhledem k tomu, že cílová tabulka `config` má přesně 2 sloupce (`key` a `value`), což perfektně odpovídá potřebám našeho UNION dotazu, nebylo nutné používat zástupné hodnoty. 

Byl sestrojen finální payload pro vypsání celého obsahu tabulky `config`:

`neexistuje' UNION SELECT key, value FROM config--`

**Závěr:** Po odeslání tohoto payloadu aplikace na obrazovku místo standardních výsledků vyhledávání vypsala obsah tabulky `config`, včetně hledané vlajky. Exfiltrace byla úspěšná.
