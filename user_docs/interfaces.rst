
Benutzung der Schnittstellen
============================

Im folgenden werden die in DeepGreen realisierten Schnittstellen ``REST Web-API``, 
``SWORDv2``, ``sFTP`` und ``OAI-PMH`` zur Datenannahme bzw. -abgabe genauer erklärt.


Web-API
-------

.. highlight:: bash

DeepGreen bietet eine (*rudimentäre?*)
`REST-Schnittstelle <https://de.wikipedia.org/wiki/Representational_State_Transfer>`_ an.
Die aktuelle Version der DeepGreen REST API ist ``v1``. Diese kann über die Web-Adresse 
``https://oa-deepgreen.kobv.de/api/v1`` angesprochen werden, z.B. mit dem Kommando
`curl <https://curl.haxx.se>`_ in einer Shell (``bash``, ``tcsh``, etc.)::
 
  $ curl -k -s [-X GET|POST|PUT|HEADER] https://oa-deepgreen.kobv.de/api/v1/...

Dabei werden in den nun nachfolgende Beispielen die ``...`` ersetzt durch die konkreten
REST-Resourcen ``validate``, ``notification`` und ``routed`` der Web-API-Schnittstelle.

Datenannahme
~~~~~~~~~~~~

Vor der Übertragung eines Datenpakets einer Publikation zu DeepGreen kann das Paket von 
der Datendrehscheibe überprüft werden. Es sind also zwei REST-Resourcen des Web-APIs verfügbar,

1. Validierung eines Datenpakets (``validation``),
2. Übertragung eines Datenpakets (``notification``).

Für ganz ungeduldige Leser hier zwei (funktionierende!) ``bash``-Beispielskripte, die 
``zip``-Pakete mit einem ``api_key`` an DeepGreen liefert bzw. validiert.  Die Skripte
analysieren zunächst jeweils die angegeben ``zip``-Datei, welches Metadatenschema im
Datenpaket vorliegt.  Momentan werden von DeepGreen zwei Schemata verarbeitet, 
``DTD JATS`` und ``DTD RSC``.

