# Exploring-Deafness-with-LiLa

This repository contains a set of query for exploring the concept of "deafness" in Latin using the LiLa Knowledge Base. The data extracted from these queries have been presented at the "[Deafness in the Premodern Mediterranean](https://csmbr.fondazionecomel.org/events/vivamente-conferences/deafness-premodern-mediterranean/)" conference (Pisa, 17-18 November 2026).

## Endpoint
Data can be queried through the following endpoint: [https://lila-erc.eu/sparql/](https://lila-erc.eu/sparql/).

## SPARQL queries
1) Retrieve, for the lemma _surdus_, all lexical entries that are linked to it along with their senses and definitions. Results are grouped by named graph, and anything belonging to the Latin WordNet lexicon is excluded.

```
PREFIX ontolex: <http://www.w3.org/ns/lemon/ontolex#>
PREFIX lime:    <http://www.w3.org/ns/lemon/lime#>
PREFIX skos:    <http://www.w3.org/2004/02/skos/core#>
PREFIX rdfs:    <http://www.w3.org/2000/01/rdf-schema#>
PREFIX dct:     <http://purl.org/dc/terms/>

SELECT DISTINCT ?graph ?entry ?rel ?sense ?definition
WHERE {
  BIND(<http://lila-erc.eu/data/id/lemma/127095> AS ?lemma) # change with <http://lila-erc.eu/data/id/lemma/127092> for "surditas"
  BIND(<http://lila-erc.eu/data/lexicalResources/LatinWordNet/Lexicon> AS ?lwn)

  GRAPH ?graph {
    {
      ?entry ?rel ?lemma .
    }
    UNION
    {
      BIND(?lemma AS ?entry)
      BIND(<urn:self> AS ?rel)
    }

    ?entry ontolex:sense ?sense .

    OPTIONAL { ?sense skos:definition ?def1 }
    OPTIONAL { ?sense rdfs:label      ?def2 }
    OPTIONAL { ?sense dct:description ?def3 }
    BIND(COALESCE(?def1, ?def2, ?def3) AS ?definition)
  }

  # exclude Latin WordNet
  FILTER (?graph != ?lwn)
  FILTER NOT EXISTS { ?lwn lime:entry ?entry }
}
ORDER BY ?graph ?entry ?sense
```

2) Finds every lemma in the knowledge base that shares the same base of _surdus_. In practice, it retrieves the family of lemmas derived from the same root as _surdus_.

```
PREFIX lila: <http://lila-erc.eu/ontologies/lila/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT DISTINCT ?base ?baseLabel ?lemma ?label ?pos
WHERE {
  # base of surdus
  <http://lila-erc.eu/data/id/lemma/127095> lila:hasBase ?base .
  OPTIONAL { ?base rdfs:label ?baseLabel . }

  # all the lemmas of the same base
  ?lemma lila:hasBase ?base ;
         rdfs:label ?label .
  OPTIONAL { ?lemma lila:hasPOS ?pos . }
}
ORDER BY ?label

```

3) Finds the lemmas belonging to the derivational family of _surdus_. For each derived lemma it returns its label, the label of its immediate parent in the derivation, the number of corpus tokens attesting it, and its Lewis & Short sense definitions (one row per sense). Results are sorted by token frequency, highest first.

```
PREFIX ontolex:  <http://www.w3.org/ns/lemon/ontolex#>
PREFIX vartrans: <http://www.w3.org/ns/lemon/vartrans#>
PREFIX lila:     <http://lila-erc.eu/ontologies/lila/>
PREFIX rdfs:     <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?derived ?label ?parentLabel ?nTokens ?definition
WHERE {
  {
    SELECT ?derived ?label ?parentLabel (COUNT(?t) AS ?nTokens)
    WHERE {
      ?src ?linkIn <http://lila-erc.eu/data/id/lemma/127095> .
      ?src (^vartrans:source/vartrans:target)+ ?tgt .

      ?tgt ?linkOut ?derived .
      ?derived a lila:Lemma ;
               rdfs:label ?label .

      ?rel vartrans:target ?tgt ;
           vartrans:source ?parentEntry .
      ?parentEntry ?linkP ?parent .
      ?parent a lila:Lemma ;
              rdfs:label ?parentLabel .

      OPTIONAL { ?t lila:hasLemma ?derived . }
    }
    GROUP BY ?derived ?label ?parentLabel
  }

  # definitions from Lewis & Short: one row for each sense
  OPTIONAL {
    GRAPH <http://lila-erc.eu/sparql/lila_knowledge_base/lexicalResources/CIRCSE/LewisShort> {
      ?le ontolex:canonicalForm ?derived ;
          ontolex:sense ?sense .
      ?sense rdfs:label ?definition .
    }
  }
}
ORDER BY DESC(?nTokens) ?label ?definition

```

