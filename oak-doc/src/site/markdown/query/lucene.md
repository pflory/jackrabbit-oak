<!--
   Licensed to the Apache Software Foundation (ASF) under one or more
   contributor license agreements.  See the NOTICE file distributed with
   this work for additional information regarding copyright ownership.
   The ASF licenses this file to You under the Apache License, Version 2.0
   (the "License"); you may not use this file except in compliance with
   the License.  You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
  -->

## Lucene Index

**Following details are applicable for Oak release 1.0.9 onwards. For pre 1.0
.9 release refer to [Pre 1.0.9 Lucene documentation](lucene-old.html)**

Oak supports Lucene based indexes to support both property constraint and full
text constraints. Depending on the configuration a Lucene index can be used
to evaluate property constraints, full text constraints, path restrictions
and sorting.

    SELECT * FROM [nt:base] WHERE [assetType] = 'image'

Following index definition would allow using Lucene index for above query

```
/oak:index/assetType
  - jcr:primaryType = "oak:QueryIndexDefinition"
  - compatVersion = 2
  - type = "lucene"
  - async = "async"
  + indexRules
    - jcr:primaryType = "nt:unstructured"
    + nt:base
      + properties
        - jcr:primaryType = "nt:unstructured"
        + assetType
          - propertyIndex = true
          - name = "assetType"
```

The index definition node for a lucene-based index

* must be of type `oak:QueryIndexDefinition`
* must have the `type` property set to __`lucene`__
* must contain the `async` property set to the value `async`, this is what
  sends the index update process to a background thread