.. code-block:: bash
   :caption: bash-Skript zur Validierung

   #! /usr/bin/env bash

   host_url="https://oa-deepgreen.kobv.de"

   if [ $# -ge 2 ]; then
     api_key=$1
     zip_file=$2
   else
     echo "usage: `basename $0` {api-key} {zip-file}"
     exit -1
   fi

   curl=`which curl`
   zipgrep=`which zipgrep`
   wc=`which wc`

   pkg_fmt="https://datahub.deepgreen.org/FilesAndJATS"
   has_xml=`${zipgrep} "DOCTYPE article" ${zip_file} | ${wc} -l`

   if [ ${has_xml} -eq 1 ]; then
     is_jats=`${zipgrep} "//NLM//DTD JATS " ${zip_file} | ${wc} -l`
     is_rsc=`${zipgrep} "//RSC//DTD RSC " ${zip_file} | ${wc} -l`
     if [ ${is_jats} -eq 1 ]; then
       pkg_fmt="https://datahub.deepgreen.org/FilesAndJATS"
     elif [ ${is_rsc} -eq 1 ]; then
       pkg_fmt="https://datahub.deepgreen.org/FilesAndRSC"
     else
       echo "error: no valid .xml (JATS or RSC) in zip archive found: stop."
       exit -2
     fi
   else
     echo "error: no valid (or too many?!) .xml (JATS xor RSC) in zip archive found: stop."
     exit -3
   fi

   echo "`basename $0`: packaging format in zip archive found:"
   echo "`basename $0`: ${pkg_fmt}"

   ${curl} -i -k -s -X POST  "${host_url}/api/v1/validate?api_key=${api_key}" -F "content=@${zip_file};type=application/zip" -F "metadata=@-;type=application/json" <<EOF
   {
     "content" : {
        "packaging_format" : "${pkg_fmt}"
     }
   }
   EOF

   echo

.. code-block:: bash
   :caption: bash-Skript zur Datenpaketübertragung

   #! /usr/bin/env bash

   host_url="https://oa-deepgreen.kobv.de"

   if [ $# -ge 2 ]; then
     api_key=$1
     zip_file=$2
   else
     echo "usage: `basename $0` {api-key} {zip-file}"
     exit -1
   fi

   curl=`which curl`
   zipgrep=`which zipgrep`
   wc=`which wc`

   pkg_fmt="https://datahub.deepgreen.org/FilesAndJATS"
   has_xml=`${zipgrep} "DOCTYPE article" ${zip_file} | ${wc} -l`

   if [ ${has_xml} -eq 1 ]; then
     is_jats=`${zipgrep} "//NLM//DTD JATS " ${zip_file} | ${wc} -l`
     is_rsc=`${zipgrep} "//RSC//DTD RSC " ${zip_file} | ${wc} -l`
     if [ ${is_jats} -eq 1 ]; then
       pkg_fmt="https://datahub.deepgreen.org/FilesAndJATS"
     elif [ ${is_rsc} -eq 1 ]; then
       pkg_fmt="https://datahub.deepgreen.org/FilesAndRSC"
     else
       echo "error: no valid .xml (JATS or RSC) in zip archive found: stop."
       exit -2
     fi
   else
     echo "error: no valid (or too many?!) .xml (JATS xor RSC) in zip archive found: stop."
     exit -3
   fi

   echo "`basename $0`: packaging format in zip archive found:"
   echo "`basename $0`: ${pkg_fmt}"

   ${curl} -i -k -s -X POST  "${host_url}/api/v1/notification?api_key=${api_key}" -F "content=@${zip_file};type=application/zip" -F "metadata=@-;type=application/json" <<EOF
   {
       "content" : {
           "packaging_format" : "${pkg_fmt}"
       }
   }
   EOF

   echo



Nur Metadaten (Beispiel ``validation``)
```````````````````````````````````````

Für die Verifikation einer Datenlieferung macht es durchaus Sinn, nur die Metadaten der 
Lieferung zu schicken.  Dazu wird das ``json``-Format "``Incoming Notification JSON``" 
zur Beschreibung der zu überprüfenden Metadaten verwendet.  Mit Binärdaten gebündelte
Lieferungen können natürlich ebenso geprüft werden.  Dies funktioniert genau wie im 
Beispiel ``notification`` unten angegeben.

.. code-block:: rest
   :caption: Angabe von Metadaten durch "``Incoming Notification JSON``" (internes DeepGreen-Format)

   POST /validate?api_key=<api_key>
   Content-Type: application/json

   [Incoming Notification JSON]



* **http-Header Rückgabewerte**

   +----------------------+-------------------------------------------------------------------+
   | Code                 | Beschreibung                                                      |
   +======================+===================================================================+
   | \                    | \                                                                 |
   |                      |                                                                   |
   | **204** No Content   | Datenpaket ok!                                                    |
   |                      |                                                                   |
   +----------------------+-------------------------------------------------------------------+
   | \                    | .. code-block:: rest                                              |
   |                      |                                                                   |
   | **400** Bad Request  |    HTTP 1.1  400 Bad Request                                      |
   |                      |    Content-Type: application/json                                 |
   |                      |                                                                   |
   |                      |    {                                                              |
   |                      |      "error" : "<verständliche(?) Fehlermeldung (auf englisch)>"  |
   |                      |    }                                                              |
   +----------------------+-------------------------------------------------------------------+
   | \                    | \                                                                 |
   |                      |                                                                   |
   | **401** Unauthorised | z.B. ungültiger ``api_key``, falsche Benutzertyp                  |
   |                      |                                                                   |
   +----------------------+-------------------------------------------------------------------+



Metadaten und Paket (Beispiel ``notification``)
```````````````````````````````````````````````

Artikellieferungen, die binären Inhalt enthalten sollen (z.B. der Volltext als ``pdf``), 
werden durch die Kennzeichnung "``multipart/form-data``" im http-Header gebündelt.  Dabei 
**muss** im ``json`` des Metadatenteil ("``Incoming Notifikation JSON``") das Feld 
**content.packaging_format** zwingend vorhanden sein. 

.. code-block:: rest
   :caption: http-POST: Bündelung durch "``Content-Type: multipart/form-data; ...``"

   POST /notification?api_key=<api_key>
   Content-Type: multipart/form-data; boundary=FulltextBoundary

   --FulltextBoundary

   Content-Disposition: form-data; name="metadata"
   Content-Type: application/json

   [Incoming Notification JSON]

   --FulltextBoundary

   Content-Disposition: form-data; name="content"
   Content-Type: application/zip

   [Package]

   --FulltextBoundary--


.. code-block:: rest
   :caption: Minimale Angabe der Metadaten durch "``packaging_format``"

   POST /notification?api_key=<api_key>
   Content-Type: multipart/form-data; boundary=FulltextBoundary

   --FulltextBoundary

   Content-Disposition: form-data; name="metadata"
   Content-Type: application/json

   {
       "content" : {
           "packaging_format" : "https://datahub.deepgreen.org/FilesAndJATS"
       }
   }

   --FulltextBoundary

   Content-Disposition: form-data; name="content"
   Content-Type: application/zip

   [Package]

   --FulltextBoundary--


* **http-Header Rückgabewerte**

   +----------------------+-------------------------------------------------------------------+
   | Code                 | Beschreibung                                                      |
   +======================+===================================================================+
   | \                    | .. code-block:: rest                                              |
   |                      |                                                                   |
   | **202** Accepted     |    HTTP 1.1  202 Accepted                                         |
   |                      |    Content-Type: application/json                                 |
   |                      |    Location: <URL des api-Endpunkts der akzeptierten Lieferung>   |
   |                      |                                                                   |
   |                      |    {                                                              |
   |                      |      "status" : "accepted",                                       |
   |                      |      "id" : "<eindeutige ID dieser neuen Notifikation>",          |
   |                      |      "location" : "<URL des api-Endpunkts dieser Notifikation>"   |
   |                      |    }                                                              |
   +----------------------+-------------------------------------------------------------------+
   | \                    | .. code-block:: rest                                              |
   |                      |                                                                   |
   | **400** Bad Request  |    HTTP 1.1  400 Bad Request                                      |
   |                      |    Content-Type: application/json                                 |
   |                      |                                                                   |
   |                      |    {                                                              |
   |                      |      "error" : "<verständliche(?) Fehlermeldung (auf englisch)>"  |
   |                      |    }                                                              |
   +----------------------+-------------------------------------------------------------------+
   | \                    | \                                                                 |
   |                      |                                                                   |
   | **401** Unauthorised | z.B. ungültiger ``api_key``, falscher Benutzertyp                 |
   |                      |                                                                   |
   +----------------------+-------------------------------------------------------------------+




Datenabgabe
~~~~~~~~~~~

.. code-block:: rest
   :caption: Liste aller erfolgreich zugestellten Notifikationen

   GET /routed[?<params>]

.. code-block:: rest
   :caption: Liste der zugestellten Notifikationen **einer** Einrichtung

   GET /routed/<repo_id>[?<params>]

.. code-block:: rest
   :caption: Eine bestimmte Notification, geliefert im Format "``Outgoing Notification JSON``" (internes DeepGreen-Format)
  
   GET /notification/<notification_id>



SWORD
-----


sFTP
----


OAI-PMH
-------

