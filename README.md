# Start the bear project

Helped by IA Claude 

## Collect the information

### Reference Genome

On NCBI I dwnl the genome an the annotation

```bash
wget https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/023/065/955/GCF_023065955.2_UrsArc2.0/GCF_023065955.2_UrsArc2.0_genomic.fna.gz
wget https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/023/065/955/GCF_023065955.2_UrsArc2.0/GCF_023065955.2_UrsArc2.0_genomic.gff.gz
gunzip GCF_023065955.2_UrsArc2.0_genomic.fna.gz
gunzip GCF_023065955.2_UrsArc2.0_genomic.gff.gz
mv GCF_023065955.2_UrsArc2.0_genomic.fna.gz REF_GENOME.fasta
mv GCF_023065955.2_UrsArc2.0_genomic.gff.gz Annotation_RDF_genome.gff
```

### SRA collection 

On NCBI I got all the SRA comming form bioproject conserning Ursus arctor and I fond SRA from  6 differents localisation: 
- Central apennine mountains
- Eurasia
- Irish
- Kunashir
- Quebec
- scandinave

The number of SRA is not the same for each regions and some like Quebec, Kunashir or central montains have a really low number of accessions 


## Stating the pipeline 

as previous pipeline it didn't change the most exept that I add option of the genome's annotation 

