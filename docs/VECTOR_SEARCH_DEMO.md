# Vector Search Demo - Dokumentation

## Übersicht

Die Vector Search Demo ist eine interaktive Seite, die die Limitationen der reinen Vektorsuche demonstriert und verschiedene Techniken zeigt, um die Suchqualität zu verbessern.

## Aktuelle Features

### 1. Standard Vektorsuche
- Reine semantische Suche basierend auf Cosine-Ähnlichkeit
- Zeigt typische Limitationen wie Synonym-Probleme, Negationen, Fachbegriffe

### 2. Hybrid Search
**Status:** ✅ Implementiert

Kombiniert Vektor-Ähnlichkeit mit Keyword-Matching (BM25-ähnlich).

**Vorteile:**
- Findet exakte Begriffe auch bei niedriger semantischer Ähnlichkeit
- Nutzt Tags/Kategorien für besseres Matching
- Partial Matching für Wortteile

**Implementierung:**
- [HybridSearchService.cs](../src/Dojo.Rag.Api/Services/HybridSearchService.cs)
- Gewichtung: 70% Vektor, 30% Keyword (konfigurierbar)

### 3. Query Expansion
**Status:** ✅ Implementiert

Verwendet ein LLM um die Suchanfrage mit Synonymen und verwandten Begriffen zu erweitern.

**Vorteile:**
- Findet Dokumente mit unterschiedlicher Terminologie
- Überbrückt Synonym-Lücken
- Besonders effektiv für Fachbegriffe

**Implementierung:**
- [QueryExpansionService.cs](../src/Dojo.Rag.Api/Services/QueryExpansionService.cs)
- Nutzt Chat-Model für Expansion

---

## Geplante Verbesserungen

### 4. Reranking (Cross-Encoder)
**Status:** 🔜 Geplant

**Beschreibung:**
Nach der initialen Vektorsuche werden die Top-K Ergebnisse mit einem Cross-Encoder Model neu bewertet. Cross-Encoder betrachten Query und Dokument gemeinsam und können feinere Relevanz-Urteile treffen.

**Vorteile:**
- Höhere Präzision als Bi-Encoder (Embedding-basiert)
- Kann Kontext besser verstehen
- Besonders gut bei ambigen Queries

**Implementierungsansatz:**
```csharp
public interface IRerankingService
{
    Task<List<SearchResultItem>> RerankAsync(
        string query,
        List<SearchResultItem> candidates,
        int topK = 5,
        CancellationToken cancellationToken = default);
}
```

**Optionen:**
- Lokales Model (ms-marco-MiniLM-L-6-v2)
- API-basiert (Cohere Rerank, Jina Reranker)
- LLM-basiertes Reranking (langsamer aber flexibler)

**Aufwand:** Mittel - benötigt zusätzliches ML-Model oder API-Integration

---

### 5. Semantic Chunking
**Status:** 🔜 Geplant

**Beschreibung:**
Statt Dokumente nach fixer Zeichenzahl zu chunken, werden semantische Grenzen verwendet (Absätze, Themenübergänge).

**Vorteile:**
- Chunks enthalten vollständige Gedanken
- Bessere Embedding-Qualität
- Weniger abgeschnittene Sätze

**Implementierungsansatz:**
```csharp
public interface ISemanticChunker
{
    Task<List<DocumentChunk>> ChunkBySemanticBoundariesAsync(
        string content,
        int targetChunkSize = 500,
        CancellationToken cancellationToken = default);
}
```

**Optionen:**
- Satzgrenzen-basiert (NLTK/spaCy-ähnlich)
- Embedding-Distanz zwischen aufeinanderfolgenden Sätzen
- LLM-basierte Themenanalyse

**Aufwand:** Mittel - Anpassung in DocumentChunker

---

### 6. Min-Score Threshold Slider
**Status:** 🔜 Geplant

**Beschreibung:**
Dynamische Anpassung des Minimum-Relevanz-Scores zur Laufzeit über einen UI-Slider.

**Vorteile:**
- Benutzer kann Precision/Recall Trade-off steuern
- Gut für Demos und Experimente
- Einfach zu verstehen

**Implementierungsansatz:**
- Frontend: Range-Slider (0.0 - 1.0)
- Backend: Parameter an Search-Endpoint durchreichen
- Visualisierung: Zeigt wie viele Ergebnisse bei welchem Threshold

**Aufwand:** Gering - nur UI + Parameter

---

### 7. Multi-Vector Search
**Status:** 🔜 Geplant (komplex)

**Beschreibung:**
Mehrere Embeddings pro Chunk speichern: Titel/Überschrift, Hauptinhalt, extrahierte Keywords separat embedden.

**Vorteile:**
- Spezifischere Suche möglich
- Gewichtung verschiedener Aspekte
- Bessere Ergebnisse bei strukturierten Dokumenten

**Implementierungsansatz:**
```csharp
public record MultiVectorChunk(
    string Id,
    string Content,
    ReadOnlyMemory<float> ContentEmbedding,
    ReadOnlyMemory<float>? TitleEmbedding,
    ReadOnlyMemory<float>? KeywordEmbedding
);
```

**Aufwand:** Hoch - erfordert Schema-Änderung in Qdrant, neue Ingestion-Logik

---

### 8. Contextual Embeddings
**Status:** 🔜 Geplant

**Beschreibung:**
Beim Embedding eines Chunks wird der Kontext (vorheriger/nächster Absatz, Dokumenttitel) mit einbezogen.

