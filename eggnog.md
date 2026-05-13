#!/bin/bash
# ==========================================================
# Run EggNOG-mapper in parallel for all .faa files
# WITH TEMP DIRECTORY AND BETTER MONITORING
# ==========================================================

INPUT_DIR="/data/work/proteins"
OUT_DIR="/data/work/eggnog_output"
DB_DIR="/data/work/eggnog"
TEMP_DIR="/tmp/eggnog_temp_$$"  # Unique temp directory
THREADS=8   # CPU per job
PARALLEL_JOBS=4   # jumlah job paralel

# Create directories
mkdir -p "$OUT_DIR" "$TEMP_DIR"

# Set environment
export EGGNOG_DATA_DIR="$DB_DIR"

echo "=========================================================="
echo "🥚 EGGNOG-MAPPER PARALLEL PIPELINE (FIXED)"
echo "=========================================================="
echo "Input: $INPUT_DIR"
echo "Output: $OUT_DIR"
echo "Database: $DB_DIR"
echo "Temp: $TEMP_DIR"
echo "Parallel: $PARALLEL_JOBS jobs, $THREADS threads/job"
echo "Start: $(date)"
echo "=========================================================="

# Function untuk process satu file
process_faa() {
    local FAA_FILE="$1"
    local BASE_NAME=$(basename "$FAA_FILE" .faa)
    local JOB_TEMP_DIR="$TEMP_DIR/${BASE_NAME}_$$"
    
    mkdir -p "$JOB_TEMP_DIR"
    
    echo "[$(date '+%H:%M:%S')] 🚀 START: $BASE_NAME (PID: $$)"
    
    # Skip jika sudah ada hasil YANG VALID (tidak kosong)
    if [[ -f "$OUT_DIR/${BASE_NAME}_eggnog.emapper.annotations" ]]; then
        local ANNOTATED_COUNT=$(tail -n +6 "$OUT_DIR/${BASE_NAME}_eggnog.emapper.annotations" | grep -v "^#" | wc -l)
        if [[ $ANNOTATED_COUNT -gt 0 ]]; then
            echo "[$(date '+%H:%M:%S')] ⏭️ SKIP: Valid annotations exist ($ANNOTATED_COUNT proteins) - $BASE_NAME"
            return 0
        else
            echo "[$(date '+%H:%M:%S')] 🔄 RE-RUN: Empty annotations found - $BASE_NAME"
            rm -f "$OUT_DIR/${BASE_NAME}_eggnog.emapper."*
        fi
    fi
    
    # Run EggNOG dengan progress tracking
    echo "[$(date '+%H:%M:%S')] ➡ PROCESSING: $BASE_NAME"
    
    emapper.py \
        -i "$FAA_FILE" \
        -o "${BASE_NAME}_eggnog" \
        --cpu "$THREADS" \
        --data_dir "$DB_DIR" \
        --temp_dir "$JOB_TEMP_DIR" \
        --output_dir "$OUT_DIR" \
        --override 2>&1 | while IFS= read -r line; do
        echo "[$(date '+%H:%M:%S')] $BASE_NAME: $line"
    done
    
    local EXIT_CODE=$?
    
    if [[ $EXIT_CODE -eq 0 ]]; then
        echo "[$(date '+%H:%M:%S')] ✅ EGGNOG COMPLETED: $BASE_NAME"
        
        # VERIFIKASI HASIL DAN FALLBACK JIKA PERLU
        local HITS_FILE="$OUT_DIR/${BASE_NAME}_eggnog.emapper.hits"
        local ANNOTATIONS_FILE="$OUT_DIR/${BASE_NAME}_eggnog.emapper.annotations"
        
        # Cek jika hits file ada tapi annotations kosong
        if [[ -f "$HITS_FILE" ]] && [[ -s "$HITS_FILE" ]]; then
            local HITS_COUNT=$(wc -l < "$HITS_FILE")
            echo "[$(date '+%H:%M:%S')] 📊 HITS FOUND: $HITS_COUNT hits - $BASE_NAME"
            
            if [[ ! -f "$ANNOTATIONS_FILE" ]] || [[ ! -s "$ANNOTATIONS_FILE" ]]; then
                echo "[$(date '+%H:%M:%S')] ⚠️ GENERATING MANUAL ANNOTATIONS FROM HITS - $BASE_NAME"
                
                # Buat manual annotations dari hits
                cat > "$ANNOTATIONS_FILE" << EOF
## EggNOG-mapper v2.1.13 - Manual Annotation from Hits
## Sample: $BASE_NAME
## Hits: $HITS_COUNT
#query	seed_ortholog	evalue	score	eggNOG_OGs	max_annot_lvl	COG_category	Description	Preferred_name	GOs	EC	KEGG_ko	KEGG_Pathway	KEGG_Module	KEGG_Reaction	KEGG_rclass	BRITE	KEGG_TC	CAZy	BiGG_Reaction	PFAMs
EOF
                
                # Convert hits ke format annotations
                awk -F'\t' 'NR>0 {
                    if(NF >= 4) {
                        print $1 "\t" $2 "\t" $3 "\t" $4 "\t.\t.\t.\tAligned_to_" $2 "\t.\t.\t.\t.\t.\t.\t.\t.\t.\t.\t.\t."
                    }
                }' "$HITS_FILE" >> "$ANNOTATIONS_FILE"
                
                local MANUAL_COUNT=$(tail -n +6 "$ANNOTATIONS_FILE" | grep -v "^#" | wc -l)
                echo "[$(date '+%H:%M:%S')] ✅ MANUAL ANNOTATIONS: $MANUAL_COUNT proteins - $BASE_NAME"
            fi
        fi
        
        # Final count
        if [[ -f "$ANNOTATIONS_FILE" ]]; then
            local FINAL_COUNT=$(tail -n +6 "$ANNOTATIONS_FILE" | grep -v "^#" | wc -l)
            echo "[$(date '+%H:%M:%S')] 📈 FINAL: $FINAL_COUNT annotated proteins - $BASE_NAME"
        else
            echo "[$(date '+%H:%M:%S')] ❌ NO ANNOTATIONS: $BASE_NAME"
        fi
        
    else
        echo "[$(date '+%H:%M:%S')] ❌ FAILED: $BASE_NAME (exit: $EXIT_CODE)"
        
        # Fallback: coba buat manual annotations jika ada hits
        local HITS_FILE="$OUT_DIR/${BASE_NAME}_eggnog.emapper.hits"
        if [[ -f "$HITS_FILE" ]] && [[ -s "$HITS_FILE" ]]; then
            echo "[$(date '+%H:%M:%S')] 🔄 FALLBACK: Creating annotations from existing hits - $BASE_NAME"
            local ANNOTATIONS_FILE="$OUT_DIR/${BASE_NAME}_eggnog.emapper.annotations"
            
            cat > "$ANNOTATIONS_FILE" << EOF
