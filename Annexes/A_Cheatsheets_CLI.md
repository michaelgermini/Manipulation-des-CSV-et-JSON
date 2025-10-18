# Annexe A : CLI Cheatsheets — Quick Reference Guide

## 🚀 Quick Access Index

- [CSV Tools](#csv-tools)
- [JSON Tools](#json-tools)
- [Unix Commands](#unix-commands)
- [Compression](#compression)
- [Advanced Patterns](#advanced-patterns)

---

## CSV Tools

### csvkit

```bash
# Install
pip install csvkit

# View file
csvlook data.csv

# Statistics
csvstat data.csv

# Extract columns
csvcut -c 1,3,5 data.csv

# Filter rows
csvgrep -c name -m "Alice" data.csv

# Format
csvformat -T data.csv  # Convert to TSV

# Join files
csvjoin -c id file1.csv file2.csv

# Convert to JSON
in2csv -f csv data.csv | csvjson

# Sort
csvsort -c salary -r data.csv
```

### xsv

```bash
# Install
cargo install xsv

# Count records
xsv count data.csv

# Select columns
xsv select name,email data.csv

# Search
xsv search "pattern" data.csv

# Statistics
xsv stats data.csv

# First N rows
xsv slice -l 10 data.csv

# Frequency
xsv frequency -s country data.csv
```

### miller (mlr)

```bash
# Install
brew install miller  # macOS
# or apt-get install miller  # Linux

# View
mlr --csv cat data.csv

# Convert CSV to JSON
mlr --csv --json cat data.csv

# Filter
mlr --csv filter '$salary > 50000' data.csv

# Cut columns
mlr --csv cut -f name,email data.csv

# Aggregate
mlr --csv stats1 -a sum -f salary -g country data.csv

# Join
mlr --csv join -f id -j id file1.csv file2.csv

# Sort
mlr --csv sort -f salary data.csv
```

### jq

```bash
# Install
brew install jq  # macOS

# Pretty print
cat data.json | jq .

# Extract field
cat data.json | jq '.name'

# Array iteration
cat data.json | jq '.[] | .email'

# Filter
cat data.json | jq '.[] | select(.age > 25)'

# Map
cat data.json | jq 'map(.name)'

# Group by
cat data.json | jq 'group_by(.country)'

# Unique
cat data.json | jq 'unique_by(.email)'

# Count
cat data.json | jq 'length'

# Join arrays
jq -s 'add' file1.json file2.json

# Combine with pipes
cat data.json | jq '.[] | select(.country=="FR") | .name'
```

---

## JSON Tools

### jo

```bash
# Create JSON from command line
jo name=Alice email=alice@example.com age=30

# Array
jo -a 1 2 3 4 5

# Nested
jo name=Alice address=$(jo city=Paris zip=75001)
```

### jshon

```bash
# Extract
cat data.json | jshon -e name

# Array length
cat data.json | jshon -l

# Pretty print
cat data.json | jshon -p
```

---

## Unix Commands

### cut

```bash
# By position
cut -c 1-10 file.txt

# By delimiter
cut -d',' -f1,3 data.csv

# Complement
cut -d',' -f 2- data.csv  # All except 1st column

# Output delimiter
cut -d',' -f1,3 data.csv | cut -d' ' --output-delimiter=','
```

### awk

```bash
# Print column
awk -F',' '{print $1}' data.csv

# Filter
awk -F',' '$3 > 50000 {print}' data.csv

# Sum
awk -F',' '{sum += $3} END {print sum}' data.csv

# Count
awk -F',' 'END {print NR}' data.csv

# Multi-pattern
awk -F',' '/pattern/ {print $1} {count++} END {print count}' data.csv

# Variables
awk -F',' -v OFS='|' '{print $1, $2, $3}' data.csv
```

### sed

```bash
# Replace
sed 's/old/new/' file.txt

# Global replace
sed 's/old/new/g' file.txt

# In-place edit
sed -i 's/old/new/g' file.txt

# Delete lines
sed '/pattern/d' file.txt

# Print lines
sed -n '5,10p' file.txt

# Multiple operations
sed -e 's/old/new/g' -e 's/foo/bar/g' file.txt
```

### sort

```bash
# Sort numerically
sort -n numbers.txt

# Sort by column
sort -t',' -k2 data.csv

# Reverse
sort -r data.txt

# Multiple keys
sort -t',' -k2,2 -k3,3n data.csv

# Unique + sort
sort -u data.txt
```

### uniq

```bash
# Remove duplicates
sort data.txt | uniq

# Count occurrences
sort data.txt | uniq -c

# Show only duplicates
sort data.txt | uniq -d

# Show unique
sort data.txt | uniq -u
```

### join

```bash
# Inner join
join -t',' -1 1 -2 1 sorted1.csv sorted2.csv

# Outer join
join -t',' -a1 -a2 sorted1.csv sorted2.csv

# Left join
join -t',' -a1 sorted1.csv sorted2.csv
```

### grep

```bash
# Search
grep "pattern" file.txt

# Case insensitive
grep -i "pattern" file.txt

# Inverse match
grep -v "pattern" file.txt

# Count matches
grep -c "pattern" file.txt

# Files with match
grep -l "pattern" *.txt

# Recursive
grep -r "pattern" directory/

# With context
grep -C 3 "pattern" file.txt
```

---

## Compression

### gzip

```bash
# Compress
gzip data.csv  # Creates data.csv.gz

# Decompress
gzip -d data.csv.gz
gunzip data.csv.gz

# Compress to file
gzip -c data.csv > data.csv.gz

# Stream
cat data.csv | gzip > data.csv.gz

# List contents
gzip -l data.csv.gz
```

### bzip2

```bash
# Compress
bzip2 data.csv  # Creates data.csv.bz2

# Decompress
bzip2 -d data.csv.bz2

# Stream
cat data.csv | bzip2 > data.csv.bz2
```

### xz

```bash
# Compress
xz data.csv  # Creates data.csv.xz

# Decompress
xz -d data.csv.xz

# High compression
xz -9 data.csv
```

### zstd

```bash
# Compress
zstd data.csv  # Creates data.csv.zst

# Decompress
zstd -d data.csv.zst

# Level
zstd -19 data.csv  # Max compression
```

---

## Advanced Patterns

### Pipeline CSV Cleaning

```bash
# Remove empty lines, trim spaces, filter
cat raw.csv | \
  sed '/^$/d' | \
  sed 's/, /,/g' | \
  grep -v "^#" | \
  sort | uniq > clean.csv
```

### CSV to JSON

```bash
# Using jq
mlr --csv --json cat data.csv > data.json

# Using miller
mlr --csv --jlistwrap cat data.csv > data.json
```

### Parallel Processing

```bash
# Process files in parallel
parallel gzip ::: *.csv

# Map and reduce
cat huge.csv | parallel --pipe --block 10M 'awk -F, "{sum += $2} END {print sum}"'
```

### Compression + Streaming

```bash
# Compress while processing
cat huge.csv | \
  awk -F',' '{print $1, $3}' | \
  gzip > output.csv.gz

# Decompress and process
zcat huge.csv.gz | \
  grep "2025" | \
  awk '{print $1}' | sort | uniq -c
```

### File Integrity

```bash
# Generate checksum
md5sum data.csv > data.csv.md5

# Verify
md5sum -c data.csv.md5

# SHA256
sha256sum data.csv > data.csv.sha256

# Verify
sha256sum -c data.csv.sha256
```

---

## 🎯 Common One-Liners

```bash
# Count CSV records (excluding header)
wc -l < data.csv

# Get unique values
cut -d',' -f2 data.csv | sort | uniq

# Sum column
awk -F',' '{sum += $3} END {print sum}' data.csv

# Average
awk -F',' '{sum += $3; count++} END {print sum/count}' data.csv

# Find duplicates
sort data.csv | uniq -d

# Replace in bulk
find . -name "*.csv" -exec sed -i 's/old/new/g' {} \;

# Merge CSV files (add headers)
{ head -1 file1.csv; tail -n +2 file*.csv; } > merged.csv

# Top N records
head -n 11 data.csv | tail -n 10  # Skip header, get 10 rows

# Random sample
shuf -n 100 data.csv > sample.csv

# Split large file
split -l 10000 huge.csv chunk_
```

---

## 📊 Performance Tips

```bash
# Use streaming instead of loading full file
zcat huge.csv.gz | grep "pattern" > result.csv

# Parallel with GNU Parallel
cat huge.csv | parallel --pipe --block 10M 'your_command'

# Process in chunks
split -n l/4 huge.csv chunk_
parallel 'process_command {} > result_{}.csv' ::: chunk_*

# Monitor progress
pv data.csv | your_command > output.csv
```

---

**Bookmark this page for quick reference! 📍**
