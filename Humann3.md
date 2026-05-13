#!/bin/bash
# ============================================================
# HUMAnN3 PARALLEL PIPELINE
# ============================================================
INPUT_DIR="/data/work/alignment_results/genome"
PROT_DB="/data/work/ref_human"
KRAKEN_DIR="/data/work/kraken_results/report"
OUTDIR="/data/work/funct"
THREADS=8
PARALLEL_JOBS=3

echo "============================================================"
echo " ⚡ HUMAnN3 PARALLEL PIPELINE START"
echo "============================================================"
echo "Parallel jobs: $PARALLEL_JOBS samples simultaneously"
echo "============================================================"

# Environment setup
export HUMANN_CONFIG_USE_NUCLEOTIDE_SEARCH=no
export HUMANN_NUCLEOTIDE_DATABASE=""
export HUMANN_CHOCOPHLAN_DATABASE=""
export OMP_NUM_THREADS=1

# Function untuk process satu sample
process_sample() {
    SAMPLE="$1"
    F1="${INPUT_DIR}/${SAMPLE}_genome_Unmapped.out.mate1"
    F2="${INPUT_DIR}/${SAMPLE}_genome_Unmapped.out.mate2"
    KR="${KRAKEN_DIR}/${SAMPLE}_report.txt"

    echo "🔄 STARTING: $SAMPLE"

    # Kraken profile
    TAXA_PROFILE=""
    if [[ -f "$KR" ]]; then
        TAXA_PROFILE="${OUTDIR}/${SAMPLE}_kraken_for_humann.tsv"
        awk 'NR>1 && NF>=6 {print $6, $2, $1; exit}' "$KR" > "$TAXA_PROFILE"
    fi

    # Run HUMAnN3
    humann \
        --input "$F1" \
        --input "$F2" \
        --output "$OUTDIR" \
        --output-basename "$SAMPLE" \
        --protein-database "$PROT_DB" \
        --threads "$THREADS" \
        --bypass-nucleotide-index \
        --bypass-nucleotide-search \
        ${TAXA_PROFILE:+--taxonomic-profile "$TAXA_PROFILE"} \
        --log-level ERROR

    if [[ $? -eq 0 ]]; then
        echo "✅ COMPLETED: $SAMPLE"
        return 0
    else
        echo "❌ FAILED: $SAMPLE"
        return 1
    fi
}

export -f process_sample
export INPUT_DIR PROT_DB KRAKEN_DIR OUTDIR THREADS

# ============================================================
# 🔍 AUTO-DETECT SAMPLES FROM INPUT_DIR
# ============================================================
mapfile -t SAMPLES < <(
    ls "${INPUT_DIR}"/*_genome_Unmapped.out.mate1 2>/dev/null \
    | xargs -n1 basename \
    | sed 's/_genome_Unmapped\.out\.mate1$//'
)

if [[ ${#SAMPLES[@]} -eq 0 ]]; then
    echo "❌ No samples found in $INPUT_DIR"
    echo "   Expected pattern: <sample>_genome_Unmapped.out.mate1"
    exit 1
fi

echo "📋 Detected ${#SAMPLES[@]} samples:"
printf "   - %s\n" "${SAMPLES[@]}"
echo ""

# ============================================================
# 🚀 PARALLEL EXECUTION
# ============================================================
echo "🎯 RUNNING ${#SAMPLES[@]} SAMPLES IN PARALLEL..."

if command -v parallel &> /dev/null; then
    printf "%s\n" "${SAMPLES[@]}" | parallel -j $PARALLEL_JOBS --eta 'process_sample {}'
else
    echo "⚠️ GNU Parallel not found, using background processes..."
    for SAMPLE in "${SAMPLES[@]}"; do
        process_sample "$SAMPLE" &
    done
    wait
fi

echo "============================================================"
echo "🎉 PARALLEL PIPELINE COMPLETED"
echo "============================================================"