``` bash
#!/bin/bash
set -euo pipefail

# API NCBI ---
NCBI_API_KEY="855294aa6a5d0abb5c5286e93121a14908"

# --- Génome de référence (chemin absolu, extension .fasta obligatoire) ---
REFERENCE="/home/alexis/data/Ours/genome/REF_GENOME.fasta"

# --- Fichier d'annotation (GFF/GTF) : laisser vide "" si non utilisé ---
ANNOTATION="/home/alexis/data/Ours/genome/Annotation_RDF_genome.gff"

# --- Ploïdie (1 = haploïde, 2 = diploïde) ---
PLOIDY=2

# --- Reads locaux (laisser vide "" si non utilisé) ---
READS=""

# --- Accessions SRA (laisser vide "" si non utilisé) ---
SRA_INDEX="/home/alexis/data/Ours/SRA/SRA_all.txt"

# --- BAM pré-existants (laisser vide "" si non utilisé) ---
BAM_INPUT=""

# --- Sample map CSV (laisser vide "" si non utilisé) ---
SRR_SAMPLE_MAP=""

# --- Profil d'exécution : local | local_highCPU | slurm ---
PROFILE="slurm"

# --- Répertoire temporaire ---
WORK_DIR="/scratch/my_nf_tmp"

# --- Répertoire de sortie (chemin absolu) ---
OUT_DIR="/home/alexis/data/Ours/results"

# --- Options avancées ---
REFERENCE_SEGMENTS=0
MIN_CONTIG_LENGTH=""
BWA_INDEX=""
CALL_INVAR_SITES="false"
KEEP_BAM="false"
KEEP_GVCF="false"
RESUME="true"

# --- Méthode d'installation de Nextflow : module | conda | skip ---
NEXTFLOW_INSTALL="conda"

# --- Méthode d'obtention des images Singularity : legserv | pull | skip ---
SINGULARITY_METHOD="pull"

# --- Téléchargement du génome de référence : ensembl | legserv | skip ---
REFERENCE_DOWNLOAD="skip"

#=============================================================================
# ██  FIN DE LA CONFIGURATION — NE MODIFIEZ RIEN EN DESSOUS  ██
#=============================================================================

echo "============================================================"
echo "  genomepanel_nf — Installation & Exécution automatisées"
echo "============================================================"
echo ""

# -------------------------------------------------------------------
# ÉTAPE 1 : Utilisation du dépôt local (PAS de clonage)
# -------------------------------------------------------------------
echo ">>> ÉTAPE 1 : Utilisation du dépôt local genomepanel_nf..."

if [ -d "genomepanel_nf" ]; then
    echo "    Dossier genomepanel_nf trouvé dans $(pwd)."
    cd genomepanel_nf
else
    echo "    ❌ Dossier genomepanel_nf introuvable dans $(pwd)"
    echo "    Placez une copie du dépôt genomepanel_nf ici puis relancez."
    exit 1
fi
echo ""

# -------------------------------------------------------------------
# ÉTAPE 2 : Installation de Nextflow
# -------------------------------------------------------------------
echo ">>> ÉTAPE 2 : Configuration de Nextflow (méthode: $NEXTFLOW_INSTALL)..."

case "$NEXTFLOW_INSTALL" in
    module)
        module load Nextflow
        echo "    ✅ Module Nextflow chargé."
        ;;
    conda)
        if ! command -v micromamba &> /dev/null; then
            echo "    ⚠️  micromamba non trouvé. Tentative avec conda..."
            if ! command -v conda &> /dev/null; then
                echo "    ❌ Ni micromamba ni conda ne sont installés. Installez l'un des deux."
                exit 1
            fi
            MAMBA_CMD="conda"
            ACTIVATE_CMD="source activate"
        else
            MAMBA_CMD="micromamba"
            ACTIVATE_CMD="micromamba activate"
            eval "$(micromamba shell hook --shell bash)"
        fi

        # Désactiver -u pendant l'activation pour éviter l'erreur JAVA_LD_LIBRARY_PATH
        set +u
        if $MAMBA_CMD env list 2>/dev/null | grep -q "nf_gp_env"; then
            echo "    L'environnement nf_gp_env existe déjà. Activation..."
            $ACTIVATE_CMD nf_gp_env
        else
            $MAMBA_CMD create -n nf_gp_env -y
            $ACTIVATE_CMD nf_gp_env
            $MAMBA_CMD install -c bioconda nextflow -y
        fi
        set -u

        echo "    ✅ Environnement conda activé avec Nextflow."
        ;;
    skip)
        echo "    Installation Nextflow ignorée (skip)."
        ;;
    *)
        echo "    ❌ Méthode inconnue: $NEXTFLOW_INSTALL"
        exit 1
        ;;
esac
echo ""

# -------------------------------------------------------------------
# ÉTAPE 3 : Images Singularity
# -------------------------------------------------------------------
echo ">>> ÉTAPE 3 : Récupération des images Singularity (méthode: $SINGULARITY_METHOD)..."

case "$SINGULARITY_METHOD" in
    legserv)
        rsync -va /legserv/Temp/Shared/genomepanel_nf/singularity .
        echo "    ✅ Images copiées depuis LEGserv."
        ;;
    pull)
        mkdir -p singularity
        cd singularity

        IMAGES=(
            "https://depot.galaxyproject.org/singularity/entrez-direct:24.0--he881be0_0"
            "https://depot.galaxyproject.org/singularity/sra-tools%3A3.2.1--h4304569_1"
            "https://depot.galaxyproject.org/singularity/fastp%3A0.24.1--heae3180_0"
            "https://depot.galaxyproject.org/singularity/bwa-mem2%3A2.2.1--he70b90d_8"
            "https://depot.galaxyproject.org/singularity/samtools%3A1.22.1--h96c455f_0"
            "https://depot.galaxyproject.org/singularity/picard%3A3.4.0--hdfd78af_0"
            "https://depot.galaxyproject.org/singularity/gatk4-spark%3A4.6.2.0--hdfd78af_0"
            "https://depot.galaxyproject.org/singularity/bcftools%3A1.21--h8b25389_0"
            "https://depot.galaxyproject.org/singularity/r-tidyverse%3A1.2.1"
            "https://depot.galaxyproject.org/singularity/vcftools%3A0.1.17--pl5321h077b44d_0"
        )

        for img in "${IMAGES[@]}"; do
            FILENAME_ENCODED=$(basename "$img")
            FILENAME_DECODED=$(echo "$FILENAME_ENCODED" | sed 's/%3A/:/g')
            if [ -f "${FILENAME_DECODED}.sif" ] || [ -f "$FILENAME_DECODED" ] || \
               [ -f "${FILENAME_ENCODED}.sif" ] || [ -f "$FILENAME_ENCODED" ]; then
                echo "    ⏭️  Image déjà présente : $FILENAME_DECODED"
            else
                echo "    ⬇️  Téléchargement : $FILENAME_DECODED ..."
                singularity pull --name "${FILENAME_DECODED}.sif" "$img"
            fi
        done

        cd ..
        echo "    ✅ Toutes les images Singularity sont prêtes."
        ;;
    skip)
        echo "    Récupération Singularity ignorée (skip)."
        ;;
    *)
        echo "    ❌ Méthode inconnue: $SINGULARITY_METHOD"
        exit 1
        ;;
esac
echo ""

# -------------------------------------------------------------------
# ÉTAPE 4 : Téléchargement du génome de référence (optionnel)
# -------------------------------------------------------------------
echo ">>> ÉTAPE 4 : Génome de référence (méthode: $REFERENCE_DOWNLOAD)..."

case "$REFERENCE_DOWNLOAD" in
    ensembl)
        wget -nc http://ftp.ensemblgenomes.org/pub/fungi/release-61/fasta/zymoseptoria_tritici/dna/Zymoseptoria_tritici.MG2.dna.toplevel.fa.gz
        gunzip -f Zymoseptoria_tritici.MG2.dna.toplevel.fa.gz
        mv Zymoseptoria_tritici.MG2.dna.toplevel.fa IPO323.fasta
        REFERENCE="$PWD/IPO323.fasta"
        echo "    ✅ Génome téléchargé depuis Ensembl Fungi."
        ;;
    legserv)
        cp /legserv/NGS_data/Zymoseptoria/Zt_Reference_genomes/19Pangenome_genomes/IPO323/Zymoseptoria_tritici.MG2.dna.toplevel.mt+.fa .
        mv Zymoseptoria_tritici.MG2.dna.toplevel.mt+.fa IPO323.fasta
        REFERENCE="$PWD/IPO323.fasta"
        echo "    ✅ Génome copié depuis LEGserv."
        ;;
    skip)
        echo "    Téléchargement ignoré. Utilisation de : $REFERENCE"
        ;;
    *)
        echo "    ❌ Méthode inconnue: $REFERENCE_DOWNLOAD"
        exit 1
        ;;
esac
echo ""

# -------------------------------------------------------------------
# ÉTAPE 5 : Construction de la commande Nextflow
# -------------------------------------------------------------------
echo ">>> ÉTAPE 5 : Construction et lancement du pipeline..."

export NXF_OPTS='-Xms8g -Xmx64g'

# Réactiver l'environnement pour garantir que nextflow est dans le PATH
if [ "$NEXTFLOW_INSTALL" = "conda" ]; then
    echo "    Activation de l'environnement conda nf_gp_env..."

    # Désactiver -u pour éviter le problème JAVA_LD_LIBRARY_PATH lors de l'activation
    set +u
    if command -v micromamba &> /dev/null; then
        eval "$(micromamba shell hook --shell bash)"
        micromamba activate nf_gp_env
        echo "    ✅ Environnement micromamba activé."
    elif command -v conda &> /dev/null; then
        source "$(conda info --base)/etc/profile.d/conda.sh"
        conda activate nf_gp_env
        echo "    ✅ Environnement conda activé."
    else
        echo "    ❌ Impossible d'activer l'environnement conda."
        set -u
        exit 1
    fi
    set -u
fi

# Vérifier que nextflow est disponible
if ! command -v nextflow &> /dev/null; then
    echo "    ❌ ERREUR : nextflow n'est pas disponible dans le PATH."
    echo "    Vérifiez que l'environnement conda est correctement activé."
    echo "    Essayez manuellement : micromamba activate nf_gp_env"
    exit 1
fi

NEXTFLOW_BIN=$(which nextflow)
echo "    Nextflow trouvé : $NEXTFLOW_BIN"
echo ""

CMD="$NEXTFLOW_BIN run main.nf -config nextflow.config"
CMD+=" -profile ${PROFILE}"
CMD+=" -work-dir '${WORK_DIR}'"
CMD+=" --outdir '${OUT_DIR}'"
CMD+=" --reference '${REFERENCE}'"
CMD+=" --ploidy ${PLOIDY}"

# Gestion des entrées : BAM, READS ou SRA
if [ -n "$BAM_INPUT" ]; then
    CMD+=" --bam_input '${BAM_INPUT}'"
else
    if [ -n "$READS" ]; then
        CMD+=" --reads '${READS}'"
    fi
    if [ -n "$SRA_INDEX" ]; then
        CMD+=" --SRA_index '${SRA_INDEX}'"
        CMD+=" --NCBI_API_key '${NCBI_API_KEY}'"
    fi
    if [ -n "$SRR_SAMPLE_MAP" ]; then
        CMD+=" --SRR_sample_map '${SRR_SAMPLE_MAP}'"
    fi
fi

# Options avancées
if [ "$REFERENCE_SEGMENTS" -ne 0 ] 2>/dev/null; then
    CMD+=" --reference_segments ${REFERENCE_SEGMENTS}"
fi

if [ -n "$MIN_CONTIG_LENGTH" ]; then
    CMD+=" --min_contig_length ${MIN_CONTIG_LENGTH}"
fi

if [ -n "$BWA_INDEX" ]; then
    CMD+=" --bwa_index '${BWA_INDEX}'"
fi

if [ "$CALL_INVAR_SITES" = "true" ]; then
    CMD+=" --call_invar_sites true"
fi

if [ "$KEEP_BAM" = "true" ]; then
    CMD+=" --keep_bam true"
fi

if [ "$KEEP_GVCF" = "true" ]; then
    CMD+=" --keep_gvcf true"
fi

if [ "$RESUME" = "true" ]; then
    CMD+=" -resume"
fi

echo ""
echo "============================================================"
echo "  COMMANDE NEXTFLOW GÉNÉRÉE :"
echo "============================================================"
echo ""
echo "$CMD" | sed 's/ -/\n  -/g; s/ --/\n  --/g'
echo ""
echo "============================================================"
echo ""

read -p "🚀 Lancer le pipeline maintenant ? (o/n) : " CONFIRM
if [[ "$CONFIRM" =~ ^[oOyY]$ ]]; then
    echo ""
    echo ">>> Lancement du pipeline..."
    echo ">>> ⚠️  Assurez-vous d'être dans une session tmux ou screen !"
    echo ""
    eval $CMD
    echo ""
    echo "============================================================"
    echo "  ✅ Pipeline terminé avec succès !"
    echo "  📂 Résultats dans : ${OUT_DIR}"
    echo "============================================================"
else
    echo ""
    echo "Pipeline non lancé. Vous pouvez copier la commande ci-dessus"
    echo "et l'exécuter manuellement dans une session tmux/screen."
fi
```

