
Übersicht
=========

Der ``DeepGreen Prototyp`` ist grundsätzlich als push-forward-System konzipiert.
Es werden von Daten-Lieferanten Inhalte in Form von Metadaten und Volltexten 
an das System geliefert, diese dann von DeepGreen aufgenommen, analysiert, 
zugeordnet, und schließlich bereitgestellt und nach einer einstellbaren 
Karenzzeit aus dem system wieder gelöscht.  Die Schritte im Einzelnen genauer 
erklärt:


Daten-Übernahme (``notification``)
----------------------------------

:Quellcode: ``jper: service/views/webapi.py``, ``jper: service/api.py``

Die Daten-Übernahme erfolgt primär über die Web-API-Schnittstelle:

.. code-block:: bash

   $ curl -i -k -s -XPOST "https://oa-deepgreen.kobv.de/api/v1/notification?api_key=***" \
          -F "content=article.zip; type=application/zip" \
          -F "metadata=metafile.json; type=application/json"
   HTTP/1.1 202 ACCEPTED
   Server: nginx/1.10.2
   Date: Thu, 27 Oct 2016 08:38:35 GMT
   Content-Type: application/json
   Content-Length: 160
   Connection: keep-alive
   Location: http://oa-deepgreen.kobv.de/api/v1/notification/a2121f6647534dd19e91479838b5a5e7
   Set-Cookie: session="470tQ50scr8QvJrSGlBggSetNr4=?_fresh=STAxCi4=&_id=UydkNTA3Y2Y4YzZjMjM3ZWNhZWVhNDFlNGI5Y2Q4Y2M0NicKcDEKLg==\
                                                 &user_id=VjkyNDY4MjAzOWNjZjRlNzBhNTcwN2JjZTQxYTFmZGRlCnAxCi4="; Path=/; HttpOnly

  {"status": "accepted", "id": "a2121f6647534dd19e91479838b5a5e7", \
  "location": "http://oa-deepgreen.kobv.de/api/v1/notification/a2121f6647534dd19e91479838b5a5e7"}

In diesem Beispiel ist die Übertragung der ``zip``-Datei und der ``json``-Datei 
(mit einer Format-Angabe über das ``xml`` in der ``zip``-Datei) an DeepGreen 
erfolgreich.  Die ``zip``-Datei liegt jetzt zusammen mit der extrahierten 
``xml``-Datei im Zwischenspeicher (``storage``) für die Weiterverarbeitung 
bereit.  Die Notifikation zu dieser Datenlieferung ist unter der als ``location`` 
bezeichneten Adresse abrufbar und einsehbar, und als ``unrouted`` gekennzeichnet.

Falls ein Übertragungs- oder ein interner Konvertierungsfehler in dieser Prozedur
auftritt, wird nichts gespeichert und auch nichts markiert.  Für DeepGreen hat
es somit nie eine ``zip``-Datei gegeben.  Der ``http``-Rückgabecode ist in diesem
Fall: ``HTTP/1.1 400 BAD REQUEST`` .  Zusätzlich liefert der DeepGreen-Server noch
eine genauere Fehlermeldung ``{"error": "<kurze Fehlerbeschreibung>"}`` zurück.

-----

:Quellcode: ``jper: service/scheduler.py``, ``jper-sword-in: service/sword.py``

Sowohl die sFTP-Schnittstelle, als auch die annehmende SWORDv2-Schnittstelle 
von DeepGreen setzen auch genau auf diesen dargestellten Übernahmemechanismus
mit dem Unterschied, dass der obige curl-Befehl **automatisch** umgesetzt und 
ausgelöst wird. 


Daten-Zuordnung (``match & route``)
-----------------------------------

:Quellcode: ``jper: service/routing_deepgreen.py``, ``jper: service/scheduler.py``, ``jper: service/packages.py``

Die Zuordnung der abgelieferten Verlagsdaten zu berechtigten Abnehmern verläuft über mehere
Stufen --- mit Hilfe der in der Notifikation gespeicherten Metadaten.

