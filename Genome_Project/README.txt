================================================================================
                         GENOME PROJECT - DOKUMENTATION
================================================================================

PROJEKTZIEL
-----------
Dieses Projekt analysiert Daten aus dem International Genome Sample Resource 
(IGSR), einer Sammlung von Genomsequenzdaten aus verschiedenen menschlichen 
Populationen weltweit. Das Ziel ist es, genetische Variationen zwischen 
verschiedenen ethnischen Gruppen und Populationen zu untersuchen und zu 
visualisieren.

Die Daten stammen hauptsächlich aus dem 1000 Genomes Project, das zum Ziel 
hat, die genetische Variation innerhalb der menschlichen Spezies zu 
katalogisieren.


DATENSATZ: igsr_samples.tsv
---------------------------
Die Datei enthält Metadaten zu 4990 Proben aus verschiedenen Populationen.


SPALTENBESCHREIBUNG
-------------------

1. Sample name (Probenname)
   - Eindeutige Kennung für jede Probe
   - Format: z.B. "HG00152", "NA18505"
   - HG = Human Genome, NA = Naming convention from Coriell

2. Sex (Geschlecht)
   - Biologisches Geschlecht der Person
   - Werte: "male" oder "female"

3. Biosample ID
   - Eindeutige Kennung in der BioSample-Datenbank (NCBI/EBI)
   - Format: z.B. "SAME124593"
   - Verknüpft die Probe mit weiteren biologischen Metadaten

4. Population code (Populationscode)
   - Kurzcode für die spezifische Population
   - Beispiele:
     * GBR = British (Britisch)
     * FIN = Finnish (Finnisch)
     * CHB = Han Chinese (Han-Chinesen)
     * YRI = Yoruba (Nigeria)
     * JPT = Japanese (Japanisch)
     * MXL = Mexican Ancestry (Mexikanische Abstammung)
     * PUR = Puerto Rican (Puerto-Ricaner)
     * LWK = Luhya (Kenia)

5. Population name (Populationsname)
   - Vollständiger Name der Population
   - Beschreibt die ethnische/geografische Herkunft

6. Superpopulation code (Superpopulationscode)
   - Übergeordnete geografische/genetische Gruppierung
   - Werte:
     * EUR = European Ancestry (Europäische Abstammung)
     * EAS = East Asian Ancestry (Ostasiatische Abstammung)
     * AFR = African Ancestry (Afrikanische Abstammung)
     * AMR = American Ancestry (Amerikanische Abstammung)
     * SAS = South Asian Ancestry (Südasiatische Abstammung)

7. Superpopulation name (Superpopulationsname)
   - Vollständiger Name der Superpopulation
   - z.B. "European Ancestry", "African Ancestry"

8. Population elastic ID
   - Interne Kennung für die Elasticsearch-Datenbank
   - Wird für Datenbankabfragen verwendet

9. Data collections (Datensammlungen)
   - Liste der Projekte/Studien, in denen diese Probe enthalten ist
   - Beispiele:
     * "1000 Genomes on GRCh38" - 1000 Genomes Projekt auf Referenzgenom 38
     * "1000 Genomes 30x on GRCh38" - Hochauflösende Sequenzierung (30-fach)
     * "Geuvadis" - Genexpressionsanalyse-Projekt
     * "Human Pangenome Reference Consortium" - Pangenomprojekt
     * "Simons Genome Diversity Project" - Diversitätsprojekt


WICHTIGE BEGRIFFE
-----------------

GRCh37 / GRCh38:
  Versionen des menschlichen Referenzgenoms (Genome Reference Consortium)
  GRCh38 ist die aktuellere Version (seit 2013)

30x Coverage:
  Bedeutet, dass jede Base im Genom durchschnittlich 30-mal sequenziert wurde
  Höhere Coverage = höhere Genauigkeit

ONT (Oxford Nanopore Technology):
  Long-Read Sequenzierungstechnologie für längere DNA-Fragmente


MÖGLICHE ANALYSEN
-----------------
- Populationsverteilung visualisieren
- Geschlechterverteilung nach Population
- Anzahl Proben pro Superpopulation
- Vergleich der Datenverfügbarkeit zwischen Populationen
- Geografische Verteilung der Proben


================================================================================
Erstellt: 04.01.2026
Datenquelle: https://www.internationalgenome.org/
================================================================================
