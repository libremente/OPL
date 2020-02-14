"library-olap" è il repository dal quale ottenere il codice per la messa in funzione di un semplice olap dedicato all'MLS ClavisNG (vedi)
Le informazioni conservate nel db dell'olap sono accessibile tramite le api realizzate dagli script ph presenti in questo repository.
E' sufficiente copiare gli script php e il file di configurazione .htaccess in una cartella pubblicata sul web, affinché le api per l'interrogazione dell'olap siano disponibili.
Ecco alcuni esempi che documentano la sintassi di chiamata della API. L'output è un file json

/olap/fact_tables
    restituisce informazioni sulle fact_table presenti


    /olap/<fact table name>/dimensions
		restituisce informazioni sulle dimensioni della fact_table, livelli e gerarchie
    /olap/<fact table name>/measures
		informazioni sulle misure e sui tipi di aggregazione possibili
    /olap/<fact table name>/aggregate
		(vedi sotto)


/aggregate
	restituisce le measure aggregate tramite la funzione di aggregazione di default o quella specificata, se non viene specificato altro restituisce l’aggregazione di tutte le righe della fact_table

/olap/loans/aggregate
	nel caso la fact_table preveda più misure, per default vengono restituite tutte.
	Per selezionarne una o più di una /aggregate?measure=loans|active_users



/olap/<fact_table>/cut
tramite "cut" è possibile specificare le parti della fact table da aggregare, indicando le dimensioni (nel caso esistano gerarchie, il percorso per il livello della dimensione) e i member (lo specifico valore della dimensione) da prendere come limiti per il calcolo dell’aggregazione
(approssimativamente esprime la WHERE nella query SQL che interroga la fact table)

cut=<nome dimensione>:<valore member>
eventuali altre dimensioni separate da |

olap/loans/aggregate?cut=loan_type:loan_local

olap/loans/aggregate?cut=loan_type:loan_local|consortia_id:bc

E' possibile specificare un range tramite il simbolo “-” (se il range è contiguo) o il simbolo “;” se non è contiguo. È anche possibile specificare range aperti.

olap/loans/aggregate?cut=library_id:1-120
olap/loans/aggregate?cut=library_id:23-
olap/loans/aggregate?cut=library_id:23;50

Se la dimensione ha dei livelli (per esempio, le date sono costituite dalla gerarchia anno-mese-giorno) specificare il percorso del livello con “,”
(l’ordine rispecchia l’ordine delle gerarchie dichiarato nelle proprietà della fact table)

olap/loans/aggregate?cut=date:2016,10      (per ottenere i dati di ottobre 2016)

olap/loans/aggregate?cut=date:2016,10,3    (per i dati del 3 ottobre 2016)

olap/loans/cut=date:2016,10-2017,02 	   (per i dati da ottobre 2016 a febbraio 2017)


parametro “drill-down”
Per ottenere maggiori dettagli, crea raggruppamenti in base ai dimension member delle dimension specificate (equivale al GROUP BY di SQL)

/olap/loans/aggregate?cut=date:2016&drilldown=date   (prestiti per mese nel 2016)

Per default drilldown restituisce una riga per ogni dimension member della dimension al livello gerarchico inferiore a quello indicato dal parametro cut

/olap/loans/aggregate?cut=date:2016&drilldown=date   (raggruppamento per mese, che è il livello inferiore alla dimensione dichiarata nella cut)

/olap/loans/aggregate?cut=date:2016,10&drilldown=date (in questo caso il raggruppamento avverrà per giorno)

E' anche possibile specificare il livello gerarchico della dimensione fino al quale ottenere i raggruppamenti. Anche in questo caso si partirà dal livello inferiore a quello indicato nella cut.

/olap/loans/aggregate?cut=date:2016&drilldown=date:day (raggruppa per mese a giorno)

Quando si applica il drilldown a una dimensione che non è indicata nella cut (non è esplicitato quindi un livello da cui partire) di default il drilldown verrà eseguito per il livello gerarchico massimo.

/olap/loans/aggregate?drilldown=date   (raggruppamento per anno)

È possibile effettuare il drilldown per più di una dimensione allo stesso tempo, utilizzando come separatore “|”

/olap/loans/aggregate?cut=date:2016&drilldown=date|organization_level

Nel caso in cui nella cut ci siano limiti di data a cavallo tra mesi o anni (o altre suddivisioni) è possibile includere nell drilldown (per aumentare la chiarezza) anche lo stesso livello indicato nella cut nel seguente modo:

/olap/loans/aggregate?cut=date:2016,10-2017,02&drilldown=date:year|date:month

parametro “measure”
Parametro solitamente sottinteso. Di default restituisce il valore della prima (o unica) measure dichiarata nella <fact table name>.json. È possibile dichiarare una o più misure da aggregare.

/olap/loans/aggregate?measure=loans

/olap/loans/aggregate?measure=loans|activeusers

Se non viene espressa alcuna formula di aggregazione della misura di default viene utilizzata la prima dichiarata nel <fact table name>.json (solitamente si tratta di “sum”)

Attenzione! 
Ogni "fatto" previsto nella olapSettings.json, per poter essere gestito correttamente dalla API,
deve essere corredato da un file con nome <factName>.json, che descrive il modo in cui la API
si deve comportare.
Per esempio si prenda il seguente file relativo alle accessioni eseguite da una sistema bibliotecario

{
  "dimensions": [{           // l'array "dimension" illustra l'elenco delle "dimensioni", ossia delle grandezze che,
								dal putno di vista della raccolta dei dati, fungono da variabili indipendenti
      "name": "docType",	// nome della dimensione
      "label": "tipo di documento"
    },
    {
      "name": "organization_level",  // questa dimensione è gerarchicamente ordinata: ogni organization_level,
										infatti, è composto da consortiaId e libraryId
      "levels": [{
        "name": "consortiaId"
      }, {
        "name": "libraryId"
      }],
      "label": "livello organizzativo",
      "info": "",
      "hierarchies": [{				// la api può ricevere come parametro il tipo di gerarchia da utilizzare.
										in questo caso ce n'è una sola, ma potrebbero essere molteplici
        "name": "cl",
        "order": ["consortiaId", "libraryId"]
      }]
    },
    {
      "name": "date",
      "levels": [{
        "name": "year"
      }, {
        "name": "month"
      }],
      "hierarchies": [{
        "name": "ym",
        "order": ["year", "month"]
      }]
    }
  ],
  "measures": [{				// le "misure" rappresentano le variabili dipendenti, ossia i valori che
									perlopiù, si rappresentano sull'asse delle ordinate di un sistema cartesiano
    "name": "accessions",
    "label": "numero di accessioni",
    "aggregate": "sum"			// vengono qui indicate le operazioni da eseguire sul raggruppamento di valori
									(sum, avg o count)
  }],
  "aggregates": [{
    "name": "sum",
    "label": "somma",
    "function": "sum"
  }, {
    "name": "average",
    "label": "media",
    "function": "avg"
  }, {
    "name": "count",
    "label": "conteggio",
    "function": "count"
  }],
  "mappings": [{				// mappatura che mette in corrispondenza le etichette utilizzate in questo file
									con quelle che provengono dal db OLAP
    "accessions": "accessions.value",
    "target": "accessions.target",
    "docType": "accessions.docType",
    "libraryId": "accessions.libraryId",
    "consortiaId": "accessions.consortiaId",
    "year": "accessions.year",
    "month": "accessions.month"
  }]
}

