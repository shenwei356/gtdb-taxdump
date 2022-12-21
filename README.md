# GTDB taxonomy taxdump files with trackable TaxIds

Metagenomic tools like [Kraken2](https://github.com/DerrickWood/kraken2),
 [Centrifuge](https://github.com/DaehwanKimLab/centrifuge)
 and [KMCP](https://github.com/shenwei356/kmcp) support NCBI taxonomy in format of [NCBI taxdump files](https://ftp.ncbi.nlm.nih.gov/pub/taxonomy/).
[GTDB](https://gtdb.ecogenomic.org/), a prokaryotic genomes catalogue, has its own taxonomy data.
Though the genomes, derived from GenBank and RefSeq, can be mappped to NCBI taxonomy TaxIds,
there's an urgent need to create its own taxonomy taxdump files with ***stable and trackable*** TaxIds.

A [TaxonKit](https://github.com/shenwei356/taxonkit) command, [taxonkit create-taxdump](https://bioinf.shenwei.me/taxonkit/usage/#create-taxdump) is used
to create NCBI-style taxdump files for any taxonomy dataset,
including [GTDB](https://gtdb.ecogenomic.org/) and [ICTV](https://talk.ictvonline.org/).

Related projects:

- [ictv-taxdump](https://github.com/shenwei356/ictv-taxdump): NCBI-style taxdump files for International Committee on Taxonomy of Viruses (ICTV)
- [taxid-changelog](https://github.com/shenwei356/taxid-changelog): NCBI taxonomic identifier (taxid) changelog
- [taxonkit](https://github.com/shenwei356/taxonkit): A Practical and Efficient NCBI Taxonomy Toolkit

## Table of Contents

* [Method](#method)
    + [Taxonomic hierarchy](#taxonomic-hierarchy)
    + [Generation of TaxIds](#generation-of-taxids)
    + [Data and tools](#data-and-tools)
    + [Steps](#steps)
* [Download](#download)
* [Results](#results)
    + [Taxon history of *Escherichia coli*](#taxon-history-of-escherichia-coli)
    + [Species of the genus *Escherichia*](#species-of-the-genus-escherichia)
    + [Common manipulations](#common-manipulations)
* [Known issues](#known-issues)
* [Merging GTDB and NCBI taxonomy](#merging-gtdb-and-ncbi-taxonomy)
* [Citation](#citation)
* [Contributing](#contributing)
* [License](#license)

## Method

### Taxonomic hierarchy

A GTDB species cluster contains >=1 assemblies, each can be treated as a strain.
So we can assign each assembly a TaxId with the rank of "no rank" below the species rank.
Therefore, we can also track the changes of these assemblies via the TaxId later.

### Generation of TaxIds

We just hash the taxon name (in lower case) of each taxon node to `uint64`
using [xxhash](https://github.com/cespare/xxhash/) and convert it to `int32`.

- For the NCBI assembly accession. 
  1) The prefix `GCA_` is not used because some GenBank entries (`GCA_000176655.2` in R80) were moved
  to RefSeq (`GCF_000176655.2` in R83) and the prefix changed. 
  2) The version number is trimed because it may change.
  So, `000176655` is hashed to get the TaxId.
- For the non-NCBI assembly accession. The accession per se is hashed. E.g., `UBA12275`
- For the name of a node. The taxon name per se is hashed. E.g, `Bacteria`.


For these missing some taxon nodes, GTDB uses names of parent nodes
e.g., [GCA_018897955.1](https://gtdb.ecogenomic.org/genome?gid=GCA_018897955.1).
So in these cases, TaxIds keep distinct.

    GB_GCA_018897955.1      d__Archaea;p__EX4484-52;c__EX4484-52;o__EX4484-52;f__LFW-46;g__LFW-46;s__LFW-46 sp018897155

We also detect duplicate names with different ranks, e.g., [GCA_003663585.1](https://gtdb.ecogenomic.org/genome?gid=GCA_003663585.1).
The Class and Genus have the same name `B47-G6`, while the Order and Family between them have different names.
In this case, we reassign a new TaxId by increasing the TaxId of name at lower rank until it being distinct.

    GB_GCA_003663585.1      d__Archaea;p__Thermoplasmatota;c__B47-G6;o__B47-G6B;f__47-G6;g__B47-G6;s__B47-G6 sp003663585


### Data and tools

GTDB taxnomy files are download from https://data.gtdb.ecogenomic.org/releases/, and organized as:

    tree taxonomy/
    taxonomy/
    ├── R080
    │   └── bac_taxonomy_r80.tsv
    ├── R083
    │   └── bac_taxonomy_r83.tsv
    ├── R086
    │   ├── ar122_taxonomy_r86.2.tsv
    │   └── bac120_taxonomy_r86.2.tsv
    ├── R089
    │   ├── ar122_taxonomy_r89.tsv
    │   └── bac120_taxonomy_r89.tsv
    ├── R095
    │   ├── ar122_taxonomy_r95.tsv.gz
    │   └── bac120_taxonomy_r95.tsv.gz
    ├── R202
    │   ├── ar122_taxonomy_r202.tsv.gz
    │   └── bac120_taxonomy_r202.tsv.gz
    └── R207
        ├── ar53_taxonomy_r207.tsv.gz
        └── bac120_taxonomy_r207.tsv.gz

[TaxonKit](https://github.com/shenwei356/taxonkit) v0.12.0 or a later version is needed.
[v0.14.0](https://github.com/shenwei356/taxonkit/blob/master/CHANGELOG.md) or a later version is preferred.
**Since v0.14.0, [taxonkit create-taxdump](https://bioinf.shenwei.me/taxonkit/usage/#create-taxdump) stores
TaxIds in `int32` following BLAST and DIAMOND, rather than `uint32` in previous versions**.

### Steps
    
1. Generating taxdump files for the first version r80:

        taxonkit create-taxdump taxonomy/R080/*.tsv* --gtdb --out-dir gtdb-taxdump/R080 --force
        15:19:59.816 [WARN] --gtdb-re-subs failed to extract ID for subspecies, the origninal value is used instead. e.g., UBA11420
        21:52:12.406 [INFO] 94759 records saved to gtdb-taxdump/R080/taxid.map
        21:52:12.467 [INFO] 110320 records saved to gtdb-taxdump/R080/nodes.dmp
        21:52:12.506 [INFO] 110320 records saved to gtdb-taxdump/R080/names.dmp
        21:52:12.506 [INFO] 0 records saved to gtdb-taxdump/R080/merged.dmp
        21:52:12.506 [INFO] 0 records saved to gtdb-taxdump/R080/delnodes.dmp
    
2. For later versions, we need the taxdump files of the revious version to track merged and deleted nodes.

        taxonkit create-taxdump --gtdb -x gtdb-taxdump/R080/ \
            taxonomy/R083/*.tsv*  --out-dir gtdb-taxdump/R083  --force
            
        taxonkit create-taxdump --gtdb -x gtdb-taxdump/R083/ \
            taxonomy/R086/*.tsv*  --out-dir gtdb-taxdump/R086  --force

        taxonkit create-taxdump --gtdb -x gtdb-taxdump/R086/ \
            taxonomy/R089/*.tsv*  --out-dir gtdb-taxdump/R089  --force
            
        taxonkit create-taxdump --gtdb -x gtdb-taxdump/R089/ \
            taxonomy/R095/*.tsv*  --out-dir gtdb-taxdump/R095  --force
            
        taxonkit create-taxdump --gtdb -x gtdb-taxdump/R095/ \
            taxonomy/R202/*.tsv*  --out-dir gtdb-taxdump/R202  --force
            
        taxonkit create-taxdump --gtdb -x gtdb-taxdump/R202/ \
            taxonomy/R207/*.tsv*  --out-dir gtdb-taxdump/R207  --force
            
3. Generating TaxId changelog

        taxonkit taxid-changelog -i gtdb-taxdump -o gtdb-taxid-changelog.csv.gz --verbose

## Download

The [release page](https://github.com/shenwei356/gtdb-taxdump/releases) contains taxdump files for all GTDB versions,
and a TaxId changelog file (gtdb-taxid-changelog.csv.gz).

Learn more about the [taxid-changelog](https://github.com/shenwei356/taxid-changelog).

## Results

### Summary

Lineages (R207)

    $ cat gtdb-taxdump/R207/taxid.map  \
        | csvtk freq -Ht -f 2 -nr \
        | taxonkit lineage -r -n -L --data-dir gtdb-taxdump/R207/ \
        | taxonkit reformat -I 1 -f '{k}\t{p}\t{c}\t{o}\t{f}\t{g}\t{s}' --data-dir gtdb-taxdump/R207/ \
        | csvtk add-header -t -n 'taxid,count,name,rank,superkindom,phylum,class,order,family,genus,species' \
        > taxid.map.stats.tsv
        
Frequency of species

    $ csvtk freq -t -nr -f species taxid.map.stats.tsv \
        > taxid.map.stats.freq-species.tsv
        
    $ head -n 21 taxid.map.stats.freq-species.tsv \
        | csvtk pretty -t
    species                      frequency
    --------------------------   ---------
    Escherichia coli             26859
    Staphylococcus aureus        13059
    Salmonella enterica          12285
    Klebsiella pneumoniae        11294
    Streptococcus pneumoniae     8452
    Mycobacterium tuberculosis   6836
    Pseudomonas aeruginosa       5623
    Acinetobacter baumannii      5417
    Clostridioides difficile     2225
    Enterococcus_B faecium       2177
    Streptococcus pyogenes       2144
    Neisseria meningitidis       2138
    Listeria monocytogenes       1977
    Campylobacter_D jejuni       1918
    Enterococcus faecalis        1902
    Enterobacter hormaechei_A    1867
    Mycobacterium abscessus      1820
    Burkholderia mallei          1780
    Listeria monocytogenes_B     1699
    Vibrio parahaemolyticus      1553
    

### Taxon history of Escherichia coli

[csvtk](https://github.com/shenwei356/csvtk) is used to help handle the results.


Get the TaxId:

    $ echo Escherichia coli \
        | taxonkit name2taxid --data-dir gtdb-taxdump/R207/
    Escherichia coli        1945799576

Any changes in the past? Hmm, of cause, it appeared in R80. 

    $ zcat gtdb-taxid-changelog.csv.gz \
        | csvtk grep -f taxid -p 1945799576 \
        | csvtk cut -f -lineage-taxids \
        | csvtk csv2md

|taxid     |version|change|change-value                             |name            |rank   |lineage                                                                                                     |
|:---------|:------|:-----|:----------------------------------------|:---------------|:------|:-----------------------------------------------------------------------------------------------------------|
|1945799576|R080   |NEW   |                                         |Escherichia coli|species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia coli|
|1945799576|R207   |ABSORB|209923990;721309725;1733194824;1765437261|Escherichia coli|species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia coli|

And it `absorb`s four taxa in R207, let's see what happened to them:
    
    $ zcat gtdb-taxid-changelog.csv.gz \
        | csvtk grep -f taxid -p 209923990,721309725,1733194824,1765437261 \
        | csvtk cut -f -lineage-taxids \
        | csvtk csv2md


|taxid     |version|change|change-value         |name                   |rank   |lineage                                                                                                            |
|:---------|:------|:-----|:--------------------|:----------------------|:------|:------------------------------------------------------------------------------------------------------------------|
|209923990 |R089   |NEW   |                     |Escherichia coli_C     |species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia coli_C     |
|209923990 |R089   |ABSORB|1258663139;1303135559|Escherichia coli_C     |species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia coli_C     |
|209923990 |R207   |MERGE |1945799576           |Escherichia coli_C     |species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia coli_C     |
|721309725 |R089   |NEW   |                     |Escherichia coli_D     |species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia coli_D     |
|721309725 |R207   |MERGE |1945799576           |Escherichia coli_D     |species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia coli_D     |
|1733194824|R089   |NEW   |                     |Escherichia dysenteriae|species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia dysenteriae|
|1733194824|R207   |MERGE |1945799576           |Escherichia dysenteriae|species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia dysenteriae|
|1765437261|R089   |NEW   |                     |Escherichia flexneri   |species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia flexneri   |
|1765437261|R207   |MERGE |1945799576           |Escherichia flexneri   |species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia flexneri   |

Yes, *Escherichia flexneri* is merged into *Escherichia coli* as [reported in the release note of R207](https://forum.gtdb.ecogenomic.org/t/announcing-gtdb-r07-rs207/264).

We can also check the history of an *Escherichia flexneri* assembly. Listing assemblies:

    $ taxonkit list --data-dir gtdb-taxdump/R202/ --ids 1765437261 -n -r -I "" \
        | head -n 5
    1765437261 [species] Escherichia flexneri
    23859 [no rank] 013185635
    292350 [no rank] 002736085
    344832 [no rank] 000358285
    390840 [no rank] 001748545

E.g., the taxon node `013185635` (taxid `23859`). Let's check the history via t he TaxId:

    $ zcat gtdb-taxid-changelog.csv.gz \
        | csvtk grep -f taxid -p 23859 \
        | csvtk cut -f -lineage-taxids \
        | csvtk csv2md
        
|taxid|version|change        |change-value|name     |rank   |lineage                                                                                                                   |
|:----|:------|:-------------|:-----------|:--------|:------|:-------------------------------------------------------------------------------------------------------------------------|
|23859|R202   |NEW           |            |013185635|no rank|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia flexneri;013185635|
|23859|R207   |CHANGE_LIN_TAX|            |013185635|no rank|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia coli;013185635    |

Note that we removed the prefix (`GCA_` and `GCF_`) and version number (see method).
So the original assembly accession should be `GCA_013185635.X`, which can be found in `taxid.map` file:

    $ cat gtdb-taxdump/R207/taxid.map \
        | csvtk grep -Ht -f 2 -p 23859
    GCF_013185635.1 23859

The [GCA_013185635.1](https://gtdb.ecogenomic.org/genome?gid=GCA_013185635.1) page
also shows the taxonomic information of current version (R207) and the taxon history:

|Release|Domain     |Phylum           |Class                 |Order              |Family               |Genus         |Species                |
|:------|:----------|:----------------|:---------------------|:------------------|:--------------------|:-------------|:----------------------|
|R207   |d__Bacteria|p__Proteobacteria|c__Gammaproteobacteria|o__Enterobacterales|f__Enterobacteriaceae|g__Escherichia|s__Escherichia coli    |
|R202   |d__Bacteria|p__Proteobacteria|c__Gammaproteobacteria|o__Enterobacterales|f__Enterobacteriaceae|g__Escherichia|s__Escherichia flexneri|

### Species of the genus Escherichia
    
    # set the direcotory of taxdump file
    export TAXONKIT_DB=gtdb-taxdump/R207
    
    $ echo Escherichia | taxonkit name2taxid 
    Escherichia     1187493883

    $ taxonkit list --ids 1187493883 -I "" \
        | taxonkit filter  -E species \
        | taxonkit lineage -Lnr \
        | tee Escherichia.tsv
    205980079       Escherichia ruysiae     species
    266095079       Escherichia marmotae    species
    371972804       Escherichia sp002965065 species
    525903441       Escherichia coli_E      species
    731033585       Escherichia albertii    species
    924696227       Escherichia fergusonii  species
    1474290498      Escherichia sp004211955 species
    1673233649      Escherichia sp005843885 species
    1945799576      Escherichia coli        species
    2073917181      Escherichia sp001660175 species
    
    $ csvtk join -Ht Escherichia.tsv \
        <(cut -f 1 Escherichia.tsv \
            | rush 'echo -ne "{}\t$(taxonkit list --ids {} -I "" \
            | taxonkit filter -L species | wc -l)\n"') \
        | csvtk add-header -t -n "taxid,name,rank,#assembly" \
        | csvtk sort -t -k "#assembly:nr" -k name \
        | csvtk csv2md -t
        
|taxid     |name                   |rank   |#assembly|
|:---------|:----------------------|:------|:--------|
|1945799576|Escherichia coli       |species|26859    |
|731033585 |Escherichia albertii   |species|107      |
|266095079 |Escherichia marmotae   |species|82       |
|924696227 |Escherichia fergusonii |species|77       |
|1673233649|Escherichia sp005843885|species|37       |
|205980079 |Escherichia ruysiae    |species|36       |
|2073917181|Escherichia sp001660175|species|3        |
|1474290498|Escherichia sp004211955|species|2        |
|525903441 |Escherichia coli_E     |species|1        |
|371972804 |Escherichia sp002965065|species|1        |

What's the *Escherichia coli_E*? There's only one genome: [GCF_011881725.1](https://gtdb.ecogenomic.org/genome?gid=GCF_011881725.1)

    $ taxonkit list --ids 525903441 -nr 
    525903441 [species] Escherichia coli_E
      1744010345 [no rank] 011881725

    $ grep 1744010345 gtdb-taxdump/R207/taxid.map 
    GCF_011881725.1 1744010345

### Common manipulations

Except the four taxdump files, we provide a `taxid.map` file which maps genome accessions to TaxIds.

    $ wc -l gtdb-taxdump/R207/*
     14936 gtdb-taxdump/R207/delnodes.dmp
      1530 gtdb-taxdump/R207/merged.dmp
    401815 gtdb-taxdump/R207/names.dmp
    401815 gtdb-taxdump/R207/nodes.dmp
       107 gtdb-taxdump/R207/ranks.txt
    317542 gtdb-taxdump/R207/taxid.map


List all the genomes of a species, e.g., *Akkermansia muciniphila*,

    # Retreive the TaxId
    $ echo Akkermansia muciniphila | taxonkit name2taxid --data-dir gtdb-taxdump/R207
    Akkermansia muciniphila 415593052
    
    # list subtree
    $ taxonkit list --data-dir gtdb-taxdump/R207 -nr --ids  415593052 | head -n 5
    2563076700 [species] Akkermansia muciniphila
      54773322 [no rank] 002885595
      56256420 [no rank] 004015265
      78545007 [no rank] 002885335
      81917184 [no rank] 002885695
    
    # mapping TaxIds to Genome accessions with taxid.map
    $ taxonkit list --data-dir gtdb-taxdump/R207 -I "" --ids  415593052 \
        | csvtk join -Ht -f '1;2' - gtdb-taxdump/R207/taxid.map \
        | head -n 5
    54773322        GCF_002885595.1
    56256420        GCF_004015265.1
    78545007        GCF_002885335.1
    81917184        GCF_002885695.1
    88269675        GCF_008423215.1
    
Find the history of a taxon using scientific name:

    $ zcat gtdb-taxid-changelog.csv.gz \
        | csvtk grep -f name -i -r -p "Escherichia dysenteriae" \
        | csvtk cut -f -lineage,-lineage-taxids \
        | csvtk csv2md
    |taxid     |version|change|change-value|name                   |rank   |
    |:---------|:------|:-----|:-----------|:----------------------|:------|
    |1733194824|R089   |NEW   |            |Escherichia dysenteriae|species|
    |1733194824|R207   |MERGE |1945799576  |Escherichia dysenteriae|species|
    
    
    # another example
    $ zcat gtdb-taxid-changelog.csv.gz \
        | csvtk grep -f name -i -r -p "Escherichia coli" \
        | csvtk cut -f -lineage,-lineage-taxids \
        | csvtk csv2md
    |taxid     |version|change|change-value                             |name              |rank   |
    |:---------|:------|:-----|:----------------------------------------|:-----------------|:------|
    |209923990 |R089   |NEW   |                                         |Escherichia coli_C|species|
    |209923990 |R089   |ABSORB|1258663139;1303135559                    |Escherichia coli_C|species|
    |209923990 |R207   |MERGE |1945799576                               |Escherichia coli_C|species|
    |525903441 |R202   |NEW   |                                         |Escherichia coli_E|species|
    |721309725 |R089   |NEW   |                                         |Escherichia coli_D|species|
    |721309725 |R207   |MERGE |1945799576                               |Escherichia coli_D|species|
    |1258663139|R086   |NEW   |                                         |Escherichia coli_B|species|
    |1258663139|R089   |MERGE |209923990                                |Escherichia coli_B|species|
    |1303135559|R080   |NEW   |                                         |Escherichia coli_A|species|
    |1303135559|R089   |MERGE |209923990                                |Escherichia coli_A|species|
    |1945799576|R080   |NEW   |                                         |Escherichia coli  |species|
    |1945799576|R207   |ABSORB|209923990;721309725;1733194824;1765437261|Escherichia coli  |species|


Check more [TaxonKit commands and usages](https://bioinf.shenwei.me/taxonkit/usage/).

## Known issues

Note: the TaxIds below may be not the lastest (taxonkit v0.14.0 save TaxIds in `int32` instead of `uint32`).

### Inaccurate delnodes.dmp and merged.dmp for a few taxa with same names

In old versions, some taxa had the same names, e.g., `1-14-0-10-36-11`.
    
    # r86.2
    
    # taxid of 1-14-0-10-36-11: 3509163818
    GB_GCA_002762845.1	d__Archaea;p__Nanoarchaeota;c__Woesearchaeia;o__GW2011-AR9;f__GW2011-AR9;g__1-14-0-10-36-11;s__    
    
    # taxid of 1-14-0-10-36-11: 3509163819
    GB_GCA_002778535.1	d__Bacteria;p__Patescibacteria;c__ABY1;o__Kuenenbacterales;f__UBA2196;g__1-14-0-10-36-11;s__
    
Later in r89, the Archaea genus `1-14-0-10-36-11` was renamed,
while `taxid 3509163818` was assigned to Bacteria genus `1-14-0-10-36-11` and `taxid 3509163819` was marked in `delnodes.dmp`.

    # genus changed, and assigned a new species
    GB_GCA_002762845.1	d__Archaea;p__Nanoarchaeota;c__Nanoarchaeia;o__Woesearchaeales;f__GW2011-AR9;g__PCYB01;s__PCYB01 sp002762845
    
    # assigned a new species
    # taxid of 1-14-0-10-36-11: 3509163818
    GB_GCA_002778535.1	d__Bacteria;p__Patescibacteria;c__ABY1;o__UBA2196;f__UBA2196;g__1-14-0-10-36-11;s__1-14-0-10-36-11 sp002778535
    
As a result, the taxid-changelog showed:

    $ zcat gtdb-taxid-changelog.csv.gz | csvtk grep -f taxid -p 3509163818
    taxid,version,change,change-value,name,rank,lineage,lineage-taxids
    3509163818,R086,NEW,,1-14-0-10-36-11,genus,Archaea;Nanoarchaeota;Woesearchaeia;GW2011-AR9;1-14-0-10-36-11,2587168575;2246723321;236669313;1472230377;3509163818
    3509163818,R089,CHANGE_LIN_TAX,,1-14-0-10-36-11,genus,Bacteria;Patescibacteria;ABY1;UBA2196;1-14-0-10-36-11,609216830;741652572;2027207876;1322712682;3509163818

    $ zcat gtdb-taxid-changelog.csv.gz | csvtk grep -f taxid -p 3509163819
    taxid,version,change,change-value,name,rank,lineage,lineage-taxids
    3509163819,R086,NEW,,1-14-0-10-36-11,genus,Bacteria;Patescibacteria;ABY1;Kuenenbacterales;UBA2196;1-14-0-10-36-11,609216830;741652572;2027207876;2441366341;1322712682;3509163819
    3509163819,R089,DELETE,,1-14-0-10-36-11,genus,Bacteria;Patescibacteria;ABY1;Kuenenbacterales;UBA2196;1-14-0-10-36-11,609216830;741652572;2027207876;2441366341;1322712682;3509163819

### Unstable delnodes.dmp and merged.dmp for a few taxa of which genomes are mreged into different taxa

An example: In R95, some (_Sphingobium japonicum_A_) genomes ([GCF_000445085.1](https://gtdb.ecogenomic.org/genome?gid=GCF_000445085.1))
were merged into (*Sphingobium chinhatense*), while others ([GCF_000091125.1](https://gtdb.ecogenomic.org/genome?gid=GCF_000091125.1)) 
into *Sphingobium indicum*. Check [details](https://github.com/shenwei356/gtdb-taxdump/issues/2#issuecomment-1233655355)
    
## Merging GTDB and NCBI taxonomy

- If you needs the taxdump files and the `taxid.map` file mapping genome assembly accesions to TaxIds, please follow
  [Merging the GTDB taxonomy (for prokaryotic genomes from GTDB) and NCBI taxonomy (for genomes from NCBI)](https://bioinf.shenwei.me/kmcp/database/#merging-gtdb-and-ncbi-taxonomy).
- If you just need the taxdump files, please follow [Merging GTDB and NCBI taxonomy](https://bioinf.shenwei.me/taxonkit/tutorial/#merging-gtdb-and-ncbi-taxonomy).


## Citation

> Shen, W., Ren, H., TaxonKit: a practical and efficient NCBI Taxonomy toolkit,
> Journal of Genetics and Genomics, [https://doi.org/10.1016/j.jgg.2021.03.006](https://www.sciencedirect.com/science/article/pii/S1673852721000837) [![Citation Badge](https://api.juleskreuer.eu/citation-badge.php?doi=10.1016/j.jgg.2021.03.006)](https://scholar.google.com/citations?view_op=view_citation&hl=en&user=wHF3Lm8AAAAJ&citation_for_view=wHF3Lm8AAAAJ:ULOm3_A8WrAC)

## Contributing

We welcome pull requests, bug fixes and issue reports.

## License

[MIT License](https://github.com/shenwei356/gtdb-taxdump/blob/master/LICENSE)

## Similar tools

- [gtdb_to_taxdump](https://github.com/nick-youngblut/gtdb_to_taxdump), Convert GTDB taxonomy to NCBI taxdump format
