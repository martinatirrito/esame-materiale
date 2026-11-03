# LEFT JOIN (nodi con valore 0)

## Indice
1. [LEFT JOIN vs join nel WHERE](#1-left-join-vs-join-nel-where)
2. [Quando serve all'esame](#2-quando-serve-allesame)
3. [Esempio in stile esame (SQL)](#3-esempio-in-stile-esame-sql)
4. [Trappola: i filtri vanno dentro la subquery](#4-trappola-i-filtri-vanno-dentro-la-subquery)
5. [Alternativa senza LEFT JOIN (più sicura)](#5-alternativa-senza-left-join-più-sicura)
6. [Codice completo: DTO + DAO + Model](#6-codice-completo-dto--dao--model)
7. [Note: soglia e Decimal](#7-note-soglia-e-decimal)

---

## 1. LEFT JOIN vs join nel WHERE

Il join che scrivi nel WHERE è un **INNER JOIN**: tiene solo le righe che hanno una corrispondenza in **entrambe** le tabelle.
Il **LEFT JOIN** invece tiene **tutte** le righe della tabella di sinistra, anche quelle senza corrispondenza, e lì mette NULL nelle colonne dell'altra tabella.

```sql
SELECT ar.ArtistId, ar.Name, COUNT(al.AlbumId) AS nAlbum
FROM Artist ar LEFT JOIN Album al ON al.ArtistId = ar.ArtistId
GROUP BY ar.ArtistId, ar.Name
```
Questa query ti dà anche gli artisti con 0 album. Con il join nel WHERE quegli artisti sparirebbero.

## 2. Quando serve all'esame

Quasi mai. Ti serve solo se il testo dice che i nodi sono *tutti* gli X, **anche quelli con valore 0**.
Ma anche lì puoi evitarlo (vedi sezione 5). Gli **archi** richiedono per forza una corrispondenza, quindi lì il join nel WHERE va benissimo: continua a usare quello.

## 3. Esempio in stile esame (SQL)

**Testo:** "I nodi sono *tutti* gli artisti che hanno almeno una traccia di genere G. Il valore di ogni nodo è la quantità venduta di quelle tracce nell'anno Y (**0 se non ne ha vendute**)."

```sql
SELECT ar.ArtistId, ar.Name, COALESCE(v.tot, 0) AS valore
FROM Artist ar
     LEFT JOIN (SELECT al.ArtistId, SUM(il.Quantity) AS tot
                FROM Album al, Track t, InvoiceLine il, Invoice i
                WHERE al.AlbumId = t.AlbumId AND t.TrackId = il.TrackId
                  AND il.InvoiceId = i.InvoiceId
                  AND t.GenreId = %s AND YEAR(i.InvoiceDate) = %s
                GROUP BY al.ArtistId) v
     ON v.ArtistId = ar.ArtistId
WHERE ar.ArtistId IN (SELECT al.ArtistId
                      FROM Album al, Track t
                      WHERE al.AlbumId = t.AlbumId AND t.GenreId = %s)
```

Come funziona:
- `v` è il solito **blocco valore**, costruito con i join nel WHERE come sempre.
- Il LEFT JOIN attacca `v` agli artisti. Un artista che non compare in `v` resta comunque nel risultato, con `tot = NULL`.
- `COALESCE(v.tot, 0)` trasforma quel NULL in 0.
- Il WHERE finale tiene solo gli artisti del genere G.
- I parametri sono `(genre, anno, genre)`, in quest'ordine.

## 4. Trappola: i filtri vanno dentro la subquery

⚠ I filtri su genere e anno vanno **dentro** la subquery `v`.
Se li metti nel WHERE esterno su colonne di `v`, scartano le righe NULL e il LEFT JOIN torna a comportarsi come un join normale.

## 5. Alternativa senza LEFT JOIN (più sicura)

1. **Query nodi:** `SELECT DISTINCT ar.* ... WHERE t.GenreId = %s`. Con questa crei i nodi e l'idMap.
2. **Query valore:** il blocco `v` di sopra da solo. Ti restituisce le coppie `(ArtistId, tot)`.
3. **In Python:**
```python
valori = {row["ArtistId"]: row["tot"] for row in DAO.getValori(g, y)}
for a in self._nodes:
    a.valore = valori.get(a.ArtistId, 0)
```
Danno lo stesso risultato. Se non ti viene il LEFT JOIN, usa questa.

### Dove va ogni pezzo (versione senza LEFT JOIN)

**DTO**: `valore` con default `= 0`, perché la query nodi (`ar.ArtistId, ar.Name`) non lo restituisce.
```python
@dataclass
class Artist:
    ArtistId: int
    Name: str
    valore: int = 0

    def __hash__(self):
        return hash(self.ArtistId)
```

**DAO**: due metodi.
```python
@staticmethod
def getNodi(genre):
    conn = DBConnect.get_connection()
    if conn is None:
        print("Connessione fallita")
        return []
    cursor = conn.cursor(dictionary=True)
    query = """SELECT DISTINCT ar.ArtistId, ar.Name
               FROM Artist ar, Album al, Track t
               WHERE ar.ArtistId = al.ArtistId AND al.AlbumId = t.AlbumId
                 AND t.GenreId = %s"""
    cursor.execute(query, (genre,))
    result = []
    for row in cursor:
        result.append(Artist(**row))
    cursor.close()
    conn.close()
    return result

@staticmethod
def getValori(genre, anno):
    conn = DBConnect.get_connection()
    if conn is None:
        print("Connessione fallita")
        return []
    cursor = conn.cursor(dictionary=True)
    query = """SELECT al.ArtistId, SUM(il.Quantity) AS tot
               FROM Album al, Track t, InvoiceLine il, Invoice i
               WHERE al.AlbumId = t.AlbumId AND t.TrackId = il.TrackId
                 AND il.InvoiceId = i.InvoiceId
                 AND t.GenreId = %s AND YEAR(i.InvoiceDate) = %s
               GROUP BY al.ArtistId"""
    cursor.execute(query, (genre, anno))
    result = []
    for row in cursor:
        result.append(row)          # dizionario {"ArtistId": .., "tot": ..}
    cursor.close()
    conn.close()
    return result
```

**Model**: le tre righe vanno in `buildGraph`, **dopo** aver creato i nodi e **prima** degli archi.
```python
def buildGraph(self, genre, anno):
    self._graph.clear()
    self._idMap = {}
    nodi = DAO.getNodi(genre)
    for a in nodi:
        self._idMap[a.ArtistId] = a
    self._graph.add_nodes_from(nodi)

    # --- valore dei nodi (0 se non compare nella query) ---
    valori = {row["ArtistId"]: row["tot"] for row in DAO.getValori(genre, anno)}
    for a in nodi:
        a.valore = int(valori.get(a.ArtistId, 0))

    for row in DAO.getArchi(genre, anno):   # la tua solita query archi
        a1 = self._idMap.get(row["id1"])
        a2 = self._idMap.get(row["id2"])
        if a1 is not None and a2 is not None:
            self._graph.add_edge(a1, a2, weight=row["peso"])
```

## 6. Codice completo: DTO + DAO + Model

### DTO (`model/artist.py`)
Il campo `valore` deve avere lo stesso nome della colonna della query.
```python
from dataclasses import dataclass

@dataclass
class Artist:
    ArtistId: int
    Name: str
    valore: int

    def __hash__(self):
        return hash(self.ArtistId)

    def __str__(self):
        return f"{self.Name} ({self.valore})"
```

### DAO
```python
@staticmethod
def getNodiConValore(genre, anno):
    conn = DBConnect.get_connection()
    if conn is None:
        print("Connessione fallita")
        return []
    cursor = conn.cursor(dictionary=True)
    query = """SELECT ar.ArtistId, ar.Name, COALESCE(v.tot, 0) AS valore
               FROM Artist ar
                    LEFT JOIN (SELECT al.ArtistId, SUM(il.Quantity) AS tot
                               FROM Album al, Track t, InvoiceLine il, Invoice i
                               WHERE al.AlbumId = t.AlbumId AND t.TrackId = il.TrackId
                                 AND il.InvoiceId = i.InvoiceId
                                 AND t.GenreId = %s AND YEAR(i.InvoiceDate) = %s
                               GROUP BY al.ArtistId) v
                    ON v.ArtistId = ar.ArtistId
               WHERE ar.ArtistId IN (SELECT al.ArtistId
                                     FROM Album al, Track t
                                     WHERE al.AlbumId = t.AlbumId AND t.GenreId = %s)"""
    cursor.execute(query, (genre, anno, genre))
    result = []
    for row in cursor:
        result.append(Artist(**row))
    cursor.close()
    conn.close()
    return result
```
Ci sono tre `%s`, quindi la tupla ha 3 valori nell'ordine in cui compaiono nel testo: genere, anno, genere.

### Model
```python
def buildGraph(self, genre, anno):
    self._graph.clear()
    self._idMap = {}
    nodi = DAO.getNodiConValore(genre, anno)
    for a in nodi:
        self._idMap[a.ArtistId] = a
    self._graph.add_nodes_from(nodi)

    for row in DAO.getArchi(genre, anno):   # la tua solita query archi
        a1 = self._idMap.get(row["id1"])
        a2 = self._idMap.get(row["id2"])
        if a1 is not None and a2 is not None:
            self._graph.add_edge(a1, a2, weight=row["peso"])

def getNodiOrdinatiPerValore(self):
    return sorted(self._graph.nodes, key=lambda a: (-a.valore, a.Name))
```

## 7. Note: soglia e Decimal

- Un artista che ha venduto 0 è comunque un nodo, con `a.valore == 0`.
- Se il testo chiede una **soglia** sul valore (≥ S), non serve HAVING: aggiungi `AND COALESCE(v.tot, 0) >= %s` al WHERE esterno e passa S come 4° parametro: `(genre, anno, genre, soglia)`.
- `COALESCE` ti restituisce un Decimal da MySQL. Se poi lo usi nei conti, convertilo con `int(...)` o `float(...)`.
