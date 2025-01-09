# DNA Vaccine Design

## Running Pipeline

**comment out problematic positions in the YAML file**

## Generating Review Files

This is slightly modified to create a 33mer instead of 51mer. 

```
cd $WORKING_BASE
mkdir ../generate_protein_fasta
cd ../generate_protein_fasta
mkdir candidates
mkdir all

zcat $WORKING_BASE/final_results/annotated.expression.vcf.gz | grep -v "^##" | head -n 1 # Get sample ID Found in the #CHROM header of VCF
export TUMOR_SAMPLE_ID="TWJF-10146-0029-0029_Tumor_Lysate"


bsub -Is -q general-interactive -G $GROUP -a "docker(griffithlab/pvactools:4.0.1)" /bin/bash

pvacseq generate_protein_fasta \
  -p $WORKING_BASE/final_results/pVACseq/phase_vcf/phased.vcf.gz \
  --pass-only --mutant-only -d 150 \
  -s $TUMOR_SAMPLE_ID \
  --aggregate-report-evaluation {Accept,Review} \
  --input-tsv ../itb-review-files/*.tsv  \
  $WORKING_BASE/final_results/annotated.expression.vcf.gz \
  16 \
  $WORKING_BASE/../generate_protein_fasta/candidates/annotated_filtered.vcf-pass-51mer.fa

pvacseq generate_protein_fasta \
  -p $WORKING_BASE/final_results/pVACseq/phase_vcf/phased.vcf.gz \
  --pass-only --mutant-only -d 150 \
  -s $TUMOR_SAMPLE_ID  \
  $WORKING_BASE/final_results/annotated.expression.vcf.gz \
  16  \
  $WORKING_BASE/../generate_protein_fasta/all/annotated_filtered.vcf-pass-51mer.fa

exit
```

## Proceed with Manual review as normal

## Preparing the Final Files After Completed Manual Review

After reviewing all candidates it is time to start preparing the result to be submitted for vaccine design.

1. Remove any candidates that have been rejected from the `annotated_filtered.vcf-candidates-33mer.fa` file.
2. Edit any sequences in the `annotated_filtered.vcf-candidates-33mer.fa` where proximal variants are not accounted for.
3. Run pVACvector to produce the vector result

### Running pVACvector

This step requires a little bit of playing around to get a result sometimes. 

```
cd ../$WORKING_BASE

mkdir pvacvector

cd pvacvector
```

We will start with the most stringent, preferred parameters.

```
mkdir v1
cd v1
```

Create a bash script called `pvacvector_versions.sh` containing the below code
   
```
export HLA_ALLELES="PATIENT-SPECIFIC-HLA-ALLELES"
export BASE_DIR="LOCATION-FOR-RESULTS"
export INPUT_FASTA="annotated_filtered.vcf-candidates-33mer.fa" # final fasta file of neoantigen candidates, e.g. 33-mer in length
export SAMPLE_NAME="hcc1395"

pvacvector run --n-threads 8 \  
 --top-score-metric median --binding-threshold 1000 --percentile-threshold 2 \  
 --class-i-epitope-length 8,9,10,11 \
 --iedb-install-directory /opt/iedb \
 $INPUT_FASTA $SAMPLE_NAME \
 $HLA_ALLELES \
 'all_class_i' \
 $BASE_DIR/vaccine-vector_median_b1000_p2_33mer
```

Execute the bash script using this docker command:
```
bsub -M 64G -G compute-oncology -n 8 -R 'select[mem>64G] rusage[mem=64G]' -q oncology -a 'docker(griffithlab/pvactools:4.0.6)' \
-oo $WORKING_BASE/pvacvector/v1/pvacvector.stdout \
-eo $WORKING_BASE/pvacvector/v1/pvacvector.stderr \
/bin/bash $WORKING_BASE/pvacvector/v1/pvacvector_versions.sh
```

#### Expected Results

We are looking for pVACvector to make a DNA vector where all peptides are combined into one 
molecule minimizing the effects of junctional epitopes (that may create novel peptides) between the sequences. 
pVACvector inserts spacer amino acid sequences to minimize the binding score of junction epitopes.
We are looking for an overall molecule that has poor binding.