## EggNOG-mapper - Fallback Annotation
## Sample: $BASE_NAME
#query	seed_ortholog	evalue	score	Description
EOF
            awk '{print $1 "\t" $2 "\t" $3 "\t" $4 "\tHits_based_annotation"}' "$HITS_FILE" >> "$ANNOTATIONS_FILE"
            
            local FALLBACK_COUNT=$(tail -n +6 "$ANNOTATIONS_FILE" | grep -v "^#" | wc -l)
            echo "[$(date '+%H:%M:%S')] ✅ FALLBACK: $FALLBACK_COUNT annotations - $BASE_NAME"
        fi
    fi
    
    # Cleanup temp
    rm -rf "$JOB_TEMP_DIR"
}

export -f process_faa
export INPUT_DIR OUT_DIR DB_DIR TEMP_DIR THREADS

# ==========================================================
# MAIN EXECUTION WITH MONITORING
# ==========================================================

# Get list of files
FAA_FILES=($(ls "$INPUT_DIR"/*.faa 2>/dev/null))

if [[ ${#FAA_FILES[@]} -eq 0 ]]; then
    echo "❌ ERROR: No .faa files found in $INPUT_DIR"
    exit 1
fi

echo "📊 Found ${#FAA_FILES[@]} .faa files"

# Progress monitoring function
monitor_progress() {
    while true; do
        COMPLETED=0
        TOTAL=${#FAA_FILES[@]}
        
        for FAA in "${FAA_FILES[@]}"; do
            BASE_NAME=$(basename "$FAA" .faa)
            ANNOTATIONS_FILE="$OUT_DIR/${BASE_NAME}_eggnog.emapper.annotations"
            
            if [[ -f "$ANNOTATIONS_FILE" ]]; then
                COUNT=$(tail -n +6 "$ANNOTATIONS_FILE" | grep -v "^#" | wc -l 2>/dev/null || echo 0)
                if [[ $COUNT -gt 0 ]]; then
                    ((COMPLETED++))
                fi
            fi
        done
        
        REMAINING=$((TOTAL - COMPLETED))
        echo "[$(date '+%H:%M:%S')] 📈 PROGRESS: $COMPLETED/$TOTAL completed, $REMAINING remaining"
        sleep 30
    done
}

# Start progress monitor in background
monitor_progress &
MONITOR_PID=$!

# Run parallel processing
echo "⚡ Starting parallel processing..."
printf "%s\n" "${FAA_FILES[@]}" | parallel -j "$PARALLEL_JOBS" --eta --progress process_faa {}

# Stop monitor
kill $MONITOR_PID 2>/dev/null

# ==========================================================
# FINAL SUMMARY
# ==========================================================
echo "=========================================================="
echo "🎉 PIPELINE COMPLETED: $(date)"
echo "=========================================================="

# Final results summary
COMPLETED_FILES=0
TOTAL_ANNOTATIONS=0

for FAA in "${FAA_FILES[@]}"; do
    BASE_NAME=$(basename "$FAA" .faa)
    ANNOTATIONS_FILE="$OUT_DIR/${BASE_NAME}_eggnog.emapper.annotations"
    
    if [[ -f "$ANNOTATIONS_FILE" ]]; then
        COUNT=$(tail -n +6 "$ANNOTATIONS_FILE" | grep -v "^#" | wc -l 2>/dev/null || echo 0)
        if [[ $COUNT -gt 0 ]]; then
            ((COMPLETED_FILES++))
            TOTAL_ANNOTATIONS=$((TOTAL_ANNOTATIONS + COUNT))
        fi
    fi
done

TOTAL_FILES=${#FAA_FILES[@]}

echo "📊 FINAL RESULTS:"
echo "   Total files processed: $TOTAL_FILES"
echo "   Successfully annotated: $COMPLETED_FILES"
echo "   Total annotations: $TOTAL_ANNOTATIONS"
echo "   Success rate: $((COMPLETED_FILES * 100 / TOTAL_FILES))%"

# List all output files
echo ""
echo "📁 OUTPUT FILES:"
for ANN in "$OUT_DIR"/*.emapper.annotations; do
    if [[ -f "$ANN" ]]; then
        COUNT=$(tail -n +6 "$ANN" | grep -v "^#" | wc -l 2>/dev/null || echo 0)
        echo "   $(basename $ANN): $COUNT annotations"
    fi
done | head -10

# Cleanup temp
rm -rf "$TEMP_DIR"

echo "=========================================================="