**Vorteile:**
- Chunks haben mehr Kontext-Information im Embedding
- Bessere Disambiguation
- Hilfreich bei kurzen Chunks

**Implementierungsansatz:**
```csharp
string contextualText = $"Dokument: {documentTitle}\n" +
                        $"Vorheriger Abschnitt: {previousChunkSummary}\n" +
                        $"Inhalt: {chunkContent}";
var embedding = await GenerateEmbeddingAsync(contextualText);
```

**Aufwand:** Mittel - Anpassung Ingestion Pipeline

---

### 9. HyDE (Hypothetical Document Embeddings)
**Status:** 🔜 Geplant

**Beschreibung:**
Statt die Query direkt zu embedden, generiert ein LLM zuerst ein hypothetisches Dokument, das die Antwort enthalten würde. Dieses wird dann embedded.

**Vorteile:**
- Query und Dokument sind im gleichen "Stil"
- Besser bei kurzen/vagen Queries
- Kann fehlende Begriffe ergänzen

**Implementierungsansatz:**
```csharp
public interface IHyDEService
{
    Task<string> GenerateHypotheticalDocumentAsync(
        string query,
        CancellationToken cancellationToken = default);
}
```

**Aufwand:** Gering - ähnlich zu Query Expansion, nutzt LLM

---

### 10. Graph-Vector Search (HNSW)
**Status:** 🔜 Geplant

**Beschreibung:**
Verwendet einen graphbasierten ANN-Index (HNSW = Hierarchical Navigable Small World), um ähnliche Vektoren schneller zu finden. Die Suche ist approximativ und lässt sich ueber Parameter zwischen Recall und Geschwindigkeit steuern.

**Vorteile:**
- Deutlich schnellere Suche bei grossen Datenmengen
- Skalierbar fuer Echtzeit-Use-Cases
- Recall/Speed-Tradeoff konfigurierbar (z. B. über `efSearch`)

**Implementierungsansatz:**
Indexierung als HNSW in der Vector-DB konfigurieren und zur Laufzeit `efSearch` (oder vergleichbar) setzen.
```csharp
public record HnswConfig(
    int M = 16,
    int EfConstruction = 200,
    int EfSearch = 64);
```

**Aufwand:** Mittel - Index-Konfiguration + optionaler UI-Toggle fuer Approximation

---

## Demo-Szenarien

Die JSON-Datei `docs/demo-sentences.json` enthält vordefinierte Szenarien, die spezifische Limitationen demonstrieren:

| Szenario | Query | Problem | Beste Lösung |
|----------|-------|---------|--------------|
| Synonym-Problem | "Java Getränk" | Java ≠ Kaffee semantisch | Query Expansion |
| Negations-Problem | "Kaffee ohne Hitze" | Negation schwer zu verstehen | Hybrid Search |
| Fachbegriff-Problem | "Schaum auf dem Kaffee" | Crema nicht erkannt | Query Expansion |
| Zahlen-Problem | "Temperatur unter 95 Grad" | Numerischer Vergleich | Hybrid Search |

---

## API-Endpunkte

### GET /api/vectorsearchdemo/sentences
Gibt alle Demo-Sätze und -Szenarien zurück.

### POST /api/vectorsearchdemo/initialize
Generiert Embeddings für alle Demo-Sätze. Muss vor der Suche aufgerufen werden.

### POST /api/vectorsearchdemo/search
Führt Suche mit optionalen Verbesserungen durch.

**Request:**
```json
{
  "query": "Kaffee Temperatur",
  "enhancements": {
    "useHybridSearch": true,
    "useQueryExpansion": false
  },
  "topK": 5
}
```

**Response:**
```json
{
  "standardResults": { ... },
  "enhancedResults": { ... },
  "originalQuery": "Kaffee Temperatur",
  "appliedEnhancements": { ... }
}
```

### GET /api/vectorsearchdemo/status
Gibt den aktuellen Status der Demo zurück (initialisiert, Anzahl Embeddings).

---

## Architektur

```
┌─────────────────────────────────────────────────────────────┐
│                    VectorSearchDemo.tsx                      │
├─────────────────┬───────────────────┬───────────────────────┤
│ SentenceList    │ SearchInput       │ EnhancementToggles    │
│ (Demo-Sätze)    │ (Query-Eingabe)   │ (Hybrid, Expansion)   │
├─────────────────┴───────────────────┴───────────────────────┤
│                 Side-by-Side Results                         │
│  ┌──────────────────────┬──────────────────────┐            │
│  │ Standard Suche       │ Mit Verbesserungen   │            │
│  │ - Ergebnis 1 (72%)   │ - Ergebnis 1 (89%)   │            │
│  │ - Ergebnis 2 (65%)   │ - Ergebnis 2 (85%)   │            │
│  └──────────────────────┴──────────────────────┘            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │ VectorSearchDemoController    │
              └───────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ EmbeddingService│ │HybridSearchSvc  │ │QueryExpansionSvc│
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

---

## Weiterentwicklung

Um ein neues Enhancement hinzuzufügen:

1. **Service erstellen:** `src/Dojo.Rag.Api/Services/New<Feature>Service.cs`
2. **DI registrieren:** In `Program.cs` hinzufügen
3. **Models erweitern:** `SearchEnhancements` um neues Flag erweitern
4. **Controller anpassen:** Logik in `PerformEnhancedSearchAsync` einbauen
5. **Frontend Toggle:** In `VectorSearchDemo.tsx` neuen Toggle hinzufügen
6. **Types erweitern:** TypeScript-Interface aktualisieren