4) Finds every corpus token of the lemma _surdus_ and returns the token’s form, the work and author it comes from, and the full sentence in which it occurs. The sentence text is rebuilt by concatenating the string values of all tokens under the same sentence node, ordered by the numeric suffix of their URIs. Results are sorted by author, work and sentence.

```
PREFIX rdf:   <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>
PREFIX powla: <http://purl.org/powla/powla.owl#>
PREFIX dc:    <http://purl.org/dc/elements/1.1/>
PREFIX dct:   <http://purl.org/dc/terms/>
PREFIX lila:  <http://lila-erc.eu/ontologies/lila/>
PREFIX xsd:   <http://www.w3.org/2001/XMLSchema#>

SELECT ?tokenURI ?tokenLabel ?work ?author ?sentence ?sentenceText
WHERE {
  # sentence content
  {
    SELECT ?tokenURI ?sentence
           (GROUP_CONCAT(?w; separator=" ") AS ?sentenceText)
    WHERE {
      {
        SELECT ?tokenURI ?sentence ?w WHERE {
          ?tokenURI lila:hasLemma <http://lila-erc.eu/data/id/lemma/127095> ; # http://lila-erc.eu/data/id/lemma/127092
                    rdf:type powla:Terminal .
          { ?tokenURI powla:hasParent ?sentence } UNION { ?sentence powla:hasChild ?tokenURI }
          FILTER (CONTAINS(STR(?sentence), "/Sentence_"))

          { ?sib powla:hasParent ?sentence } UNION { ?sentence powla:hasChild ?sib }
          ?sib powla:hasStringValue ?w .
          BIND(xsd:integer(REPLACE(STR(?sib), "^.*[^0-9]([0-9]+)/?$", "$1")) AS ?pos)
        }
        ORDER BY ?tokenURI ?pos
      }
    }
    GROUP BY ?tokenURI ?sentence
  }

  # form
  ?tokenURI powla:hasStringValue ?tokenLabel .

  # work
  ?tokenURI powla:hasLayer ?docLayer .
  ?docLayer powla:hasDocument ?doc .
  ?doc dc:title ?work .

  # author
  OPTIONAL {
    { ?doc dc:creator ?auth } UNION { ?doc dct:creator ?auth }
    OPTIONAL { ?auth rdfs:label ?authLabel }
  }
  BIND(COALESCE(?authLabel, STR(?auth)) AS ?author)
}
ORDER BY ?author ?work ?sentence
```

5) Finds the Latin WordNet synsets (lexical concepts) evoked by the lemma _surdus_. For each synset it returns a definition, the number of distinct lemmas that evoke it, and a comma-separated list of their labels, i.e. the synonyms of surdus in each sense.

```
PREFIX ontolex: <http://www.w3.org/ns/lemon/ontolex#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
PREFIX wn: <https://globalwordnet.github.io/schemas/wn#>

SELECT ?syn
       (SAMPLE(?def) AS ?definition)
       (COUNT(DISTINCT ?l) AS ?nLemmas)
       (GROUP_CONCAT(DISTINCT ?lab; separator=", ") AS ?lemmas)
WHERE {
  # synsets evoked by the selected lemma
  ?le0 ontolex:canonicalForm <http://lila-erc.eu/data/id/lemma/127095> ; # <http://lila-erc.eu/data/id/lemma/127092> for "surditas"
       ontolex:evokes ?syn .

  # synsets taken from Latin WordNet
  FILTER(STRSTARTS(STR(?syn), "http://lila-erc.eu/data/lexicalResources/LatinWordNet/id/LexicalConcept/"))

  # lemmas of the same synset
  ?le ontolex:evokes ?syn ;
      ontolex:canonicalForm ?l .
  ?l rdfs:label ?lab .

  # definitions
  OPTIONAL {
    ?syn (skos:definition | wn:definition | wn:definition/rdf:value | wn:gloss) ?def .
  }
}
GROUP BY ?syn
ORDER BY ?syn
```
