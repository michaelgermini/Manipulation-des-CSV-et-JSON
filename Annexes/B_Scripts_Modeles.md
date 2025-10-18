# Annexe B : Template Scripts — Réutilisable Code Snippets

## 📋 Table des matières

1. [Python Templates](#python-templates)
2. [Bash Scripts](#bash-scripts)
3. [Node.js Templates](#nodejs-templates)
4. [SQL Queries](#sql-queries)

---

## Python Templates

### CSV Reader with Error Handling

```python
import csv
import logging
from pathlib import Path

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def read_csv_safe(filepath: str, encoding='utf-8'):
    """
    Safely read CSV with error handling and encoding detection.
    """
    try:
        with open(filepath, 'r', encoding=encoding) as f:
            reader = csv.DictReader(f)
            for row in reader:
                yield row
    except FileNotFoundError:
        logger.error(f"File not found: {filepath}")
    except UnicodeDecodeError:
        logger.warning(f"Encoding error with {encoding}, trying latin-1")
        with open(filepath, 'r', encoding='latin-1') as f:
            reader = csv.DictReader(f)
            for row in reader:
                yield row

# Usage
for row in read_csv_safe('data.csv'):
    print(row)
```

### CSV Cleaner Template

```python
import pandas as pd
import re
from typing import Callable, Dict

class CSVCleaner:
    def __init__(self, filepath: str):
        self.df = pd.read_csv(filepath)
        self.original_shape = self.df.shape
    
    def remove_duplicates(self, subset=None, keep='first'):
        """Remove duplicate rows"""
        self.df = self.df.drop_duplicates(subset=subset, keep=keep)
        return self
    
    def handle_missing(self, strategy='drop', fill_value=None):
        """Handle missing values"""
        if strategy == 'drop':
            self.df = self.df.dropna()
        elif strategy == 'fill':
            self.df = self.df.fillna(fill_value)
        return self
    
    def normalize_column(self, col: str, func: Callable):
        """Apply normalization function"""
        self.df[col] = self.df[col].apply(func)
        return self
    
    def export(self, output_path: str):
        """Save cleaned data"""
        self.df.to_csv(output_path, index=False)
        print(f"Saved: {self.original_shape} -> {self.df.shape}")

# Usage
cleaner = CSVCleaner('raw.csv')
cleaner.remove_duplicates(subset=['email']) \
    .handle_missing(strategy='drop') \
    .normalize_column('name', str.lower) \
    .normalize_column('email', str.strip) \
    .export('clean.csv')
```

### JSON Processor Template

```python
import json
from pathlib import Path
from typing import List, Dict

class JSONProcessor:
    def __init__(self, filepath: str = None):
        self.data = []
        if filepath:
            self.load(filepath)
    
    def load(self, filepath: str):
        """Load JSON file"""
        with open(filepath, 'r') as f:
            self.data = json.load(f)
    
    def filter(self, condition: callable) -> 'JSONProcessor':
        """Filter records"""
        self.data = [item for item in self.data if condition(item)]
        return self
    
    def map(self, transform: callable) -> 'JSONProcessor':
        """Transform records"""
        self.data = [transform(item) for item in self.data]
        return self
    
    def to_csv(self, output_path: str, fields: List[str]):
        """Convert to CSV"""
        import csv
        with open(output_path, 'w', newline='') as f:
            writer = csv.DictWriter(f, fieldnames=fields)
            writer.writeheader()
            writer.writerows(self.data)
    
    def save(self, filepath: str):
        """Save as JSON"""
        with open(filepath, 'w') as f:
            json.dump(self.data, f, indent=2)

# Usage
processor = JSONProcessor('input.json')
processor.filter(lambda x: x.get('age', 0) > 18) \
    .map(lambda x: {**x, 'name': x['name'].upper()}) \
    .to_csv('output.csv', fields=['name', 'email', 'age'])
```

### Data Validation Template

```python
from dataclasses import dataclass
from typing import List, Callable
import re

@dataclass
class ValidationRule:
    field: str
    validator: Callable[[str], bool]
    error_msg: str

class DataValidator:
    def __init__(self):
        self.rules: List[ValidationRule] = []
        self.errors = []
    
    def add_rule(self, field: str, validator: Callable, error_msg: str):
        """Add validation rule"""
        self.rules.append(ValidationRule(field, validator, error_msg))
        return self
    
    def validate(self, row: dict) -> bool:
        """Validate single row"""
        is_valid = True
        for rule in self.rules:
            if rule.field in row:
                if not rule.validator(row[rule.field]):
                    self.errors.append(f"Row {row}: {rule.error_msg}")
                    is_valid = False
        return is_valid
    
    def validate_csv(self, filepath: str) -> tuple:
        """Validate entire CSV"""
        import csv
        valid_rows = 0
        with open(filepath, 'r') as f:
            reader = csv.DictReader(f)
            for row in reader:
                if self.validate(row):
                    valid_rows += 1
        return valid_rows, len(self.errors)

# Usage
validator = DataValidator()
validator.add_rule(
    'email', 
    lambda x: re.match(r'^[\w\.-]+@[\w\.-]+\.\w+$', x),
    'Invalid email format'
).add_rule(
    'age',
    lambda x: 18 <= int(x) <= 120,
    'Age out of range'
)

valid, errors = validator.validate_csv('data.csv')
print(f"Valid rows: {valid}, Errors: {errors}")
```

---

## Bash Scripts

### CSV Processing Pipeline

```bash
#!/bin/bash
# csv_process.sh - CSV processing pipeline

set -euo pipefail

INPUT_FILE="${1:?Usage: $0 <input.csv>}"
OUTPUT_FILE="${2:-output.csv}"

# Color codes
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m' # No Color

echo -e "${GREEN}Processing CSV: $INPUT_FILE${NC}"

# Step 1: Remove empty lines
sed '/^$/d' "$INPUT_FILE" > temp1.csv

# Step 2: Trim spaces after commas
sed 's/, /,/g' temp1.csv > temp2.csv

# Step 3: Remove duplicate lines (keep header)
{
  head -1 temp2.csv
  tail -n +2 temp2.csv | sort -u
} > temp3.csv

# Step 4: Sort by first column
sort -t',' -k1 temp3.csv > "$OUTPUT_FILE"

# Cleanup
rm -f temp1.csv temp2.csv temp3.csv

echo -e "${GREEN}Done! Output: $OUTPUT_FILE${NC}"
```

### Backup Script with Rotation

```bash
#!/bin/bash
# backup.sh - Daily backup with rotation

set -euo pipefail

BACKUP_DIR="/backup/data"
SOURCE_DIR="/data/csv"
RETENTION_DAYS=30

mkdir -p "$BACKUP_DIR"

# Create backup
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/backup_$TIMESTAMP.tar.gz"

echo "Creating backup: $BACKUP_FILE"
tar -czf "$BACKUP_FILE" "$SOURCE_DIR"

# Calculate checksum
sha256sum "$BACKUP_FILE" > "${BACKUP_FILE}.sha256"
echo "Checksum: $(cat ${BACKUP_FILE}.sha256)"

# Remove old backups
echo "Cleaning up old backups..."
find "$BACKUP_DIR" -name "backup_*.tar.gz" -mtime +$RETENTION_DAYS -delete

echo "✅ Backup complete"
```

### JSON to CSV Converter

```bash
#!/bin/bash
# json_to_csv.sh - Convert JSON to CSV

set -euo pipefail

INPUT="${1:?Usage: $0 <input.json>}"
OUTPUT="${2:-output.csv}"

if command -v jq &> /dev/null; then
    # Extract headers
    HEADERS=$(jq -r '.[0] | keys_unsorted | @csv' "$INPUT")
    
    # Extract data
    {
        echo "$HEADERS"
        jq -r '.[] | [.[] | tostring] | @csv' "$INPUT"
    } > "$OUTPUT"
    
    echo "✅ Converted: $INPUT -> $OUTPUT"
else
    echo "❌ jq not installed"
    exit 1
fi
```

### Log Aggregation

```bash
#!/bin/bash
# aggregate_logs.sh - Aggregate and analyze JSON logs

set -euo pipefail

LOG_DIR="${1:-.}"
OUTPUT="${2:-log_report.json}"

echo "Aggregating logs from: $LOG_DIR"

# Combine and aggregate
jq -s 'group_by(.level) | map({
  level: .[0].level,
  count: length,
  messages: map(.message) | unique
})' "$LOG_DIR"/*.json > "$OUTPUT"

echo "✅ Report saved: $OUTPUT"
```

---

## Node.js Templates

### CSV Streaming Processor

```javascript
const fs = require('fs');
const { parse } = require('csv-parse');
const { stringify } = require('csv-stringify');

async function processCSV(inputFile, outputFile) {
    const records = [];
    
    fs.createReadStream(inputFile)
        .pipe(parse({
            columns: true,
            skip_empty_lines: true
        }))
        .on('data', (row) => {
            // Transform
            row.processed = true;
            row.timestamp = new Date().toISOString();
            records.push(row);
        })
        .on('end', () => {
            // Write output
            const output = fs.createWriteStream(outputFile);
            stringify(records, { header: true }).pipe(output);
            console.log(`✅ Processed: ${records.length} records`);
        })
        .on('error', (err) => {
            console.error('Error:', err.message);
        });
}

processCSV('input.csv', 'output.csv');
```

### JSON API Client

```javascript
const https = require('https');
const fs = require('fs');

class APIClient {
    constructor(baseURL, apiKey) {
        this.baseURL = baseURL;
        this.apiKey = apiKey;
    }
    
    async get(path) {
        return new Promise((resolve, reject) => {
            const options = {
                method: 'GET',
                headers: {
                    'Authorization': `Bearer ${this.apiKey}`,
                    'Content-Type': 'application/json'
                }
            };
            
            https.get(`${this.baseURL}${path}`, options, (res) => {
                let data = '';
                res.on('data', chunk => data += chunk);
                res.on('end', () => resolve(JSON.parse(data)));
            }).on('error', reject);
        });
    }
    
    async saveToFile(path, filename) {
        const data = await this.get(path);
        fs.writeFileSync(filename, JSON.stringify(data, null, 2));
        console.log(`✅ Saved: ${filename}`);
    }
}

// Usage
const client = new APIClient('https://api.example.com', 'YOUR_KEY');
client.saveToFile('/data/users', 'users.json');
```

---

## SQL Queries

### Common Data Analysis Patterns

```sql
-- Row number and ranking
SELECT 
    id, name, salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) as rank,
    RANK() OVER (ORDER BY salary DESC) as salary_rank
FROM employees;

-- Running total
SELECT 
    date, amount,
    SUM(amount) OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) as running_total
FROM transactions;

-- Window functions
SELECT 
    customer_id, order_date, total,
    LAG(total) OVER (PARTITION BY customer_id ORDER BY order_date) as prev_order,
    LEAD(total) OVER (PARTITION BY customer_id ORDER BY order_date) as next_order
FROM orders;

-- Pivot table
SELECT 
    country,
    SUM(CASE WHEN year=2023 THEN revenue END) as "2023",
    SUM(CASE WHEN year=2024 THEN revenue END) as "2024"
FROM sales
GROUP BY country;
```

---

## 📦 Quick Template Installation

Save these as files and use immediately:

```bash
# Python
curl https://example.com/csv_cleaner.py -o csv_cleaner.py
python csv_cleaner.py data.csv

# Bash
chmod +x csv_process.sh
./csv_process.sh input.csv output.csv

# Node.js
npm install csv-parse csv-stringify
node process.js
```

---

**Copy, modify, and reuse these templates! 🚀**
