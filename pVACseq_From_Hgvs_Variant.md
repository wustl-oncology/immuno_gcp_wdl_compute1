### Run pVACseq on a single variant in HGVS format

#### Goal. Feed a single variant, provided by "name" only, into pVACseq

e.g. ERBB3 p.S846I

In this version the pVACseq analysis is focused on the variant only, to cover the use case where we may not even have access to tumor normal exome and tumor RNAseq data (and thus have no BAMs, no phased VCF, etc.).

#### Step 1. Use the [ClinGen Allele Registry](https://reg.clinicalgenome.org/redmine/projects/registry/genboree_registry/landing) to resolve it to HGVS

This works out to:

NC_000012.12:g.56097861G>T / NM_001982.4:c.2537G>T / NP_001973.2:p.Ser846Ile
NC_000012.12:g.56097861G>T / ENST00000267101.8:c.2537G>T / ENSP00000267101.4:p.Ser846Ile 

Create a simple text file: erbb3-variant-hgvs.txt with a single HGVS g. entry: `NC_000012.12:g.56097861G>T`

#### Step 2. Use VEP to create an annotated VCF - using the HGVS format as input

docker: "mgibio/vep_helper-cwl:vep_105.0_v1"

`isub -m 64 -n 8 --preserve false -i 'mgibio/vep_helper-cwl:vep_105.0_v1'`

Using the docker image above, annotate the variant with Ensembl VEP as follows

```
/usr/bin/perl -I /opt/lib/perl/VEP/Plugins /usr/bin/variant_effect_predictor.pl \
--format hgvs \
--vcf \
--fork 4 \
--term SO \
--transcript_version \
--cache \
--symbol \
-o annotated.vcf \
-i /storage1/fs1/mgriffit/Active/JLF_MCDB/cases/mcdb022/erbb3/erbb3-pvacseq/erbb3-variant-hgvs.txt \
--synonyms /storage1/fs1/mgriffit/Active/griffithlab/pipeline_test/malachi/refs_backup/May-2023/griffith-lab-workflow-inputs/human_GRCh38_ens105/reference_genome/chromAlias.ensembl.txt \
--flag_pick \
--dir /storage1/fs1/mgriffit/Active/griffithlab/pipeline_test/malachi/refs_backup/May-2023/griffith-lab-workflow-inputs/human_GRCh38_ens105/vep_cache \
--fasta /storage1/fs1/mgriffit/Active/griffithlab/pipeline_test/malachi/refs_backup/May-2023/griffith-lab-workflow-inputs/human_GRCh38_ens105/aligner_indices/bwamem2_2.2.1/all_sequences.fa \
--check_existing \
--plugin Frameshift --plugin Wildtype  \
--everything \
--assembly GRCh38 \
--cache_version 105 \
--species homo_sapiens
```

#### Step 3. Add tumor/normal genotype information to the VCF

`isub -m 64 -n 8 --preserve false -i 'griffithlab/vatools:latest'`

Using the docker image above, use VA tools to add genotype columns to the VCF

```
vcf-genotype-annotator /storage1/fs1/mgriffit/Active/JLF_MCDB/cases/mcdb022/erbb3/erbb3-pvacseq/annotated.vcf JLF-100-016-tumor 0/1 -o /storage1/fs1/mgriffit/Active/JLF_MCDB/cases/mcdb022/erbb3/erbb3-pvacseq/annotated.genotyped.1.vcf

vcf-genotype-annotator /storage1/fs1/mgriffit/Active/JLF_MCDB/cases/mcdb022/erbb3/erbb3-pvacseq/annotated.genotyped.1.vcf JLF-100-016-normal 0/0 -o /storage1/fs1/mgriffit/Active/JLF_MCDB/cases/mcdb022/erbb3/erbb3-pvacseq/annotated.genotyped.2.vcf
```

#### Step 4. Run pVACseq on the genotyped VCF

`isub -m 64 -n 8 --preserve false -i 'susannakiwala/pvactools:latest'`

Using the docker image above, use pVACseq to perform neoantigen analysis on the VCF

```
export HLA_ALLELES="HLA-A*01:01,HLA-A*02:13,HLA-B*08:01,HLA-B*57:03,HLA-C*07:01,HLA-C*06:02"
export PEPTIDE_FASTA=/storage1/fs1/gillandersw/Active/Project_0001_Clinical_Trials/annotation_files_for_review/Homo_sapiens.GRCh38.pep.all.fa.gz

pvacseq run /storage1/fs1/mgriffit/Active/JLF_MCDB/cases/mcdb022/erbb3/erbb3-pvacseq/annotated.genotyped.2.vcf \
            JLF-100-016-tumor \
            $HLA_ALLELES \
            all /storage1/fs1/mgriffit/Active/JLF_MCDB/cases/mcdb022/erbb3/erbb3-pvacseq/pvacseq/ \
            -e1 8,9,10,11 -e2 12,13,14,15,16,17,18 --iedb-install-directory /opt/iedb \
            -b 500  -m median -k -t 8 --run-reference-proteome-similarity \
            --aggregate-inclusion-binding-threshold 1500 \
            --peptide-fasta $PEPTIDE_FASTA \
            -d 100 --normal-sample-name JLF-100-016-normal --problematic-amino-acids C \
            --allele-specific-anchors \
            --maximum-transcript-support-level 1 --pass-only 

```


