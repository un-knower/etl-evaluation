## Optimierung von Analytischen Abfragen über Statistical Linked Data durch Horizontale Skalierung

Dies ist die Projektseite der Abschlussarbeit "Optimierung von Analytischen Abfragen über Statistical Linked Data durch Horizontale Skalierung" von Sébastien Jelsch am Institut für Angewandte Informatik und formale Beschreibungssprachen - Karlsruher Institut für Technologie.

### Struktur der Projektseite

| Ordner        | Beschreibung      |
| ------------- |-------------------|
| [ausarbeitung](https://github.com/sjelsch/etl-evaluation/tree/master/ausarbeitung) | Enthält die Latex-Dateien sowie eine PDF-Version der Abschlussarbeit. |
| [etl-process](https://github.com/sjelsch/etl-evaluation/tree/master/etl-process) | Enthält den Java-Quellcode des implementierten ETL-Prozesses mit allen verwendeten Bibliotheken sowie eine ausführbare JAR-Datei. |
| [kylin](https://github.com/sjelsch/etl-evaluation/tree/master/kylin) | Auflistung der verwendeten MDX- und SQL-Abfragen für Apache Kylin. |
| [lib](https://github.com/sjelsch/etl-evaluation/tree/master/lib) | Verwendete Bibliotheken für die Transformation der RDF-Dateien in das N-Triples-Format. |
| [mysql](https://github.com/sjelsch/etl-evaluation/tree/master/mysql) | MySQL Schema, Import-Skript und eine Auflistung der verwendeten SQL-Abfragen für MySQL. |
| [open_virtuoso](https://github.com/sjelsch/etl-evaluation/tree/master/open_virtuoso) | Insert-Befehle für die Generierung der Levels, Import-Skript, Dump-Prozedur und eine Auflistung der verwendeten SPARQL-Abfragen für Open Virtuoso. |
| [results](https://github.com/sjelsch/etl-evaluation/tree/master/results) | Ergebnisse der Evaluation: 1.) Ausführungsdauer des ETL-Prozesses. 2.) Antwortzeiten der OLAP-Abfragen bei verschiedenen Datenmengen sowie Anzahl an DataNodes. |
| [ttl](https://github.com/sjelsch/etl-evaluation/tree/master/ttl) | BIBM-Konfigurationsdatei für die Transformation der TBL-Daten in das Turtle-Format sowie die verwendete DataStructureDefinition in der Turtle-Notation. |

### Cluster-Installation
Bei der Evaluation wurde [Cloudera Director](http://www.cloudera.com/content/www/en-us/products/cloudera-director.html) in der Version 1.5.2 zur Installation eines Apache Hadoop Clusters eingesetzt. Die Anforderungen sowie die notwendige Schritte zur Installation von Cloudera Director sind auf dem [Dokumentationsportal](http://www.cloudera.com/content/www/en-us/documentation/director/latest/topics/director_get_started_aws.html) von Cloudera aufgelistet.

Als Distribution wurde CDH (Cloudera Distribution Including Apache Hadoop) in der Version 5.3.8 mit der Auswahl *Core Hadoop with HBase* gewählt. Aus diesem Grund wurde das *Default Parcel Repository* durch die URL http://archive.cloudera.com/cdh5/parcels/5.3.8/ ersetzt.

Zur Durchführung der Evaluation sind auf dem MasterNode einige Programme notwendig. In der folgenden Tabelle sind alle Befehle aufgelistet, um diese Tools zu installieren:

| Befehl | Beschreibung |
|---|---|
| sudo yum install git | Installiert git, um dieses Repository zu pullen |
| sudo yum install ant | Installiert [Apache Ant](https://ant.apache.org/index.html) - Ein Kompilierungsprogramm |
| sudo yum groupinstall 'Development Tools' | Notwendig, um Star Schema Benchmark zu kompilieren |
| sudo yum install openssl-devel | Notwendig, um Open Virtuoso zu kompilieren |

### Datengenerierung mit dem Star Schema Benchmark

#### Generierung der TBL-Daten
Im Rahmen der Evaluation wurden die TBL-Daten mit dem Star Schema Benchmark [dbgen-Programm](https://github.com/electrum/ssb-dbgen) [1] erstellt.

**Einstellungen**
> DATABASE = SQLSERVER;
> MASCHINE = LINUX;

**Befehle**
> ./dbgen -s 1 -T a
> ./dbgen -s 10 -T a
> ./dbgen -s 20 -T a

#### Transformation der TBL-Daten in die Turtle-Notation
Die Generierung der RDF-Daten wurde mit dem Programm [Business Intelligence Benchmark](http://sourceforge.net/projects/bibm/) (BIBM) durchgeführt.

**Einstellung**
> java -Xmx8192M com.openlinksw.util.csv2ttl.Main

**Befehl**
> ./csv2ttl.sh -ext tbl -schema ../ttl/01_schema.json -d ../rdf-data ../ssb-dbgen/data/*

#### Deklaration der Fakten als QB Observations
Damit die Fakten im QB-Vokabular vorliegen, muss der SED-Befehl zum Hinzufügen der Property *qb:Observation* ausgeführt werden.

**Befehl**
> sed 's/a rdfh:lineorder ;/a rdfh:lineorder ; a <http:\/\/purl.org\/linked-data\/cube#Observation>; <http:\/\/purl.org\/linked-data\/cube#dataSet> <http:\/\/lod2.eu\/schemas\/rdfh-inst#ds>;/g' lineorder.ttl > lineorder_qb.ttl

#### Generierung der Levels in der Turtle-Notation
Hierzu werden die zuvor generierten Dateien *customer.ttl*, *dates.ttl*, *part.ttl* und *supplier.ttl* in Open Virtuoso geladen (s. [Bulk-Import-Prozedur](https://github.com/sjelsch/etl-evaluation/tree/master/open_virtuoso/bulk_import.txt)). Die verwendeten SPARQL-Insert-Befehle sind im Ordner [open_virtuoso](https://github.com/sjelsch/etl-evaluation/tree/master/open_virtuoso) aufgelistet.

**Befehle**
> /usr/local/virtuoso-opensource/bin/isql 1111 dba dba < ~/etl-evaluation/open_virtuoso/insert_customer_levels.rq
> /usr/local/virtuoso-opensource/bin/isql 1111 dba dba < ~/etl-evaluation/open_virtuoso/insert_date_levels.rq
> /usr/local/virtuoso-opensource/bin/isql 1111 dba dba < ~/etl-evaluation/open_virtuoso/insert_part_levels.rq
> /usr/local/virtuoso-opensource/bin/isql 1111 dba dba < ~/etl-evaluation/open_virtuoso/insert_supplier_levels.rq
>
> dump_one_graph('http://lod2.eu/schemas/rdfh#ssb1_ttl_levels', './data_', 1000000000);

#### Ausführung des ETL-Prozesses
Vor der Ausfürung der ETL-Prozesses muss die Datei [config.properties](https://github.com/sjelsch/etl-evaluation/blob/master/config.properties) angepasst werden, z.B. sind hier die JDBC-Konfigurationen für die Verbindung zu Apache Hive zu definieren. Ferner werden in dieser Konfigurationsdatei einige Variablen definiert die den ETL-Prozess steuern.

**Befehl**
> sh etl_process.sh

### Beschreibung der OLAP-Abfragen

Die Evaluation ist an die Arbeit "No Size Fits All – Running the Star Schema Benchmark with SPARQL and RDF Aggregate Views" [2] von Kämpgen und Harth angelehnt. Eine genaue Beschreibung der ebenfalls in dieser Evaluation verwendeten OLAP-Abfragen ist in der dazugehörigen [Projektseite](http://people.aifb.kit.edu/bka/ssb-benchmark/#ssb-queries-as-olap-queries) zu finden.

### Evaluation: Aufwand und Kosten
In diesem Abschnitt werden die benötigten Stunden und die dadurch entstandenen Kosten zur Durchführung der Evaluation bei Amazon Web Services (AWS) aufgelistet.

| EC2-Instanz-Typ        | Kosten pro Stunde | Anzahl Stunden  | Kosten Gesamt | Infos |
| ------------- |-------------------:|-----:|-----:|-----|
| m4.large | $0.199  | 162 | $32.24 | Monitoring-Server |
| m4.xlarge | $0.338 | 508 | $171.70 | Master- und DataNodes |

Bei den dargestellten Preisen handelt es sich um Nettopreise. Zusätzlich wurde bei der Evaluation 505.81 GB-Monate für den Transfer der Daten zwischen den DataNodes benötigt. Dadurch erhöht sich der Gesamt-Nettopreis nochmals um $55.64. Insgesamt hatte die Evaluation einen Gesamtbetrag von $259.58.

### Literaturverzeichnis
[1] O’Neil, P., O’Neil, E., Chen, X.: Star Schema Benchmark - Revision 3. Tech. rep., UMass/Boston (2009). ([pdf](http://www.cs.umb.edu/~poneil/StarSchemaB.pdf))

[2] Benedikt Kämpgen, Andreas Harth. No Size Fits All – Running the Star Schema Benchmark with SPARQL and RDF Aggregate Views. ESWC 2013, LNCS 7882, Seiten: 290-304, Springer, Heidelberg, Mai, 2013. ([Projektseite](http://people.aifb.kit.edu/bka/ssb-benchmark/))