_Note that compared to [Property Index](query.html#property-index) Lucene
Property Index is always configured in Async mode hence it might lag behind
in reflecting the current repository state while performing the query_

Taking another example. To support following query

    //*[jcr:contains(., 'text')]

The Lucene index needs to be configured to index all properties

    /oak:index/assetType
      - jcr:primaryType = "oak:QueryIndexDefinition"
      - compatVersion = 2
      - type = "lucene"
      - async = "async"
      + indexRules
        - jcr:primaryType = "nt:unstructured"
        + nt:base
          + properties
            - jcr:primaryType = "nt:unstructured"
            + allProps
              - name = ".*"
              - isRegexp = true
              - nodeScopeIndex = true

### Index Definition

Lucene index definition consist of `indexingRules`, `analyzers` ,
`aggregates` etc which determine which node and properties are to be indexed
and how they are indexed.

Below is the canonical index definition structure

    luceneIndex (oak:QueryIndexDefinition)
      - type (string) = 'lucene' mandatory
      - async (string) = 'async' mandatory
      - blobSize (long) = 32768
      - evaluatePathRestrictions (boolean) = false
      - name (string)
      - compatMode (long) = 2
      + indexRules (nt:unstructured)
      + aggregates (nt:unstructured)
      + analyzers (nt:unstructured)

Following are the config options which can be defined at the index definition
level

type
: Required and should always be `lucene`

async
: Required and should always be `async`

[blobSize][OAK-2201]
: Default value 32768 (32kb)
: Size in bytes used for splitting the index files when storing them in NodeStore

functionName
: Name to be used to enable index usage with [native query support](#native-query)

evaluatePathRestrictions
: Optional boolean property defaults to `false`
: If enabled the index can evaluate [path restrictions](#path-restrictions)

name
: Optional property
: Captures the name of the index which is used while logging

compatMode
: Required integer property and should be set to 2
: By default Oak uses older Lucene index implementation which does not
  supports property restrictions, index time aggregation etc. To make use of
  this feature set it to 2

#### Indexing Rules

Indexing rules defines which types of node and properties are indexed. An
index configuration can define one or more `indexingRules` for different
nodeTypes.

    fulltextIndex
      - jcr:primaryType = "oak:QueryIndexDefinition"
      - compatVersion = 2
      - type = "lucene"
      - async = "async"
      + indexRules
        - jcr:primaryType = "nt:unstructured"
        + app:Page
          + properties
            - jcr:primaryType = "nt:unstructured"
            + publishedDate
              - propertyIndex = true
              - name = "jcr:content/publishedDate"
        + app:Asset
          + properties
            - jcr:primaryType = "nt:unstructured"
            + imageType
              - propertyIndex = true
              - name = "jcr:content/metadata/imageType"

Rules are defined per nodeType and each rule has one or more property
definitions determine which properties are indexed. Below is the canonical index
definition structure

    ruleName (nt:unstructured)
      - inherited (boolean) = true
      - includePropertyTypes (string) multiple
      + properties (nt:unstructured)

Following are the config options which can be defined at the index rule
level

inherited
: Optional boolean property defaults to true
: Determines if the rule is applicable on exact match or can be applied if
  match is done on basis of nodeType inheritance

includePropertyTypes
: Applicable when index is enabled for fulltext indexing
: For full text index defaults to include all types
: String array of property types which should be indexed. The values can be one
  specified in [PropertyType Names][1]


##### Indexing Rule inheritance

`indexRules` are defined per nodeType and support nodeType inheritance. For
example while indexing any node the indexer would lookup for applicable
indexRule for that node based on its _primaryType_. If a direct match is
found then that rule would be used otherwise it would look for rule for any
of the parent types. The rules are looked up in the order of there entry
under `indexRules` node (indexRule node itself is of type `nt:unstructured`
which has `orderable` child nodes)

If `inherited` is set to false on any rule then that rule would only be
applicable if exact match is found

##### Property Definitions

Each index rule consist of one ore more property definition defined under
`properties`. Order of property definition node is important as some properties
are based on regular expressions. Below is the canonical property definition
structure

    propNode (nt:unstructured)
      - name (string)
      - boost (double) = '1.0'
      - index (boolean) = true
      - useInExcerpt (boolean) = false
      - analyzed (boolean) = false
      - nodeScopeIndex (boolean) = false
      - ordered (boolean) = false
      - isRegexp (boolean) = false
      - type (string) = 'undefined'

Following are the details about the above mentioned config options which can be
defined at the property definition level

name
: Property name. If not defined then property name is set to the node name.
  If `isRegexp` is true then it defines the regular expression. Can also be set
  to a relative property.

isRegexp
: If set to true then property name would be interpreted as a regular
  expression and the given definition would be applicable for matching property
  names. Note that expression should be structured such that it does not
  match '/'.
    * `.*` - This property definition is applicable for all properties of given
      node
    * `jcr:content/metadata/.*` - This property definition is
      applicable for all properties of child node _jcr:content/metadata_

boost
: If the property is included in `nodeScopeIndex` then it defines the boost
  done for the index value against the given property name.

index
: Determines if this property should be indexed. Mostly useful for fulltext
  index where some properties need to be _excluded_ from getting indexed.

useInExcerpt
: Controls whether the value of a property should be used to create an excerpt.
  The value of the property is still full-text indexed when set to false, but it
  will never show up in an excerpt for its parent node. If set to true then
  property value would be stored separately within index causing the index
  size to increase. So set it to true only if you make use of excerpt feature

nodeScopeIndex
: Control whether the value of a property should be part of fulltext index. That
  is, you can do a _jcr:contains(., 'foo')_ and it will return nodes that have a
  string property that contains the word foo. Example
    * _//element(*, app:Asset)[jcr:contains(., 'image')]_

analyzed
: Set this to true if the property is used as part of `contains`. Example
    * _//element(*, app:Asset)[jcr:contains(type, 'image')]_
    * _//element(*, app:Asset)[jcr:contains(jcr:content/metadata/@format, 'image')]_

ordered
: If the property is to be used in _order by_ clause to perform sorting then
  this should be set to true. This should be set to true only if the property
  is to be used to perform sorting as it increases the index size. Example
    * _//element(*, app:Asset)[jcr:contains(type, 'image')] order by @size_
    * _//element(*, app:Asset)[jcr:contains(type, 'image')] order by
    jcr:content/@jcr:lastModified_

  Refer to [Lucene based Sorting][OAK-2196] for more details

type
: JCR Property type. Can be one of `Date`, `Boolean`, `Double` or `Long`. Mostly
  inferred from the indexed value. However in some cases where same property
  type is not used consistently across various nodes then it would recommened
   to specify the type explicitly.

**Property Names**

Property name can be one of following

1. Simple name - Like _assetType_ etc. These are used for properties which are
   defined directly on the indexed node
2. Relative name - Like _jcr:content/metadata/title_. These are used for
   properties which are defined relative to the node being indexed.
3. Regular Expression - Like _.*_. Used when only property whose name
   match given pattern are to be indexed.
   They can also be used for relative properties like
   _jcr:content/metadata/dc:.*$_
   which indexes all property names starting with _dc_ from node with
   relative path _jcr:content/metadata_

<a name="path-restrictions"></a>
##### Evaluate Path Restrictions

Lucene index provides support for evaluating path restrictions natively.
Consider a query like

    select * from [app:Asset] as a where isdescendantnode(a, [/content/app/old]) AND contains(*, 'white')

By default the index would return all node which _contain white_ and Query
engine would filter out nodes which are not under _/content/app/old_. This
can perform slow if lots of nodes are not under that path. To speed up such
queries one can enable `evaluatePathRestrictions` in Lucene index and index
would only return nodes which are under _/content/app/old_.

Enabling this feature would incur cost in terms of slight increase in index
size. Refer to [OAK-2306][OAK-2306] for more details.

#### Aggregation

Sometimes it is useful to include the contents of descendant nodes into a single
node to easier search on content that is scattered across multiple nodes.

Oak allows you to define index aggregates based on relative path patterns and
primary node types. Changes to aggregated items cause the main item to be
reindexed, even if it was not modified.

Aggregation configuration is defined under the `aggregates` node under index
configuration. The following example creates an index aggregate on nt:file that
includes the content of the jcr:content node:

    fulltextIndex
      - jcr:primaryType = "oak:QueryIndexDefinition"
      - compatVersion = 2
      - type = "lucene"
      - async = "async"
      + aggregates
        + nt:file
          + include0
            - path = "jcr:content"

For a given nodeType multiple includes can be defined. Below is the aggregate
definition structure for any specific include rule

    aggregateNodeInclude (nt:unstructured)
      - path (string) mandatory
      - primaryType (string)
      - relativeNode (boolean) = false

Following are the details about the above mentioned config options which can be
defined as part of aggregation include. (Refer to [OAK-2268][OAK-2268] for
implementation details)

path
: Path pattern to include. Example
    * `jcr:content` - Name explicitly specified
    * `*` - Any child node at depth 1
    * `*/*` - Any child node at depth 2

primaryType
: Restrict the included nodes to a certain type. The restriction would be
  applied on the last node in given path

        + aggregates
          + nt:file
            + include0
              - path = "jcr:content"
              - primaryType = "nt:resource"

relativeNode
: Boolean property indicates that query can be performed against specific node
  For example for following content

        + space.txt (app:Asset)
          + renditions (nt:folder)
            + original (nt:file)
              + jcr:content (nt:resource)
                - jcr:data

  And a query like

        select * from [app:Asset] where contains(renditions/original/*, "pluto")

  Following index configuration would be required

        fulltextIndex
          - jcr:primaryType = "oak:QueryIndexDefinition"
          - compatVersion = 2
          - type = "lucene"
          - async = "async"
          + aggregates
            + nt:file
              + include0
                - path = "jcr:content"
            + app:Asset
              + include0
                - path = "renditions/original"
                - relativeNode = true
          + indexRules
            - jcr:primaryType = "nt:unstructured"
            + app:Asset

**Aggregation and Recursion**

While performing aggregation the aggregation rules are again applied on node
being aggregated. For example while aggregating for _app:Asset_ above when
_renditions/original/*_ is being aggregated then aggregation rule would again
be applied. In this case as  _renditions/original_ is _nt:file_ then aggregation
rule applicable for _nt:file_ would be applied. Such a logic might result in
recursion. (See [JCR-2989][JCR-2989] for details).

For such case `reaggregateLimit` is set on aggregate definition node and
defaults to 5

      + aggregates
        + app:Asset
          - reaggregateLimit (long) = 5
          + include0
            - path = "renditions/original"
            - relativeNode = true

#### Analyzers

Analyzers can be configured as part of index definition via `analyzers` node.
The default analyzer can be configured via `analyzers/default` node

    + sampleIndex
        - jcr:primaryType = "oak:QueryIndexDefinition"
        + analyzers
            + default
            + pathText
            ...

##### Specify analyzer class directly

If any of the out of the box analyzer is to be used then it can configured directly

    + analyzers
            + default
                - class = "org.apache.lucene.analysis.standard.StandardAnalyzer"
                - luceneMatchVersion = "LUCENE_47" (optional)

To confirm to specific version specify it via `luceneMatchVersion` otherwise Oak
would use a default version depending on version of Lucene it is shipped with.

One can also provide a stopword file via `stopwords` `nt:file` node under
the analyzer node

    + analyzers
            + default
                - class = "org.apache.lucene.analysis.standard.StandardAnalyzer"
                - luceneMatchVersion = "LUCENE_47" (optional)
                + stopwords (nt:file)

##### Create analyzer via composition

Analyzers can also be composed based on `Tokenizers`, `TokenFilters` and
`CharFilters`. This is similar to the support provided in Solr where you can
[configure analyzers in xml][solr-analyzer]

    + analyzers
            + default
                + charFilters (nt:unstructured) //The filters needs to be ordered
                    + HTMLStrip
                    + Mapping
                + tokenizer
                    - name = "Standard"
                + filters (nt:unstructured) //The filters needs to be ordered
                    + LowerCase
                    + Stop
                        - stopWordFiles = "stop1.txt, stop2.txt"
                        + stop1.txt (nt:file)
                        + stop2.txt (nt:file)
                    + PorterStem

Points to note

1. Name of filters, charFilters and tokenizer are formed by removing the
   factory suffixes. So
    * org.apache.lucene.analysis.standard.StandardTokenizerFactory -> standard
    * org.apache.lucene.analysis.charfilter.MappingCharFilterFactory -> Mapping
    * org.apache.lucene.analysis.core.StopFilterFactory -> Stop
2. Any config parameter required for the factory is specified as property of
   that node
    * If the factory requires to load a file e.g. stop words from some file then
      file content can be provided via creating child `nt:file` node of the
      filename


<a name="osgi-config"></a>
### LuceneIndexProvider Configuration

Some of the runtime aspects of the Oak Lucene support can be configured via OSGi
configuration. The configuration needs to be done for PID `org.apache
.jackrabbit.oak.plugins.index.lucene.LuceneIndexProviderService`

![OSGi Configuration](lucene-osgi-config.png)

enableCopyOnReadSupport
: Enable copying of Lucene index to local file system to improve query 
performance. See [Copy Indexes On Read](#copy-on-read)

localIndexDir
: Directory to be used for when copy index files to local file system. To be 
specified when `enableCopyOnReadSupport` is enabled

debug
: Boolean value. Defaults to `false`
: If enabled then Lucene logging would be integrated with Slf4j

<a name="non-root-index"></a>
### Non Root Index Definitions

Lucene index definition can be defined at any location in repository and need
not always be defined at root. For example if your query involves path 
restrictions like

    select * from [app:Asset] as a where ISDESCENDANTNODE(a, '/content/companya') and [format] = 'image'
    
Then you can create the required index definition say `assetIndex` at 
`/content/companya/oak:index/assetIndex`. In such a case that index would 
contain data for the subtree under `/content/companya`

<a name="native-query"></a>
### Native Query and Index Selection

Oak query engine supports native queries like

    //*[rep:native('lucene', 'name:(Hello OR World)')]

If multiple Lucene based indexes are enabled on the system and you need to 
make use of specific Lucene index like `/oak:index/assetIndex` then you can 
specify the index name via `functionName` attribute on index definition. 

For example for assetIndex definition like 

    - jcr:primaryType = "oak:QueryIndexDefinition"
    - type = "lucene"
    ...
    - functionName = "lucene-assetIndex"

Executing following query would ensure that Lucene index from `assetIndex` 
should be used

    //*[rep:native('lucene-assetIndex', 'name:(Hello OR World)')]

### Persisting indexes to FileSystem

By default Lucene indexes are stored in the `NodeStore`. If required they can
be stored on the file system directly

    - jcr:primaryType = "oak:QueryIndexDefinition"
    - type = "lucene"
    ...
    - persistence = "file"
    - path = "/path/to/store/index"

To store the Lucene index in the file system, in the Lucene index definition
node, set the property `persistence` to `file`, and set the property `path` 
to the directory where the index should be stored. Then start reindexing by 
setting `reindex` to `true`.

Note that this setup would only for those non cluster `NodeStore`. If the 
backend `NodeStore` supports clustering then index data would not be 
accessible on other cluster nodes

<a name="copy-on-read"></a>
### CopyOnRead

Lucene indexes are stored in `NodeStore`. Oak Lucene provides a custom directory
implementation which enables Lucene to load index from `NodeStore`. This 
might cause performance degradation if the `NodeStore` storage is remote. For
such case Oak Lucene provide a `CopyOnReadDirectory` which copies the index 
content to a local directory and enables Lucene to make use of local 
directory based indexes while performing queries.

At runtime various details related to copy on read features are exposed via
`CopyOnReadStats` MBean. Indexes at JCR path e.g. `/oak:index/assetIndex` 
would be copied to `<index dir>/<hash of jcr path>`. To determine mapping 
between local index directory and JCR path refer to the MBean details

![CopyOnReadStats](lucene-copy-on-read.png)
  
For more details refer to [OAK-1724][OAK-1724]. This feature can be enabled via
[Lucene Index provider service configuration](#osgi-config)

### Lucene Index MBeans

Oak Lucene registers a JMX bean `LuceneIndex` which provide details about the 
index content e.g. size of index, number of documents present in index etc

![Lucene Index MBean](lucene-index-mbean.png)

<a name="luke"></a>
### Analyzing created Lucene Index

[Luke]  is a handy development and diagnostic tool, which accesses already 
existing Lucene indexes and allows you to display index details. In Oak 
Lucene index files are stored in `NodeStore` and hence not directly 
accessible. To enable analyzing the index files via Luke follow below 
mentioned steps

1. Download the Luke version which includes the matching Lucene jars used by 
   Oak. As of Oak 1.0.8 release the Lucene version used is 4.7.1. So download
    the jar from [here](https://github.com/DmitryKey/luke/releases)
     
        $wget https://github.com/DmitryKey/luke/releases/download/4.7.0/luke-with-deps.jar
        
2. Use the [Oak Console][oak-console] to dump the Lucene index from `NodeStore`
   to filesystem directory. Use the `lc dump` command
   
        $ java -jar oak-run-*.jar console /path/to/oak/repository
        Apache Jackrabbit Oak 1.1-SNAPSHOT
        Jackrabbit Oak Shell (Apache Jackrabbit Oak 1.1-SNAPSHOT, JVM: 1.7.0_55)
        Type ':help' or ':h' for help.
        -------------------------------------------------------------------------
        /> lc info /oak:index/lucene
        Index size : 74.1 MB
        Number of documents : 235708
        Number of deleted documents : 231
        /> lc 
        dump   info   
        /> lc dump /path/to/dump/index/lucene /oak:index/lucene
        Copying Lucene indexes to [/path/to/dump/index/lucene]
        Copied 74.1 MB in 1.209 s
        /> lc dump /path/to/dump/index/slingAlias /oak:index/slingAlias
        Copying Lucene indexes to [/path/to/dump/index/lucene-index/slingAlias]
        Copied 8.5 MB in 218.7 ms
        />
       
3. Post dump open the index via Luke. Oak Lucene uses a [custom 
   Codec][OAK-1737]. So oak-lucene jar needs to be included in Luke classpath
   for it to display the index details

        $ java -XX:MaxPermSize=512m luke-with-deps.jar:oak-lucene-1.0.8.jar org.getoptuke.Luke
        
From the Luke UI shown you can access various details.

### Index performance

Following are some best practices to get good performance from Lucene based 
indexes

1. Make use on [non root indexes](#non-root-index). If you query always 
  perform search under certain paths then create index definition under those 
  paths only. This might be helpful in multi tenant deployment where each tenant 
  data is stored under specific repository path and all queries are made under 
  those path.
   
2. Index only required data. Depending on your requirement you can create 
   multiple Lucene indexes. For example if in majority of cases you are 
   querying on various properties specified under `<node>/jcr:content/metadata`
   where node belong to certain specific nodeType then create single index 
   definition listing all such properties and restrict it that nodeType. You 
   can the size of index via mbean

[1]: http://www.day.com/specs/jsr170/javadocs/jcr-2.0/constant-values.html#javax.jcr.PropertyType.TYPENAME_STRING
[OAK-2201]: https://issues.apache.org/jira/browse/OAK-2201
[OAK-1724]: https://issues.apache.org/jira/browse/OAK-1724
[OAK-2196]: https://issues.apache.org/jira/browse/OAK-2196
[OAK-2005]: https://issues.apache.org/jira/browse/OAK-2005
[OAK-1737]: https://issues.apache.org/jira/browse/OAK-1737 
[OAK-2306]: https://issues.apache.org/jira/browse/OAK-2306
[OAK-2268]: https://issues.apache.org/jira/browse/OAK-2268
[luke]: https://code.google.com/p/luke/
[oak-console]: https://github.com/apache/jackrabbit-oak/tree/trunk/oak-run#console
[JCR-2989]: https://issues.apache.org/jira/browse/JCR-2989?focusedCommentId=13051101
[solr-analyzer]: https://wiki.apache.org/solr/AnalyzersTokenizersTokenFilters#Specifying_an_Analyzer_in_the_schema