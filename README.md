
The code and queries etc here are unlikely to be updated as my process
evolves. Later repos will likely have progressively different approaches
and more elaborate tooling, as my habit is to try to improve at least
one part of the process each time around.

---------

Step 1: Configure meta.json
===========================

All the relevant metadata now lives in meta.json: ideally nothing will
need tweaked after this.

Step 1: Scrape the results
==========================

```sh
jq -r '.wikipedia | "\(.url) \(.section)"' meta.json | xargs bundle exec ruby scraper.rb | tee wikipedia.csv
```

Step 2: Generate possible missing IDs
=====================================

```sh
xsv search -v -s id 'Q' wikipedia.csv | xsv select name | tail +2 |
  sed -e 's/^/"/' -e 's/$/"@en/' | paste -s - |
  xargs -0 wd sparql find-candidates.js |
  jq -r '.[] | [.name, .item.value, .election.label, .constituency.label, .party.label] | @csv' |
  tee candidates.csv
```

The ones found look good, but there's no match for:

* Gillian Troughton
* Michael Guest

I can't find either in Wikidata already.

Step 3: Combine Those
=====================

```sh
xsv join -n --left 2 wikipedia.csv 1 candidates.csv | xsv select '7,1-5' | sed $'1i\\\nfoundid' | tee combo.csv
```

Step 4: Generate QuickStatements commands
=========================================

```sh
bundle exec ruby generate-qs.rb meta.json | tee commands.qs
```

Then sent to QuickStatements as https://editgroups.toolforge.org/b/QSv2T/1597074638719/

Step 5: Generate reciprocal 'candidacy' statements
==================================================

```sparql
SELECT ?person ?personLabel ?election ?electionLabel
WHERE {
  ?election wdt:P31 wd:Q7864918 ; wdt:P726 ?person.
  MINUS { ?person wdt:P3602 ?election }
  SERVICE wikibase:label { bd:serviceParam wikibase:language "[AUTO_LANGUAGE],en". }
}
```

```sh
wd sparql mirror-candidacy-to-person.rq |
  jq -r '.[] | "\(.person.value)\tP3602\t\(.election.value)\tS3452\t\(.election.value)"' |
  tee >(pbcopy)
```

-> QuickStatements as https://editgroups.toolforge.org/b/QSv2T/1597075087129/

