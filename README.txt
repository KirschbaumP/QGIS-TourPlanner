TourPlanner - QGIS-Plugin zur Tourenplanung
===========================================

BESCHREIBUNG
------------
TourPlanner ist ein generisches QGIS-Plugin zur Touren- und Routenplanung
(Vehicle Routing Problem, VRP). Das Plugin enthält selbst keinen
Optimierungsalgorithmus. Es stellt stattdessen die Infrastruktur bereit, um
beliebige Planungsalgorithmen als Python-Skript einzubinden, auszuführen und
miteinander zu vergleichen:

  * Geodaten (Depots, Fahrzeuge, Transportaufträge) werden direkt aus
    QGIS-Layern übernommen.
  * Fahrzeiten und Distanzen werden über einen Valhalla-Routing-Dienst auf
    dem realen Straßennetz berechnet und als Kostenmatrizen aufbereitet.
  * Ein frei wählbares Python-Skript berechnet auf dieser Grundlage die
    Touren. Algorithmus und Anwendungslogik sind dadurch strikt getrennt:
    Verfahren (z. B. Heuristiken, Metaheuristiken oder Constraint-Solver)
    lassen sich austauschen, erweitern und evaluieren, ohne das Plugin zu
    verändern.
  * Das Ergebnis wird als Linien-Layer wieder in QGIS dargestellt.

Die Architektur ist bewusst generisch gehalten, sodass unterschiedliche
Varianten des Tourenplanungsproblems abgebildet werden können. Entwickelt
wurde das Plugin im Rahmen einer Masterarbeit zur Optimierung von Abläufen im
Krankentransport (Vehicle Routing Problem with Pickups and Deliveries and Time
Windows, VRPPDTW).


FUNKTIONSWEISE
--------------
Der Ablauf gliedert sich in drei Schritte:

1. Eingaben (Tab "Eigenschaften")
   Auswahl der Layer, des Skripts und der Konfiguration (siehe unten).
   Mit "Eingaben übernehmen" startet die Vorverarbeitung.

2. Vorverarbeitung
   - Die QGIS-Layer werden in (Geo)DataFrames umgewandelt.
   - Für jeden Transport werden Fahrzeit und Distanz von Start- zu Zielpunkt
     berechnet (Spalten "time_sec" und "distance_km").
   - Über Valhalla (Costing "auto") werden drei Kostenmatrizen
     (Fahrzeiten in Sekunden) berechnet:
       * Depots -> Startpunkte der Touren
       * Endpunkte der Touren -> Depots
       * Endpunkt einer Tour -> Startpunkt jeder anderen Tour
   - Im Tab "Eingabe-Daten" wird die Anzahl der geladenen Depots, Fahrzeuge
     und Touren angezeigt.

3. Ausführung
   Mit "Starten" wird das gewählte Skript ausgeführt. Das zurückgegebene
   Ergebnis wird als temporärer Layer "Touren" zum QGIS-Projekt hinzugefügt.


EINGABEPARAMETER
----------------
  Depots               Punkt-Layer mit den Depotstandorten (Start und/oder
                       Ende der Fahrzeugrouten). Benötigt eine Spalte "id".
  Fahrzeuge            Nicht-räumliche Attributtabelle der Fahrzeugflotte
                       (z. B. Schichtzeiten, Pausen, zugeordnetes Depot).
                       Weitere Spalten werden unverändert an das Skript
                       übergeben.
  Touren               Linien-Layer der Transportaufträge. Jedes Feature ist
                       eine Beförderung von einem Start- zu einem Zielort
                       inklusive Auftragsattributen (z. B. Zeitfenster).
                       Benötigt eine Spalte "id".
  Skript               Pfad zum Python-Skript mit der Planungslogik.
  Python-Umgebung      Pfad zum Python-Interpreter, mit dem das Skript
                       ausgeführt wird (z. B. eine conda- oder venv-
                       Umgebung mit den benötigten Solver-Bibliotheken).
  Ausgabe-Ordner       Zielverzeichnis für die Debug-Ausgabe.
  Valhalla-URL         Adresse des Valhalla-Routing-Dienstes
                       (Vorgabe: http://localhost:14000).
  Skript-Parameter     Beliebige Schlüssel/Wert-Paare mit Datentyp
                       (String, Integer, Float, Boolean), z. B. Zeitlimit oder
                       Gewichtungen. Sie werden dem Skript als
                       "extra_params" übergeben.
  Debug-Ausgabe        Speichert die aufbereiteten Eingabedaten (depots.shp,
                       vehicles.csv, tours.shp sowie die drei Matrizen als
                       CSV) im Ausgabe-Ordner. Dadurch lassen sich Skripte
                       auch außerhalb von QGIS entwickeln und testen.


SCHNITTSTELLE FUER PLANUNGSSKRIPTE
----------------------------------
Das Skript wird im ausgewählten Python-Interpreter als eigener Prozess
gestartet, sodass es unabhängig von der QGIS-Python-Umgebung laufen kann:

    <python-umgebung> <skript.py> <eingabe-datei> <ausgabe-datei>

Die Eingabedatei ist ein Pickle-Objekt (Dictionary) mit den Schlüsseln:

    depots                 GeoDataFrame   Depotstandorte
    vehicles               DataFrame      Fahrzeugflotte
    tours                  GeoDataFrame   Transportaufträge inkl.
                                          "time_sec" und "distance_km"
    matrix_depots_starts   DataFrame      Kosten Depots -> Tourstarts
    matrix_ends_depots     DataFrame      Kosten Tourenden -> Depots
    matrix_tours_to_tours  DataFrame      Kosten Tourende -> Tourstart
                                          (Spalten: start_tour_id,
                                          end_tour_id, time, distance)
    valhalla_url           str            Adresse des Routing-Dienstes
    extra_params           dict           Skript-Parameter aus dem Dialog

Das Skript schreibt sein Ergebnis als Geodatei mit Liniengeometrien in die
übergebene Ausgabedatei. Diese wird mit GeoPandas eingelesen und als Layer in
QGIS dargestellt. Fehler des Skripts (stderr) werden in einem Dialog angezeigt.


VORAUSSETZUNGEN
---------------
  * QGIS 3.x (metadata.txt: 3.0 bis 4.99)
  * Python-Pakete in der QGIS-Python-Umgebung: numpy, pandas, geopandas,
    requests
  * Ein erreichbarer Valhalla-Routing-Dienst mit Karten für das
    Untersuchungsgebiet
  * Eine Python-Umgebung mit den für das Skript benötigten Bibliotheken
    (z. B. OR-Tools)


INSTALLATION
------------
Den Plugin-Ordner in das QGIS-Plugin-Verzeichnis kopieren (bzw. verlinken)
und das Plugin in QGIS unter "Erweiterungen > Erweiterungen verwalten und
installieren" aktivieren. Das Plugin ist anschließend über die Werkzeugleiste
verfügbar.


PROJEKTSTRUKTUR
---------------
  tour_planner.py             Plugin-Logik (Dialog, Vorverarbeitung, Ausführung)
  tour_planner_dialog.py      Dialog-Klasse
  tour_planner_dialog_base.ui Oberfläche (Qt Designer)
  metadata.txt                Plugin-Metadaten für QGIS

LINKS
-----
  Repository:  https://github.com/KirschbaumP/QGIS-TourPlanner
  Fehlermeldungen: https://github.com/KirschbaumP/QGIS-TourPlanner/issues


AUTOR UND LIZENZ
----------------
Philipp von Kirschbaum
Lizenz: MIT License
