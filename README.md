# GTDB taxonomy taxdump files with trackable TaxIds

Metagenomic tools like [Kraken2](https://github.com/DerrickWood/kraken2),
 [Centrifuge](https://github.com/DaehwanKimLab/centrifuge)
 and [KMCP](https://github.com/shenwei356/kmcp) support NCBI taxonomy in format of [NCBI taxdump files](https://ftp.ncbi.nlm.nih.gov/pub/taxonomy/).
[GTDB](https://gtdb.ecogenomic.org/), a prokaryotic genomes catalogue, has its own taxonomy data.
Though the genomes, derived from GenBank and RefSeq, can be mappped to NCBI taxonomy TaxIds,
there's an urgent need to create its own taxonomy taxdump files with ***stable and trackable*** TaxIds.

A [TaxonKit](https://github.com/shenwei356/taxonkit) command, `taxonkit create-taxdump` is created
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
using [xxhash](https://github.com/cespare/xxhash/) and convert it to `uint32`.

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
    ????????? R080
    ??????? ????????? bac_taxonomy_r80.tsv
    ????????? R083
    ??????? ????????? bac_taxonomy_r83.tsv
    ????????? R086
    ??????? ????????? ar122_taxonomy_r86.2.tsv
    ??????? ????????? bac120_taxonomy_r86.2.tsv
    ????????? R089
    ??????? ????????? ar122_taxonomy_r89.tsv
    ??????? ????????? bac120_taxonomy_r89.tsv
    ????????? R095
    ??????? ????????? ar122_taxonomy_r95.tsv.gz
    ??????? ????????? bac120_taxonomy_r95.tsv.gz
    ????????? R202
    ??????? ????????? ar122_taxonomy_r202.tsv.gz
    ??????? ????????? bac120_taxonomy_r202.tsv.gz
    ????????? R207
        ????????? ar53_taxonomy_r207.tsv.gz
        ????????? bac120_taxonomy_r207.tsv.gz

[TaxonKit](https://github.com/shenwei356/taxonkit) v0.12.0 or later version is needed.

### Steps
    
1. Generating taxdump files for the first version r80:

        taxonkit create-taxdump taxonomy/R080/*.tsv* --gtdb --out-dir gtdb-taxdump/R080 --force
        15:19:59.816 [WARN] --gtdb-re-subs failed to extract ID for subspecies, the origninal value is used instead. e.g., UBA11420
        15:19:59.964 [INFO] 94759 records saved to gtdb-taxdump/R080/taxid.map
        15:20:00.011 [INFO] 110345 records saved to gtdb-taxdump/R080/nodes.dmp
        15:20:00.048 [INFO] 110345 records saved to gtdb-taxdump/R080/names.dmp
        15:20:00.048 [INFO] 0 records saved to gtdb-taxdump/R080/merged.dmp
        15:20:00.048 [INFO] 0 records saved to gtdb-taxdump/R080/delnodes.dmp
    
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
    Escherichia coli        4093283224

Any changes in the past? Hmm, of cause, it appeared in R80. 

    $ zcat gtdb-taxid-changelog.csv.gz \
        | csvtk grep -f taxid -p 4093283224 \
        | csvtk cut -f -lineage-taxids \
        | csvtk csv2md

|taxid     |version|change|change-value                               |name            |rank   |lineage                                                                                                     |
|:---------|:------|:-----|:------------------------------------------|:---------------|:------|:-----------------------------------------------------------------------------------------------------------|
|4093283224|R080   |NEW   |                                           |Escherichia coli|species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia coli|
|4093283224|R207   |ABSORB|1733194824;2357407638;2868793373;3912920909|Escherichia coli|species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia coli|

And it `absorb`s four taxa in R207, let's see what happened to them:
    
    $ zcat gtdb-taxid-changelog.csv.gz \
        | csvtk grep -f taxid -p 1733194824,2357407638,2868793373,3912920909 \
        | csvtk cut -f -lineage-taxids \
        | csvtk csv2md


|taxid     |version|change|change-value         |name                   |rank   |lineage                                                                                                            |
|:---------|:------|:-----|:--------------------|:----------------------|:------|:------------------------------------------------------------------------------------------------------------------|
|1733194824|R089   |NEW   |                     |Escherichia dysenteriae|species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia dysenteriae|
|1733194824|R207   |MERGE |4093283224           |Escherichia dysenteriae|species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia dysenteriae|
|2357407638|R089   |NEW   |                     |Escherichia coli_C     |species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia coli_C     |
|2357407638|R089   |ABSORB|1258663139;3450619207|Escherichia coli_C     |species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia coli_C     |
|2357407638|R207   |MERGE |4093283224           |Escherichia coli_C     |species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia coli_C     |
|2868793373|R089   |NEW   |                     |Escherichia coli_D     |species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia coli_D     |
|2868793373|R207   |MERGE |4093283224           |Escherichia coli_D     |species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia coli_D     |
|3912920909|R089   |NEW   |                     |Escherichia flexneri   |species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia flexneri   |
|3912920909|R207   |MERGE |4093283224           |Escherichia flexneri   |species|Bacteria;Proteobacteria;Gammaproteobacteria;Enterobacterales;Enterobacteriaceae;Escherichia;Escherichia flexneri   |

Yes, *Escherichia flexneri* is merged into *Escherichia coli* as [reported in the release note of R207](https://forum.gtdb.ecogenomic.org/t/announcing-gtdb-r07-rs207/264).

We can also check the history of an *Escherichia flexneri* assembly. Listing assemblies:

    $ taxonkit list --data-dir gtdb-taxdump/R202/ --ids 3912920909 -n -r -I "" \
        | head -n 5
    3912920909 [species] Escherichia flexneri
    23859 [no rank] 013185635
    292350 [no rank] 002736085
    344832 [no rank] 000358285
    660349 [no rank] 001441345

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
    
    $ taxonkit list --ids 3334977531 -I "" \
        | taxonkit filter  -E species \
        | taxonkit lineage -Lnr \
        | tee Escherichia.tsv
    205980079       Escherichia ruysiae     species
    266095079       Escherichia marmotae    species
    1474290498      Escherichia sp004211955 species
    2519456452      Escherichia sp002965065 species
    2673387089      Escherichia coli_E      species
    2878517233      Escherichia albertii    species
    3072179875      Escherichia fergusonii  species
    3820717297      Escherichia sp005843885 species
    4093283224      Escherichia coli        species
    4221400829      Escherichia sp001660175 species
    
    $ csvtk join -Ht Escherichia.tsv \
        <(cut -f 1 Escherichia.tsv \
            | rush 'echo -ne "{}\t$(taxonkit list --ids {} -I "" \
            | taxonkit filter -L species | wc -l)\n"') \
        | csvtk add-header -t -n "taxid,name,rank,#assembly" \
        | csvtk sort -t -k "#assembly:nr" -k name \
        | csvtk csv2md -t
        
|taxid     |name                   |rank   |#assembly|
|:---------|:----------------------|:------|:--------|
|4093283224|Escherichia coli       |species|26859    |
|2878517233|Escherichia albertii   |species|107      |
|266095079 |Escherichia marmotae   |species|82       |
|3072179875|Escherichia fergusonii |species|77       |
|3820717297|Escherichia sp005843885|species|37       |
|205980079 |Escherichia ruysiae    |species|36       |
|4221400829|Escherichia sp001660175|species|3        |
|1474290498|Escherichia sp004211955|species|2        |
|2673387089|Escherichia coli_E     |species|1        |
|2519456452|Escherichia sp002965065|species|1        |

What's the *Escherichia coli_E*? There's only one genome: [GCF_011881725.1](https://gtdb.ecogenomic.org/genome?gid=GCF_011881725.1)

    $ taxonkit list --ids 2673387089 -nr 
    2673387089 [species] Escherichia coli_E
      1744010345 [no rank] 011881725

    $ grep 1744010345 gtdb-taxdump/R207/taxid.map 
    GCF_011881725.1 1744010345

### Common manipulations

Except the four taxdump files, we provide a `taxid.map` file which maps genome accessions to TaxIds.

    $ wc -l gtdb-taxdump/R207/*
     14787 gtdb-taxdump/R207/delnodes.dmp
      6483 gtdb-taxdump/R207/merged.dmp
    401815 gtdb-taxdump/R207/names.dmp
    401815 gtdb-taxdump/R207/nodes.dmp
    317542 gtdb-taxdump/R207/taxid.map


List all the genomes of a species, e.g., *Akkermansia muciniphila*,

    # Retreive the TaxId
    $ echo Akkermansia muciniphila | taxonkit name2taxid --data-dir gtdb-taxdump/R207
    Akkermansia muciniphila 2563076700
    
    # list subtree
    $ taxonkit list --data-dir gtdb-taxdump/R207 -nr --ids  2563076700 | head -n 5
    2563076700 [species] Akkermansia muciniphila
      54773322 [no rank] 002885595
      56256420 [no rank] 004015265
      78545007 [no rank] 002885335
      101987851 [no rank] 004015245
    
    # mapping TaxIds to Genome accessions with taxid.map
    $ taxonkit list --data-dir gtdb-taxdump/R207 -I "" --ids  2563076700 \
        | csvtk join -Ht -f '1;2' - gtdb-taxdump/R207/taxid.map \
        | head -n 5
    54773322        GCF_002885595.1
    56256420        GCF_004015265.1
    78545007        GCF_002885335.1
    101987851       GCF_004015245.1
    138593819       GCF_010223575.1

Check more [TaxonKit commands and usages](https://bioinf.shenwei.me/taxonkit/usage/).

## Known issues

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
    
## Citation

> Shen, W., Ren, H., TaxonKit: a practical and efficient NCBI Taxonomy toolkit,
> Journal of Genetics and Genomics, [https://doi.org/10.1016/j.jgg.2021.03.006](https://www.sciencedirect.com/science/article/pii/S1673852721000837)

## Contributing

We welcome pull requests, bug fixes and issue reports.

## License

[MIT License](https://github.com/shenwei356/gtdb-taxdump/blob/master/LICENSE)

## Similar tools

- [gtdb_to_taxdump](https://github.com/nick-youngblut/gtdb_to_taxdump), Convert GTDB taxonomy to NCBI taxdump format
