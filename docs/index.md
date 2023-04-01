# <KBD>epi2me-labs/wf-human-variation</KBD> for NeSI Mahuika cluster

modified version of <KBD>epi2me-labs/wf-human-variation</KBD>        (https://github.com/epi2me-labs/wf-human-variation) to make the workflow compatible with NeSI Mahuika cluster <KBD>Slurm</KDB> scheduler

??? terminal "03-ont-bam-merge.slurm"
    ```
    !#/bin/bash -e

    #SBATCH --account         nesi12345
    #SBATCH --job-name        ont-bam-merge
    #SBATCH --cpus-per-task   16
    #SBATCH --mem             64G
    #SBATCH --time            24:00:00

    module purge
    module load BamTools/2.5.2-GCC-11.3.0
    module load Sambamba/0.8.0

    # define variables
    WKDIR='/path/to/wkir'                           #For an example : "/nesi/nobackup/nesi12345/.."
    SAMPLE='PBXP289487'

    # move to working dir
    cd ${WKDIR}/${SAMPLE}/bam

    # sambamba
    # sambamba merge -t 4 -p ${SAMPLE}_sorted_merged.bam pass/*.bam
    # bamtools
    ls pass/*.bam > bam_list.txt
    bamtools merge -l bam_list.txt -out ${SAMPLE}_merged.bam
    sambamba sort -m 52GB -t 16 ${SAMPLE}_merged.bam -o ${SAMPLE}_sorted_merged.bam
    sambamba index -t 16 ${SAMPLE}_sorted_merged.bam
    ```

??? terminal "04-ont--human-variation-calling.slurm"

    - Create a file named *nextflow_local_override.config* with the following content 
    
        ```
        executor {
        $local {
                   cpus = 48
                   memory = "128 GB"
               }
                 }

        singularity {
          runOptions = "-B ${TMPDIR}"
                    }
        ```

    - Slurm script 

    ```
    #!/bin/bash -e
    #SBATCH --name          ont-human-variation
    #SBATCH --cpus-per-task 48
    #SBATCH --mem           128G
    #SBATCH --partition     milan
    #SBATCH --time          24:00:00

    module purge
    module load Nextflow/22.10.3
    module load Singularity/3.10.3
     
    
    # define variables
    WKDIR='/path/to/workingdir'                           #For an example : "/nesi/nobackup/nesi12345/.."
    SAMPLE='PBXP289487'
    MODEL='/path/to/clair3_models/ont_guppy5'
    REFERENCE='/path/to/reference/GCA_000001405.15_GRCh38_no_alt_analysis_set.fasta'
    TDREPETS='/path/to/bed/reference/human_GRCh38_no_alt_analysis_set.trf.bed'
    OUTPUT='results'

    #Singularity and Nextflow variables
    export SINGULARITY_TMPDIR=/some/path/in/nobackup/singularity_cache
    export SINGULARITY_CACHEDIR=$SINGULARITY_TMPDIR
    setfacl -b "$SINGULARITY_TMPDIR"

    export NXF_EXECUTOR=slurm
    export NXF_SINGULARITY_CACHEDIR=$SINGULARITY_CACHEDIR

    NFCONFIG='/path/to/nextflow_local_overide.config'
    # note: created an overide config to provide modified CPU and Memory values
    # change these values if you want to tweak performance based on resources
    
    # move to working dir
    cd ${WKDIR}/${SAMPLE}
    
    # pull nextflow pipeline (if haven't already)
    nextflow run epi2me-labs/wf-human-variation --help
    
    # run Clair3 variant calling
    nextflow -c ${NFCONFIG} run epi2me-labs/wf-human-variation \
      -resume \
      --threads ${SLURM_CPUS_PER_TASK} \
      -profile singularity,local \
      --snp --sv \
      --phase_vcf \
      --use_longphase \
      --tr_bed ${TDREPETS} \
      --model ${MODEL} \
      --bam ./bam/${SAMPLE}_sorted_merged.bam \
      --ref ${REFERENCE} \
      --out_dir ${OUTPUT}
    ```

??? terminal "05-ont-whatshap-phase.slurm"
    ```
    #!/bin/bash -e

    #SBATCH --account         nesi12345
    #SBATCH --job-name        ont-whatshap-phase
    #SBATCH --cpus-per-task   16
    #SBATCH --mem             64G
    #SBATCH --time            24:00:00
    
    module purge
    module load WhatsHap/1.6-gimkl-2022a-Python-3.10.5
    module load SAMtools/1.16.1-GCC-11.3.0
    
    # define variables
    WKDIR='/path/to/workingdir'
    SAMPLE='PBXP289487'
    REFERENCE='/path/to/reference/GCA_000001405.15_GRCh38_no_alt_analysis_set.fasta'
    OUTPUT='results'
    
    # move to working dir
    cd ${WKDIR}/${SAMPLE}
    
    # whatshap phase tagging of bam output
    whatshap haplotag \
        --ignore-read-groups \
        --output ./bam/${SAMPLE}_sorted_merged.hp.bam  \
        --reference ${REFERENCE} \
        ${OUTPUT}/all.wf_snp.vcf.gz ./bam/${SAMPLE}_sorted_merged.bam
    # index bam
    samtools index -@ 16 ./bam/${SAMPLE}_sorted_merged.hp.bam
    
    # Notes:
    # this step phases the data based on the clair3 output and generates a 
    # phased bam file. This contains information assigning reads to each
    # haplotype. At this stage the bam is able to loaded into a genome 
    # viewer/browser and contains aligned reads, with base modification
    # (methylation) information, as well as the haplotype information.
    ```
??? terminal "06-ont-methyl-calling.slurm"

    ```
    #!/bin/bash -e
    
    #SBATCH --account         nesi12345
    #SBATCH --job-name        ont-bam-merge
    #SBATCH --cpus-per-task   16
    #SBATCH --mem             32G
    #SBATCH --time            08:00:00
    
    module purge
    module load modbam2bed/0.9.4-GCC-11.3.0
    
    # define variables
    WKDIR='/path/to/workdir'
    SAMPLE='PBXP289487'
    REFERENCE='/path/to/reference/GCA_000001405.15_GRCh38_no_alt_analysis_set.fasta'
    
    # move to working dir
    cd ${WKDIR}/${SAMPLE}
    
    # create methylation bed files
    for HP in 1 2; do
        modbam2bed \
            -e -m 5mC --cpg -t 16 --haplotype ${HP} \
            ${REFERENCE} \
            ./bam/${SAMPLE}_sorted_merged.hp.bam \
            | bgzip -c > ./bed/${SAMPLE}_methylation.hp${HP}.cpg.bed.gz
    done;
    
    # create an aggregated bed file
    modbam2bed \
      -e -m 5mC --cpg --aggregate -t 16 \
      ${REFERENCE} \
      ./bam/${SAMPLE}_sorted_merged.hp.bam \
      | bgzip -c > ./bed/${SAMPLE}_methylation.aggregated.cpg.bed.gz
    
    # Notes:
    # this step extracts the methylation information from the bam file, generating
    # bed files. There are two processes here, the first creates two bed files, one
    # for each haplotype. These are very useful for exploring allele specific methylation.
    # The second process generates a single aggragated bed file, it merges the
    # haplotype data to give site specific methylation. It should be noted that this is
    # per strand, so some processing will be required to 'collapse' the data to a
    # single CpG site, but this type of work is usually performed in the downstream\
    # analysis, using packages such as methylkit.
    ```