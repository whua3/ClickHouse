---
machine_translated: true
machine_translated_rev: f865c9653f9df092694258e0ccdd733c339112f5
toc_priority: 19
toc_title: Interface HTTP
---

# Interface HTTP {#http-interface}

L'interface HTTP vous permet D'utiliser ClickHouse sur n'importe quelle plate-forme à partir de n'importe quel langage de programmation. Nous l'utilisons pour travailler à partir de Java et Perl, ainsi que des scripts shell. Dans d'autres départements, L'interface HTTP est utilisée à partir de Perl, Python et Go. L'interface HTTP est plus limitée que l'interface native, mais elle a une meilleure compatibilité.

Par défaut, clickhouse-server écoute HTTP sur le port 8123 (cela peut être modifié dans la configuration).

Si vous faites une requête GET / sans Paramètres, elle renvoie le code de réponse 200 et la chaîne définie dans [http\_server\_default\_response](../operations/server_configuration_parameters/settings.md#server_configuration_parameters-http_server_default_response) valeur par défaut “Ok.” (avec un saut de ligne à la fin)

``` bash
$ curl 'http://localhost:8123/'
Ok.
```

Utilisez la requête GET /ping dans les scripts de vérification de la santé. Ce gestionnaire revient toujours “Ok.” (avec un saut de ligne à la fin). Disponible à partir de la version 18.12.13.

``` bash
$ curl 'http://localhost:8123/ping'
Ok.
```

Envoyer la demande sous forme D'URL ‘query’ paramètre, ou comme un POSTE. Ou envoyer le début de la requête dans l' ‘query’ paramètre, et le reste dans le POST (nous expliquerons plus tard pourquoi cela est nécessaire). La taille de L'URL est limitée à 16 Ko, alors gardez cela à l'esprit lors de l'envoi de requêtes volumineuses.

En cas de succès, vous recevez le code de réponse 200 et le résultat dans le corps de réponse.
Si une erreur se produit, vous recevez le code de réponse 500 et un texte de description de l'erreur dans le corps de la réponse.

Lorsque vous utilisez la méthode GET, ‘readonly’ est définie. En d'autres termes, pour les requêtes qui modifient les données, vous ne pouvez utiliser que la méthode POST. Vous pouvez envoyer la requête elle-même dans le corps du message ou dans le paramètre URL.

Exemple:

``` bash
$ curl 'http://localhost:8123/?query=SELECT%201'
1

$ wget -O- -q 'http://localhost:8123/?query=SELECT 1'
1

$ echo -ne 'GET /?query=SELECT%201 HTTP/1.0\r\n\r\n' | nc localhost 8123
HTTP/1.0 200 OK
Date: Wed, 27 Nov 2019 10:30:18 GMT
Connection: Close
Content-Type: text/tab-separated-values; charset=UTF-8
X-ClickHouse-Server-Display-Name: clickhouse.ru-central1.internal
X-ClickHouse-Query-Id: 5abe861c-239c-467f-b955-8a201abb8b7f
X-ClickHouse-Summary: {"read_rows":"0","read_bytes":"0","written_rows":"0","written_bytes":"0","total_rows_to_read":"0"}

1
```

Comme vous pouvez le voir, curl est un peu gênant en ce sens que les espaces doivent être échappés URL.
Bien que wget échappe à tout lui-même, nous ne recommandons pas de l'utiliser car il ne fonctionne pas bien sur HTTP 1.1 lors de l'utilisation de keep-alive et Transfer-Encoding: chunked.

``` bash
$ echo 'SELECT 1' | curl 'http://localhost:8123/' --data-binary @-
1

$ echo 'SELECT 1' | curl 'http://localhost:8123/?query=' --data-binary @-
1

$ echo '1' | curl 'http://localhost:8123/?query=SELECT' --data-binary @-
1
```

Si une partie de la requête est envoyée dans le paramètre et une partie dans la publication, un saut de ligne est inséré entre ces deux parties de données.
Exemple (cela ne fonctionnera pas):

``` bash
$ echo 'ECT 1' | curl 'http://localhost:8123/?query=SEL' --data-binary @-
Code: 59, e.displayText() = DB::Exception: Syntax error: failed at position 0: SEL
ECT 1
, expected One of: SHOW TABLES, SHOW DATABASES, SELECT, INSERT, CREATE, ATTACH, RENAME, DROP, DETACH, USE, SET, OPTIMIZE., e.what() = DB::Exception
```

Par défaut, les données sont renvoyées au format TabSeparated (pour plus d'informations, voir “Formats” section).
Vous utilisez la clause FORMAT de la requête pour demander tout autre format.

``` bash
$ echo 'SELECT 1 FORMAT Pretty' | curl 'http://localhost:8123/?' --data-binary @-
┏━━━┓
┃ 1 ┃
┡━━━┩
│ 1 │
└───┘
```

La méthode POST de transmission des données est nécessaire pour les requêtes INSERT. Dans ce cas, vous pouvez écrire le début de la requête dans le paramètre URL et utiliser POST pour transmettre les données à insérer. Les données à insérer pourraient être, par exemple, un vidage séparé par tabulation de MySQL. De cette façon, la requête INSERT remplace LOAD DATA LOCAL INFILE de MySQL.

Exemples: création d'une table:

``` bash
$ echo 'CREATE TABLE t (a UInt8) ENGINE = Memory' | curl 'http://localhost:8123/' --data-binary @-
```

Utilisation de la requête D'insertion familière pour l'insertion de données:

``` bash
$ echo 'INSERT INTO t VALUES (1),(2),(3)' | curl 'http://localhost:8123/' --data-binary @-
```

Les données peuvent être envoyées séparément de la requête:

``` bash
$ echo '(4),(5),(6)' | curl 'http://localhost:8123/?query=INSERT%20INTO%20t%20VALUES' --data-binary @-
```

Vous pouvez spécifier n'importe quel format de données. Le ‘Values’ le format est le même que ce qui est utilisé lors de L'écriture INSERT dans les valeurs t:

``` bash
$ echo '(7),(8),(9)' | curl 'http://localhost:8123/?query=INSERT%20INTO%20t%20FORMAT%20Values' --data-binary @-
```

Pour insérer des données à partir d'un vidage séparé par des tabulations, spécifiez le format correspondant:

``` bash
$ echo -ne '10\n11\n12\n' | curl 'http://localhost:8123/?query=INSERT%20INTO%20t%20FORMAT%20TabSeparated' --data-binary @-
```

La lecture de la table des matières. Les données sont sorties dans un ordre aléatoire en raison d'un traitement de requête parallèle:

``` bash
$ curl 'http://localhost:8123/?query=SELECT%20a%20FROM%20t'
7
8
9
10
11
12
1
2
3
4
5
6
```

Suppression de la table.

``` bash
$ echo 'DROP TABLE t' | curl 'http://localhost:8123/' --data-binary @-
```

Pour les requêtes réussies qui ne renvoient pas de table de données, un corps de réponse vide est renvoyé.

Vous pouvez utiliser le format de compression ClickHouse interne lors de la transmission de données. Les données compressées ont un format non standard, et vous devrez utiliser le spécial `clickhouse-compressor` programme de travail (il s'est installé avec le `clickhouse-client` paquet). Pour augmenter l'efficacité de l'insertion de données, vous pouvez désactiver la vérification de la somme de contrôle côté serveur en utilisant [http\_native\_compression\_disable\_checksumming\_on\_decompress](../operations/settings/settings.md#settings-http_native_compression_disable_checksumming_on_decompress) paramètre.

Si vous avez spécifié `compress=1` dans l'URL, le serveur compresse les données qu'il vous envoie.
Si vous avez spécifié `decompress=1` dans L'URL, le serveur décompresse les mêmes données que vous transmettez `POST` méthode.

Vous pouvez également choisir d'utiliser [La compression HTTP](https://en.wikipedia.org/wiki/HTTP_compression). Pour envoyer un compressé `POST` demande, ajouter l'en-tête de requête `Content-Encoding: compression_method`. Pour que ClickHouse compresse la réponse, vous devez ajouter `Accept-Encoding: compression_method`. Supports ClickHouse `gzip`, `br`, et `deflate` [méthodes de compression](https://en.wikipedia.org/wiki/HTTP_compression#Content-Encoding_tokens). Pour activer la compression HTTP, vous devez utiliser le ClickHouse [enable\_http\_compression](../operations/settings/settings.md#settings-enable_http_compression) paramètre. Vous pouvez configurer le niveau de compression des données dans le [http\_zlib\_compression\_level](#settings-http_zlib_compression_level) pour toutes les méthodes de compression.

Vous pouvez l'utiliser pour réduire le trafic réseau lors de la transmission d'une grande quantité de données, ou pour créer des vidages qui sont immédiatement compressés.

Exemples d'envoi de données avec compression:

``` bash
#Sending data to the server:
$ curl -vsS "http://localhost:8123/?enable_http_compression=1" -d 'SELECT number FROM system.numbers LIMIT 10' -H 'Accept-Encoding: gzip'

#Sending data to the client:
$ echo "SELECT 1" | gzip -c | curl -sS --data-binary @- -H 'Content-Encoding: gzip' 'http://localhost:8123/'
```

!!! note "Note"
    Certains clients HTTP peuvent décompresser les données du serveur par défaut (avec `gzip` et `deflate`) et vous pouvez obtenir des données décompressées même si vous utilisez les paramètres de compression correctement.

Vous pouvez utiliser l' ‘database’ Paramètre URL pour spécifier la base de données par défaut.

``` bash
$ echo 'SELECT number FROM numbers LIMIT 10' | curl 'http://localhost:8123/?database=system' --data-binary @-
0
1
2
3
4
5
6
7
8
9
```

Par défaut, la base de données enregistrée dans les paramètres du serveur est utilisée comme base de données par défaut. Par défaut, c'est la base de données appelée ‘default’. Alternativement, vous pouvez toujours spécifier la base de données en utilisant un point avant le nom de la table.

Le nom d'utilisateur et le mot de passe peuvent être indiqués de l'une des trois façons suivantes:

1.  Utilisation de L'authentification de base HTTP. Exemple:

<!-- -->

``` bash
$ echo 'SELECT 1' | curl 'http://user:password@localhost:8123/' -d @-
```

1.  Dans le ‘user’ et ‘password’ Les paramètres d'URL. Exemple:

<!-- -->

``` bash
$ echo 'SELECT 1' | curl 'http://localhost:8123/?user=user&password=password' -d @-
```

1.  Utiliser ‘X-ClickHouse-User’ et ‘X-ClickHouse-Key’ tête. Exemple:

<!-- -->

``` bash
$ echo 'SELECT 1' | curl -H 'X-ClickHouse-User: user' -H 'X-ClickHouse-Key: password' 'http://localhost:8123/' -d @-
```

Si le nom d'utilisateur n'est spécifié, le `default` le nom est utilisé. Si le mot de passe n'est spécifié, le mot de passe vide est utilisé.
Vous pouvez également utiliser les paramètres D'URL pour spécifier des paramètres pour le traitement d'une seule requête ou de profils entiers de paramètres. Exemple: http: / / localhost: 8123/?profil = web & max\_rows\_to\_read=1000000000 & query=sélectionner + 1

Pour plus d'informations, voir le [Paramètre](../operations/settings/index.md) section.

``` bash
$ echo 'SELECT number FROM system.numbers LIMIT 10' | curl 'http://localhost:8123/?' --data-binary @-
0
1
2
3
4
5
6
7
8
9
```

Pour plus d'informations sur les autres paramètres, consultez la section “SET”.

De même, vous pouvez utiliser des sessions ClickHouse dans le protocole HTTP. Pour ce faire, vous devez ajouter l' `session_id` GET paramètre à la demande. Vous pouvez utiliser n'importe quelle chaîne comme ID de session. Par défaut, la session est terminée après 60 secondes d'inactivité. Pour modifier ce délai d'attente, de modifier la `default_session_timeout` dans la configuration du serveur, ou ajoutez le `session_timeout` GET paramètre à la demande. Pour vérifier l'état de la session, utilisez `session_check=1` paramètre. Une seule requête à la fois peut être exécutée dans une seule session.

Vous pouvez recevoir des informations sur le déroulement d'une requête en `X-ClickHouse-Progress` en-têtes de réponse. Pour ce faire, activez [send\_progress\_in\_http\_headers](../operations/settings/settings.md#settings-send_progress_in_http_headers). Exemple de l'en-tête de séquence:

``` text
X-ClickHouse-Progress: {"read_rows":"2752512","read_bytes":"240570816","total_rows_to_read":"8880128"}
X-ClickHouse-Progress: {"read_rows":"5439488","read_bytes":"482285394","total_rows_to_read":"8880128"}
X-ClickHouse-Progress: {"read_rows":"8783786","read_bytes":"819092887","total_rows_to_read":"8880128"}
```

Possibles champs d'en-tête:

-   `read_rows` — Number of rows read.
-   `read_bytes` — Volume of data read in bytes.
-   `total_rows_to_read` — Total number of rows to be read.
-   `written_rows` — Number of rows written.
-   `written_bytes` — Volume of data written in bytes.

Les requêtes en cours d'exécution ne s'arrêtent pas automatiquement si la connexion HTTP est perdue. L'analyse et le formatage des données sont effectués côté serveur et l'utilisation du réseau peut s'avérer inefficace.
Facultatif ‘query\_id’ le paramètre peut être passé comme ID de requête (n'importe quelle chaîne). Pour plus d'informations, consultez la section “Settings, replace\_running\_query”.

Facultatif ‘quota\_key’ le paramètre peut être passé comme clé de quota (n'importe quelle chaîne). Pour plus d'informations, consultez la section “Quotas”.

L'interface HTTP permet de transmettre des données externes (tables temporaires externes) pour l'interrogation. Pour plus d'informations, consultez la section “External data for query processing”.

## Tampon De Réponse {#response-buffering}

Vous pouvez activer la mise en mémoire tampon des réponses côté serveur. Le `buffer_size` et `wait_end_of_query` Les paramètres D'URL sont fournis à cette fin.

`buffer_size` détermine le nombre d'octets dans le résultat de tampon dans la mémoire du serveur. Si un corps de résultat est supérieur à ce seuil, le tampon est écrit sur le canal HTTP et les données restantes sont envoyées directement au canal HTTP.

Pour vous assurer que la réponse entière est mise en mémoire tampon, définissez `wait_end_of_query=1`. Dans ce cas, les données ne sont pas stockées dans la mémoire tampon temporaire du serveur de fichiers.

Exemple:

``` bash
$ curl -sS 'http://localhost:8123/?max_result_bytes=4000000&buffer_size=3000000&wait_end_of_query=1' -d 'SELECT toUInt8(number) FROM system.numbers LIMIT 9000000 FORMAT RowBinary'
```

Utilisez la mise en mémoire tampon pour éviter les situations où une erreur de traitement de requête s'est produite après l'envoi du code de réponse et des en-têtes HTTP au client. Dans cette situation, un message d'erreur est écrit à la fin du corps de la réponse, et du côté client, l'erreur ne peut être détectée qu'à l'étape d'analyse.

### Requêtes avec paramètres {#cli-queries-with-parameters}

Vous pouvez créer une requête avec paramètres et transmettre des valeurs des paramètres de la requête HTTP. Pour plus d'informations, voir [Requêtes avec des paramètres pour CLI](cli.md#cli-queries-with-parameters).

### Exemple {#example}

``` bash
$ curl -sS "<address>?param_id=2&param_phrase=test" -d "SELECT * FROM table WHERE int_column = {id:UInt8} and string_column = {phrase:String}"
```

## Interface HTTP prédéfinie {#predefined_http_interface}

ClickHouse prend en charge des requêtes spécifiques via L'interface HTTP. Par exemple, vous pouvez écrire des données dans un tableau comme suit:

``` bash
$ echo '(4),(5),(6)' | curl 'http://localhost:8123/?query=INSERT%20INTO%20t%20VALUES' --data-binary @-
```

ClickHouse prend également en charge L'Interface HTTP prédéfinie qui peut vous aider à une intégration plus facile avec des outils tiers tels que [Prometheus exportateur](https://github.com/percona-lab/clickhouse_exporter).

Exemple:

-   Tout d'abord, ajoutez cette section au fichier de configuration du serveur:

<!-- -->

``` xml
<http_handlers>
  <predefine_query_handler>
      <url>/metrics</url>
        <method>GET</method>
        <queries>
            <query>SELECT * FROM system.metrics LIMIT 5 FORMAT Template SETTINGS format_template_resultset = 'prometheus_template_output_format_resultset', format_template_row = 'prometheus_template_output_format_row', format_template_rows_between_delimiter = '\n'</query>
        </queries>
  </predefine_query_handler>
</http_handlers>
```

-   Vous pouvez maintenant demander l'url directement pour les données au format Prometheus:

<!-- -->

``` bash
curl -vvv 'http://localhost:8123/metrics'
*   Trying ::1...
* Connected to localhost (::1) port 8123 (#0)
> GET /metrics HTTP/1.1
> Host: localhost:8123
> User-Agent: curl/7.47.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Wed, 27 Nov 2019 08:54:25 GMT
< Connection: Keep-Alive
< Content-Type: text/plain; charset=UTF-8
< X-ClickHouse-Server-Display-Name: i-tl62qd0o
< Transfer-Encoding: chunked
< X-ClickHouse-Query-Id: f39235f6-6ed7-488c-ae07-c7ceafb960f6
< Keep-Alive: timeout=3
< X-ClickHouse-Summary: {"read_rows":"0","read_bytes":"0","written_rows":"0","written_bytes":"0","total_rows_to_read":"0"}
<
# HELP "Query" "Number of executing queries"
# TYPE "Query" counter
"Query" 1

# HELP "Merge" "Number of executing background merges"
# TYPE "Merge" counter
"Merge" 0

# HELP "PartMutation" "Number of mutations (ALTER DELETE/UPDATE)"
# TYPE "PartMutation" counter
"PartMutation" 0

# HELP "ReplicatedFetch" "Number of data parts being fetched from replica"
# TYPE "ReplicatedFetch" counter
"ReplicatedFetch" 0

# HELP "ReplicatedSend" "Number of data parts being sent to replicas"
# TYPE "ReplicatedSend" counter
"ReplicatedSend" 0

* Connection #0 to host localhost left intact
```

Comme vous pouvez le voir dans l'exemple, si `<http_handlers>` est configuré dans la configuration.fichier xml, ClickHouse fera correspondre les requêtes HTTP reçues au type prédéfini dans `<http_handlers>`, puis ClickHouse exécutera la requête prédéfinie correspondante si la correspondance est réussie.

Maintenant `<http_handlers>` pouvez configurer `<root_handler>`, `<ping_handler>`, `<replicas_status_handler>`, `<dynamic_query_handler>` et `<no_handler_description>` .

## root\_handler {#root_handler}

`<root_handler>` renvoie le contenu spécifié pour la requête de chemin racine. Le contenu de retour spécifique est configuré par `http_server_default_response` dans la configuration.XML. si non spécifié, le retour **OK.**

`http_server_default_response` n'est pas défini et une requête HTTP est envoyée à ClickHouse. Le résultat est comme suit:

``` xml
<http_handlers>
    <root_handler/>
</http_handlers>
```

    $ curl 'http://localhost:8123'
    Ok.

`http_server_default_response` est défini et une requête HTTP est envoyée à ClickHouse. Le résultat est comme suit:

``` xml
<http_server_default_response><![CDATA[<html ng-app="SMI2"><head><base href="http://ui.tabix.io/"></head><body><div ui-view="" class="content-ui"></div><script src="http://loader.tabix.io/master.js"></script></body></html>]]></http_server_default_response>

<http_handlers>
    <root_handler/>
</http_handlers>
```

    $ curl 'http://localhost:8123'
    <html ng-app="SMI2"><head><base href="http://ui.tabix.io/"></head><body><div ui-view="" class="content-ui"></div><script src="http://loader.tabix.io/master.js"></script></body></html>%

## ping\_handler {#ping_handler}

`<ping_handler>` peut être utilisé pour sonder la santé du serveur clickhouse actuel. Lorsque le serveur HTTP ClickHouse est normal, l'accès à ClickHouse via `<ping_handler>` sera de retour **OK.**.

Exemple:

``` xml
<http_handlers>
    <ping_handler>/ping</ping_handler>
</http_handlers>
```

``` bash
$ curl 'http://localhost:8123/ping'
Ok.
```

## replicas\_status\_handler {#replicas_status_handler}

`<replicas_status_handler>` est utilisé pour détecter l'état du nœud de réplica et le retour **OK.** si le nœud réplique n'a pas de délai. S'il y a un retard, renvoyez le retard spécifique. La valeur de `<replicas_status_handler>` prend en charge la personnalisation. Si vous ne spécifiez pas `<replicas_status_handler>`, ClickHouse réglage par défaut `<replicas_status_handler>` être **/ replicas\_status**.

Exemple:

``` xml
<http_handlers>
    <replicas_status_handler>/replicas_status</replicas_status_handler>
</http_handlers>
```

Aucun retard de cas:

``` bash
$ curl 'http://localhost:8123/replicas_status'
Ok.
```

Retard de cas:

``` bash
$ curl 'http://localhost:8123/replicas_status'
db.stats:  Absolute delay: 22. Relative delay: 22.
```

## predefined\_query\_handler {#predefined_query_handler}

Vous pouvez configurer `<method>`, `<headers>`, `<url>` et `<queries>` dans `<predefined_query_handler>`.

`<method>` est responsable de la correspondance de la partie méthode de la requête HTTP. `<method>` entièrement conforme à la définition de [méthode](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods) dans le protocole HTTP. C'est une option de configuration. S'il n'est pas défini dans le fichier de configuration, il ne correspond pas à la partie méthode de la requête HTTP

`<url>` est responsable de la correspondance de la partie url de la requête HTTP. Il est compatible avec [RE2](https://github.com/google/re2)s 'expressions régulières. C'est une option de configuration. S'il n'est pas défini dans le fichier de configuration, il ne correspond pas à la partie url de la requête HTTP

`<headers>` est responsable de la correspondance de la partie d'en-tête de la requête HTTP. Il est compatible avec les expressions régulières de RE2. C'est une option de configuration. S'il n'est pas défini dans le fichier de configuration, il ne correspond pas à la partie d'en-tête de la requête HTTP

`<queries>` la valeur est une requête prédéfinie de `<predefined_query_handler>`, qui est exécuté par ClickHouse lorsqu'une requête HTTP est mise en correspondance et que le résultat de la requête est renvoyé. C'est une configuration incontournable.

`<predefined_query_handler>` prend en charge les paramètres de réglage et les valeurs query\_params.

L'exemple suivant définit les valeurs de `max_threads` et `max_alter_threads` Paramètres, puis interroge la table système pour vérifier si ces paramètres ont été définis avec succès.

Exemple:

``` xml
<root_handlers>
    <predefined_query_handler>
        <method>GET</method>
        <headers>
            <XXX>TEST_HEADER_VALUE</XXX>
            <PARAMS_XXX><![CDATA[(?P<name_1>[^/]+)(/(?P<name_2>[^/]+))?]]></PARAMS_XXX>
        </headers>
        <url><![CDATA[/query_param_with_url/\w+/(?P<name_1>[^/]+)(/(?P<name_2>[^/]+))?]]></url>
        <queries>
            <query>SELECT value FROM system.settings WHERE name = {name_1:String}</query>
            <query>SELECT name, value FROM system.settings WHERE name = {name_2:String}</query>
        </queries>
    </predefined_query_handler>
</root_handlers>
```

``` bash
$ curl -H 'XXX:TEST_HEADER_VALUE' -H 'PARAMS_XXX:max_threads' 'http://localhost:8123/query_param_with_url/1/max_threads/max_alter_threads?max_threads=1&max_alter_threads=2'
1
max_alter_threads   2
```

!!! note "Note"
    Dans un `<predefined_query_handler>`, un `<queries>` prend en charge un seul `<query>` d'un type d'insertion.

## dynamic\_query\_handler {#dynamic_query_handler}

`<dynamic_query_handler>` que `<predefined_query_handler>` augmenter `<query_param_name>` .

Clickhouse extrait et exécute la valeur correspondant au `<query_param_name>` valeur dans l'url de la requête HTTP.
ClickHouse réglage par défaut `<query_param_name>` être `/query` . C'est une option de configuration. Si il n'y a pas de définition dans le fichier de configuration, le paramètre n'est pas passé.

Pour expérimenter cette fonctionnalité, l'exemple définit les valeurs de max\_threads et max\_alter\_threads et demande si les paramètres ont été définis avec succès.
La différence est que dans `<predefined_query_handler>`, la requête est écrite dans le fichier de configuration. Mais dans `<dynamic_query_handler>`, la requête est écrite sous la forme de param de la requête HTTP.

Exemple:

``` xml
<root_handlers>
    <dynamic_query_handler>
        <headers>
            <XXX>TEST_HEADER_VALUE_DYNAMIC</XXX>
            <PARAMS_XXX><![CDATA[(?P<param_name_1>[^/]+)(/(?P<param_name_2>[^/]+))?]]></PARAMS_XXX>
        </headers>
        <query_param_name>query_param</query_param_name>
    </dynamic_query_handler>
</root_handlers>
```

``` bash
$ curl  -H 'XXX:TEST_HEADER_VALUE_DYNAMIC' -H 'PARAMS_XXX:max_threads' 'http://localhost:8123/?query_param=SELECT%20value%20FROM%20system.settings%20where%20name%20=%20%7Bname_1:String%7D%20OR%20name%20=%20%7Bname_2:String%7D&max_threads=1&max_alter_threads=2&param_name_2=max_alter_threads'
1
2
```

[Article Original](https://clickhouse.tech/docs/en/interfaces/http_interface/) <!--hide-->