An example result which would be found in the `SAMPLE_results.dna.fa` folder is:
```
>MT.193.HCN2.ENST00000251287.3.missense.429G/E,MT.145.EDNRB.ENST00000377211.8.FS.399CA/C,AAY,MT.106.PTEN.ENST00000371953.8.missense.53V/A,AAY,MT.167.CYFIP1.ENST00000610365.4.missense.1217E/K,MT.111.TMEM151A.ENST00000327259.5.missense.147P/L,HHHH,MT.43.RAB24.ENST00000303251.11.missense.173V/I,MT.19.HHAT.ENST00000426968.2.missense.34T/K,MT.78.CNTNAP2.ENST00000361727.8.missense.55S/Y,HHHH,MT.6.BCAR3.ENST00000260502.11.missense.670R/W,HHHH,MT.40.ATP10B.ENST00000327245.10.missense.726A/T,HHHH,MT.198.C19orf38.ENST00000397820.5.missense.208R/W,MT.203.IL4I1.ENST00000341114.7.missense.500F/S,MT.14.MAGI3.ENST00000369617.8.missense.301M/V

|Median_Junction_Score:1698.0511666666664
|Lowest_Junction_Score:1017.05
|All_Junction_Scores:1086.671,1216.514,1017.05,1279.169,3133.529,1145.59,1433.67,2089.98,2440.689,2025.29,2443.191,1065.271

CTGTACAGCTTCGCCCTGTTCAAGGCCATGAGCCACATGCTGTGCATCGAGTACGGCAGA
CAGGCCCCCGAGAGCATGACCGACATCTGGCTGACCATGTACACCCTGATGACCTGCGAG
ATGCTGAGAAAGAAGAGCGGCATGCAGATGCTGGCCGCCTACTTCCCCGCCGAGAGACTG
GAGGGCGTGTACAGAAACAACATCGACGACGCCGTGAGATTCCTGGACAGCAAGCACAAG
AACCACTACAAGATCTACAACGCCGCCTACCTGAAGAAGATGGTGGAGAGAATCAGAAAG
TTCCAGATCCTGAACGACAAGATCATCACCATCCTGGACAAGTACCTGAAGAGCGGCGAC
GGCGAGGGCGACGCCCACACCGTGCTGGCCCTGATCAGAAGACTGCAGCAGGCCCCCCTG
TGCGTGTGGTGGAAGGCCACCAGCTACCACTACGTGAGAAGAACCAGACACCACCACCAC
ACCGGCCAGAGCGTGGACGAGCTGTTCCAGAAGGTGGCCGAGGACTACATCAGCGTGGCC
GCCTTCCAGGTGATGACCGAGGACAAGGGCGTGGACCTGGTGAGCCAGATGGCCACCCTG
CTGGCCAGAAAGAGAAGATGGTACAAGAAGGAGAACGAGTACTACCTGCTGCAGTTCACC
CTGACCGTGAGATGCCTGCTGGTGAGCGGCCTGCCCCACGTGGCCTTCAGCAGCAGCAGC
AGCATCTACGGCAGCTACAGCCCCGGCTACGCCAAGATCAACAAGAGAGGCGGCGCCCAC
CACCACCACCTGGAGATGCCCCAGATCACCAGACTGGAGAAGACCTGGACCGCCCTGTGG
CACCAGTACACCCAGACCGCCATCCTGTACGAGAAGCAGCTGAAGCCCCACCACCACCAC
CTGGCCAGACCCGAGTTCTGCTACGAGGCCGAGAGCCCCGACGAGGCCACCCTGGTGCAC
GCCGCCCACGCCTACAGCTTCACCCTGGTGAGCAGAACCCACCACCACCACCTGGACGAC
CACAGCGGCACCACCGCCACCCCCAGCAACAGCAGAACCTGGAAGAGACCCACCAGCACC
AGCAGCAGCCCCGAGACCCCCGAGTTCAGCTGGCAGACCGAGAAGGACGACTGGACCGTG
CCCTACGGCAGAATCTACAGCGCCGGCGAGCACACCGCCTACCCCCACGGCTGGGTGGAG
ACCGCCGTGTACATGATGAGAGACGAGACCCTGGAGCCCCTGCCCAAGAACTGGGAGGTG
GCCTACACCGACACCGGCATGATCTACTTCATCGACCACAACACCAAG
```
It listed the candidates which it has been vectorized. Then the median and lowest junction score, which in this case we expected to be over 1000,
and all the individual junction scores. For further explanation read the pVACvector [documentation](https://pvactools.readthedocs.io/en/latest/pvacvector.html).
**There is a really useful graphic on the [Output Files Page](https://pvactools.readthedocs.io/en/latest/pvacvector/output_files.html).**

The long sequence is our final DNA vector and the contents of this file should be pasted into the genomics report and the classI peptides colored red to more visually indicate the,

<img width="493" alt="Example of DNA Vector in Genomic Report" src="https://github.com/user-attachments/assets/68cba798-d09f-4ea2-ab49-ad658486fdfd" />


#### Running Different Versions of pVACvector
Running pVACvector with these parameters might be too strict and so no DNA Vector can be created. 
Here is an example of a bunch of other versions of commands we have tried (you might have to play with the parameters to get results!)

```
export HLA_ALLELES="HLA-A*32:01,HLA-A*26:01,HLA-B*35:02,HLA-B*37:01,HLA-C*04:01,HLA-C*06:02"
export WORK_DIR="/storage1/fs1/gpdunn/Active/Project_0001_Clinical/gc2609/gbm_antigen_dunn/vaccine_G110/pvacvector/v4.0.6"
export INPUT_FASTA="/storage1/fs1/gpdunn/Active/Project_0001_Clinical/gc2609/gbm_antigen_dunn/vaccine_G110/generate_protein_fasta/33-mer/region1/candidates/annotated_filtered.vcf-candidates-33mer.fa"

export BASE_DIR="${WORK_DIR}/v1"
mkdir $BASE_DIR

# PREFERED METHOD: defaults with binding threshold 1000
pvacvector run --n-threads 8 \  
 --top-score-metric median --binding-threshold 1000 --percentile-threshold 2 \  
 --class-i-epitope-length 8,9,10,11 \
 --iedb-install-directory /opt/iedb \
 $INPUT_FASTA $SAMPLE_NAME \
 $HLA_ALLELES \
 'all_class_i' \
 $BASE_DIR/vaccine-vector_median_b1000_p2_33mer

export BASE_DIR="${WORK_DIR}/v2"
mkdir $BASE_DIR

#default pVACvector parameters (binding threshold 500, no percentile...)
pvacvector run --n-threads 8 \  
 --top-score-metric median \  
 --class-i-epitope-length 8,9,10,11 \
 --iedb-install-directory /opt/iedb \
 $INPUT_FASTA $SAMPLE_NAME \
 $HLA_ALLELES \
 'all_class_i' \
 $BASE_DIR/vaccine-vector_defaults_33mer


export BASE_DIR="${WORK_DIR}/v3"
mkdir $BASE_DIR

# defaults with binding threshold 1500 and max clip 5
pvacvector run --n-threads 8 \  
 --top-score-metric median -t 16 -b 1500 --max-clip-length 5 \  
 --class-i-epitope-length 8,9,10,11 \
 --iedb-install-directory /opt/iedb \
 $INPUT_FASTA $SAMPLE_NAME \
 $HLA_ALLELES \
 'all_class_i' \
 $BASE_DIR/vaccine-vector_b1500_clip5_33mer

export BASE_DIR="${WORK_DIR}/v4"
mkdir $BASE_DIR

#defaults with binding threshold 1000 allele specific
pvacvector run --n-threads 8 \  
 --top-score-metric median -k -t 16 -b 1000 --allele-specific-binding-thresholds \  
 --class-i-epitope-length 8,9,10,11 \
 --iedb-install-directory /opt/iedb \
 $INPUT_FASTA $SAMPLE_NAME \
 $HLA_ALLELES \
 'all_class_i' \
 $BASE_DIR/vaccine-vector_b1000_allelespecific_33mer
```

