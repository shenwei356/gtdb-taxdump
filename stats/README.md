

    csvtk join -k -f version \
        <(pigz -cd gtdb-taxid-changelog.csv.gz \
        | csvtk grep -f change -p DELETE \
        | csvtk freq -f version) \
        <(pigz -cd gtdb-taxid-changelog.csv.gz \
        | csvtk grep -f change -p NEW \
        | csvtk freq -f version) \
        <(pigz -cd gtdb-taxid-changelog.csv.gz \
        | csvtk grep -f change -p MERGE \
        | csvtk freq -f version) \
        <(pigz -cd gtdb-taxid-changelog.csv.gz \
        | csvtk grep -f change -p REUSE_DEL \
        | csvtk freq -f version) \
        <(pigz -cd gtdb-taxid-changelog.csv.gz \
        | csvtk grep -f change -p REUSE_MER \
        | csvtk freq -f version) \
        | csvtk rename -f -1 -n deleted,newly_added,merged,deleted_reused,merged_reused \
        > changes.csv
        
    cat changes.csv \
        | csvtk gather -k types -v count -f 2-6 \
        > changes.g.csv
        
    ./plot.R