But in order to add annotation i had to change a bit the mail and workflowconfig files  !!! Take care to do a backup before !!!

main:

```bash
#!/usr/bin/env nextflow

nextflow.enable.dsl=2

// ===================== PARAMÈTRES =====================

params.annotation        = null
params.outdir            = "./nf_output/"
params.reference         = ""
params.reads             = ""
params.SRA_index         = ""
params.ploidy            = ""
params.NCBI_API_key      = ""
params.keep_bam          = false   // Keep per-sample BAM files
params.keep_gvcf         = false   // Keep per-sample GVCF files
params.bwa_index         = ""      // Optional: path to pre-built BWA index files
params.min_contig_length = false   // Filter reference contigs shorter than this value (bp)
params.reference_segments = 0      // Genome segment size for parallel variant calling (bp)
params.call_invar_sites  = false   // Call invariant sites with GATK HaplotypeCaller
params.bam_input         = ""      // Optional: path to pre-existing BAM files
params.SRR_sample_map    = ""      // Optional: TSV file mapping SRR IDs to sample names

// ===================== IMPORT DU WORKFLOW =====================

include { variant_calling } from './workflows/variant_calling_wf'

// ===================== LOG DE CONFIGURATION =====================

log.info """

   ===========================================
   || GENOME PANEL VARIANT CALLING WORKFLOW ||
   ===========================================

    Steps:
     1. SRA query and download
     2. Fastp filtering
     3. BWA alignment
     4. GATK HaplotypeCaller
     5. GATK VariantFiltration
     6. Variant quality plotting, read and mapping stats   

   Configuration:
       NCBI API key    : ${params.NCBI_API_key}
       Ploidy          : ${params.ploidy}
       Inv. site calls : ${params.call_invar_sites}

       Reference       : ${params.reference}
       Annotation      : ${params.annotation}
       Ref segments    : ${params.reference_segments}
       Min contig len  : ${params.min_contig_length}
       BWA index       : ${params.bwa_index}

     Input data
       Local fastq     : ${params.reads}
       SRA ID file     : ${params.SRA_index}
       SRR-sample map  : ${params.SRR_sample_map}
       Local bam       : ${params.bam_input}

     Output options
       Working dir     : ${params.workdir}
       Output dir      : ${params.outdir}
       Keep BAM        : ${params.keep_bam}
       Keep GVCF       : ${params.keep_gvcf}
    """


// ===================== WORKFLOW PRINCIPAL =====================

workflow {

    if( params.annotation ) {
        log.info "Annotation file provided: ${params.annotation}"
    } else {
        log.info "No annotation file provided (params.annotation is null or empty)"
    }

    // Appel du workflow interne SANS argument
    variant_calling()
}
```

