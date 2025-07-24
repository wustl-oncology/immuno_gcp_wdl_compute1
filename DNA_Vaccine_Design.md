# DNA Vaccine Design

## Running Pipeline

**comment out problematic positions in the YAML file**

## Generating 33-mer after manual review

Download the post manual review file (with the final list of candidates marked as Accept) from Google Drive and rename the downloaded file to end with `(...)_post_manual_review.tsv`. 
Rerun the generate protein fasta commands (This is slightly modified to create a 33mer instead of 51mer).
Note- We will not create the all candidates version of this as the post ITB file does not have all the candidates.

```
cd $WORKING_BASE/../generate_protein_fasta
mkdir 33-mer
cd 33-mer
mkdir candidates
zcat $WORKING_BASE/final_results/annotated.expression.vcf.gz | grep -v "^##" | head -n 1 # Get sample ID Found in the #CHROM header of VCF
export TUMOR_SAMPLE_ID="TWJF-10146-0029-0029_Tumor_Lysate"


bsub -Is -q general-interactive -G $GROUP -a "docker(griffithlab/pvactools:5.3.0)" /bin/bash

  pvacseq generate_protein_fasta \
  -p $WORKING_BASE/final_results/pVACseq/phase_vcf/phased.vcf.gz \
  --pass-only --mutant-only -d 150 \
  -s $TUMOR_SAMPLE_ID \
  --aggregate-report-evaluation "Accept,Review" \
  --input-tsv $WORKING_BASE/../itb-review-files/*_post_manual_review.tsv  \
  $WORKING_BASE/final_results/annotated.expression.vcf.gz \
  16 \
  $WORKING_BASE/../generate_protein_fasta/33-mer/candidates/annotated_filtered.vcf-pass-33mer.fa

exit
```

After reviewing all candidates it is time to start preparing the result to be submitted for vaccine design.

1. Confirm that only the accepted candidates are in the `annotated_filtered.vcf-candidates-33mer.fa` file.
2. Note- if the case includes candidates from pVACsplice or pVACfuse, those will need to be manually added to the `annotated_filtered.vcf-candidates-33mer.fa` file. If you rename the file after adding, make sure that the commands below reflect the updated file name.
3. Edit any sequences in the `annotated_filtered.vcf-candidates-33mer.fa` where proximal variants are not accounted for.
4. Run pVACvector to produce the vector result

### Running pVACvector

This step may require a few iterations to get a result. 

```
cd $WORKING_BASE/../

mkdir pvacvector/v5.3.0

cd pvacvector/v5.3.0
```

We will start with the least stringent parameters.

```
mkdir v1
cd v1
```

Create a bash script called `pvacvector.sh` containing the below code
   
```
export PATIENT_ID="{ patient id }"
export HLA_ALLELES="{ hla alleles }"
export PVAC_VERSION="5.3.0"
export RUN_VERSION="1"
export BASE_DIR="{ that patient id's base folder }"
export OUTDIR="$BASE_DIR/pvacvector/v${PVAC_VERSION}/v${RUN_VERSION}/result"
export INPUT_FASTA="$BASE_DIR/generate_protein_fasta/33-mer/candidates/annotated_filtered.vcf-pass-33mer.fa"

mkdir -p $OUTDIR

#default parameters
pvacvector run --top-score-metric median --keep-tmp-files --n-threads 8 --class-i-epitope-length 8,9,10,11 \
--iedb-install-directory /opt/iedb \
$INPUT_FASTA $PATIENT_ID $HLA_ALLELES \
'all_class_i' \
$OUTDIR
```

Create a `run_5.3.0.sh` script to run the `pvacvector.sh` bash script
```
cd "$(dirname "$0")"
bsub -M 64000000 -G compute-oncology -n 8 -R 'select[mem>64000] rusage[mem=64000]' -q general -a 'docker(griffithlab/pvactools:5.3.0)' -oo pvacvector.stdout -eo pvacvector.stderr /bin/bash pvacvector.sh
```

#### Expected Results

We are looking for pVACvector to make a DNA vector where all peptides are combined into one 
molecule minimizing the effects of junctional epitopes (that may create novel peptides) between the sequences. 
pVACvector inserts spacer amino acid sequences to minimize the binding score of junction epitopes.
We are looking for an overall sequence where all the novel junctional epitopes introduced have poor binding.

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
It lists the candidates which it has vectorized. Then the median and lowest junction score, which in this case we require be over 500 (default parameters),
and all the individual junction scores. For further explanation read the pVACvector [documentation](https://pvactools.readthedocs.io/en/latest/pvacvector.html).
**There is a really useful graphic on the [Output Files Page](https://pvactools.readthedocs.io/en/latest/pvacvector/output_files.html).**

The long sequence is our final DNA vector and the contents of this file should be pasted into the genomics report and the classI and classII peptides colored red and bolded. Also add the script/parameters that resulted in the final version of the pVACvector results to the genomics report.

<img width="493" alt="Example of DNA Vector in Genomic Report" src="https://github.com/user-attachments/assets/68cba798-d09f-4ea2-ab49-ad658486fdfd" />


#### Running Different Versions of pVACvector
pVACvector should also be run with stricter parameters as below to obtain the most optimal design. 
Here is an example of a bunch of other versions of commands we have tried (you might have to play with the parameters to get results!)

```
# v2 parameters (more conservative compared to v1)
export PATIENT_ID="{ patient id }"
export HLA_ALLELES="{ hla alleles }"
export PVAC_VERSION="5.3.0"
export RUN_VERSION="2"
export BASE_DIR="{ that patient id's base folder }"
export OUTDIR="$BASE_DIR/pvacvector/v${PVAC_VERSION}/v${RUN_VERSION}/result"
export INPUT_FASTA="$BASE_DIR/generate_protein_fasta/33-mer/candidates/annotated_filtered.vcf-pass-33mer.fa"

mkdir -p $OUTDIR

pvacvector run --top-score-metric median --binding-threshold 1000 \
--keep-tmp-files --n-threads 8 --class-i-epitope-length 8,9,10,11 \
--iedb-install-directory /opt/iedb \
$INPUT_FASTA $PATIENT_ID $HLA_ALLELES \
'all_class_i' \
$OUTDIR

# v3 parameters (more conservative compared to v1 and v2)
export PATIENT_ID="{ patient id }"
export HLA_ALLELES="{ hla alleles }"
export PVAC_VERSION="5.3.0"
export RUN_VERSION="3"
export BASE_DIR="{ that patient id's base folder }"
export OUTDIR="$BASE_DIR/pvacvector/v${PVAC_VERSION}/v${RUN_VERSION}/result"
export INPUT_FASTA="$BASE_DIR/generate_protein_fasta/33-mer/candidates/annotated_filtered.vcf-pass-33mer.fa"

mkdir -p $OUTDIR

pvacvector run --top-score-metric median --binding-threshold 1000 --percentile-threshold 2 --percentile-threshold-strategy conservative \
--keep-tmp-files --n-threads 8 --class-i-epitope-length 8,9,10,11 \
--iedb-install-directory /opt/iedb \
$INPUT_FASTA $PATIENT_ID $HLA_ALLELES \
'all_class_i' \
$OUTDIR

```