In periodischen Abständen prüft DeepGreen, ob als ``unrouted`` gekennzeichnete Notifikationen
im Volltextindex (``elasticsearch``) vorliegen. Wenn dies der Fall ist, wird für jede dieser
``unrouted`` Notifikation zunächst über die enthaltene ISSN und das Publikationsdatum bestimmt, 
ob der zugehörige Artikel zu einer Allianz-Lizenz gehört.  Falls ja, werden alle teilnehmenden
Einrichtungen der gefundenen Allianz-Lizenz mit ihrer EZB-Id bestimmt.  Diese Information wird 
regelmäßig (etwa jeden Monat einmal) über die EZB in Regensburg aktuallisiert.

Die so bestimmte Teilnehmerliste wird dann mit den in DeepGreen vorhandenen Repositorien-Konten 
anhand der EZB-Id abgeglichen, und eine Liste der DeepGreen-Konten erstellt, die möglicherweise
berechtigt sind, den vorliegenden Artikel in ihr Repositorium einzustellen.

Die Liste der potentiellen Empfänger wird nun anhand gewisser Trefferkriterien mit den 
vorliegenden Artikel-Metadaten abgeglichen.  Insbesondere die Affliliationsangaben im Artikel 
werden hierzu herangezogen, aber auch E-Mailadressen oder andere identifizierbare Angaben 
(``orcid``, ``ringgold``, ``grant id``, etc.), sofern in den Metadaten des Artikels vorhanden.  
Die zu prüfenden Angaben **muss** jedes Repositorien-Konto eigenständig in DeepGreen anlegen 
und pflegen.

Falls nun **eines** der Trefferkriterien eines Repositorien-Kontos mit den vorliegenden 
Metadaten des Artikels übereinstimmt, wird dies von DeepGreen protokolliert (im Volltextindex 
als ``match_prov``) und eines neue Notifikation ``routed`` angelegt.  In diesem Schritt wird 
ggf. auch die ``zip``-Datei im Zwischenspeicher (``storage``) umgepackt, falls dies laut der 
Konto-Konfiguration der Empfänger nötig ist bzw. gewünscht wird.  Die neue Notifikation erhält 
somit *alle* internen Link-Angaben zu sämtlichen benötigten ``zip``-Formate, die dann im 
Zwischenspeicher zu diesem Artikel vorhanden sind.

Falls in diesem ganzen Prozess aber keine erfolgreiche Zuordnung vorgenommen werden konnte, wird
eine Notifikation ``failed`` erzeugt.


Daten-Löschung (``delete``)
--------------------------

:Quellcode: ``jper: service/scheduler.py``, ``jper: service/models/notification.py``, ``jper: service/packages.py``

Es werden (überraschenderweise!) nur Notifikationen gelöscht.  Der Zwischenspeicher (``storage``)
wird von DeepGreen **nicht** gelöscht, sondern, wenn überhaupt, händisch vom Administrator 
gepflegt.

Abhängig davon, wie es in der Konfigurationsdatei (lokal ``jper: ./local.cfg`` bzw. defaultmäßig 
``jper: config/service.py``) angegeben ist, werden ``unrouted`` Notifikationen gelöscht, falls
der Zuordungsversuch erfolgreich war (``DETETE_ROUTED=True``) oder falls der Zuordungsversuch
nicht erfolgreich war (``DELETE_UNROUTED=True``).  Bei den entsprechend anderen Setzungen der 
beiden Schalter (``DELETE_ROUTED``, ``DELETED_UNROUTED``) wird entsprechend nicht *sofort* nach
dem Zuordnungsversuch die Notifikation gelöscht.

Darüberhinaus gibt es noch einen Löschmechanismus für ``routed`` Notifikationen, die eine
eine festgelegte Verweildauer (``SCHEDULE_KEEP_ROUTED_MONTHS=<integer>``) überschritten haben.
Auch diese Notifikationen werden aus dem Volltextindex (``elasticsearch``) von DeepGreen
entfernt.

:Bemerkung: Somit bestände, zumindest theoretisch, die Möglichkeit, ein *dark archive* zu betreiben: 
            ``DELETE_UNROUTED=False`` zur immerwährenden Vorlage für die periodische Zuordnnug, 
            ``DELETE_ROUTED=False`` für Repositorien-Konten, die erst später zu DeepGreen hinzukommen,
            und ``SCHEDULE_KEEP_ROUTED_MONTHS=1000`` zur gut 83jährigen Aufbewahrung, zum Beispiel.