Workflow config: 

```bash
// ============================================================
// Configuration Singularity globale
// ============================================================
singularity {
    enabled    = true
    autoMounts = true
    cacheDir   = "${projectDir}/singularity"
    runOptions = '--cleanenv'
}

docker.enabled = false
podman.enabled = false

// ============================================================
// Paramètres du pipeline
// ============================================================
params {
    reference          = null
    annotation         = null
    ploidy             = 2

    reads              = null
    SRA_index          = null
    SRR_sample_map     = null
    bam_input          = null

    reference_segments = 0
    min_contig_length  = null
    bwa_index          = null

    call_invar_sites   = false
    keep_bam           = false
    keep_gvcf          = false

    outdir             = "results"
    NCBI_API_key       = null
}

// ============================================================
// Assignation des conteneurs par processus
// ============================================================
process {
    // Conteneur par défaut
    container = "${projectDir}/singularity/samtools_1.22.1--h96c455f_0.sif"

    withName: 'variant_calling:SRAresolve' {
        container = "${projectDir}/singularity/entrez-direct_24.0--he881be0_0.sif"
    }

    withName: 'variant_calling:SRAdownloadPE|variant_calling:SRAdownloadSE' {
        container = "${projectDir}/singularity/sra-tools_3.2.1--h4304569_1.sif"
    }

    withName: 'variant_calling:trimSequencesPE|variant_calling:trimSequencesSE' {
        container = "${projectDir}/singularity/fastp_0.24.1--heae3180_0.sif"
    }

    withName: 'variant_calling:bwaIndex|variant_calling:bwaMap' {
        container = "${projectDir}/singularity/bwa-mem2_2.2.1--he70b90d_8.sif"
    }

    withName: 'variant_calling:fastaIndex|variant_calling:samtoolsSort' {
        container = "${projectDir}/singularity/samtools_1.22.1--h96c455f_0.sif"
    }

    withName: 'variant_calling:addRG|variant_calling:dupRemoval' {
        container = "${projectDir}/singularity/picard_3.4.0--hdfd78af_0.sif"
    }

    withName: 'variant_calling:gatkIndex|variant_calling:GATKHC|variant_calling:CombineGVCFs|variant_calling:GenotypeGVCFs|variant_calling:FilterVCFs' {
        container = "${projectDir}/singularity/gatk4-spark_4.6.2.0--hdfd78af_0.sif"
    }

    withName: 'variant_calling:CleanVCFs|variant_calling:ConcatCleanVCFs' {
        container = "${projectDir}/singularity/bcftools_1.21--h8b25389_0.sif"
    }

    withName: 'variant_calling:vcftools.*' {
        container = "${projectDir}/singularity/vcftools_0.1.17--pl5321h077b44d_0.sif"
    }
}

// ============================================================
// Profils d'exécution
// ============================================================
profiles {

    local {
        process.executor = 'local'
        process.cpus     = 4
        process.memory   = '8 GB'
        process.time     = '24h'
    }

    local_highCPU {
        process.executor = 'local'
        process.cpus     = 16
        process.memory   = '64 GB'
        process.time     = '72h'
    }

    slurm {
        process {
            executor = 'slurm'
            queue    = 'normal.168h'
            
            // Ressources par défaut
            cpus     = 4
            memory   = '8 GB'
            time     = '24h'
            
            // Options SLURM supplémentaires si nécessaire
            // clusterOptions = '--account=TON_COMPTE --qos=drng'
        }

        // Ressources spécifiques pour certains processus gourmands
        process {
            withName: 'variant_calling:bwaMap' {
                cpus   = 8
                memory = '32 GB'
                time   = '48h'
            }

            withName: 'variant_calling:GATKHC' {
                cpus   = 4
                memory = '16 GB'
                time   = '72h'
            }

            withName: 'variant_calling:GenotypeGVCFs' {
                cpus   = 4
                memory = '32 GB'
                time   = '48h'
            }
        }
    }
}
``` 
## VCF
