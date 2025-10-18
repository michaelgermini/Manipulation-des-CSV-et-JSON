# Chapitre 8 : JavaScript / Node.js — Écosystème CSV & JSON

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- Utiliser **csv-parse** et **csv-stringify** pour CSV
- Manipuler JSON avec **lodash** et **ramda**
- Streamer gros fichiers avec **JSONStream**
- Gérer **async/await** pour opérations I/O
- Construire pipelines Node.js robustes

---

## 📖 Table des matières

1. [Installation et dépendances](#installation-et-dépendances)
2. [CSV: Lecture et écriture](#csv-lecture-et-écriture)
3. [JSON: Parsing et transformation](#json-parsing-et-transformation)
4. [Streaming: Pour gros fichiers](#streaming-pour-gros-fichiers)
5. [Programmation fonctionnelle](#programmation-fonctionnelle)
6. [Pipelines complets](#pipelines-complets)

---

## Installation et dépendances

### Setup projet

```bash
mkdir my_data_project
cd my_data_project
npm init -y

# Dépendances essentielles
npm install csv-parse csv-stringify csv \
            lodash ramda \
            @fast-csv/parse @fast-csv/format \
            jsonstream ndjson \
            papaparse
```

### Package.json minimal

```json
{
  "name": "csv-json-tools",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node index.js",
    "test": "jest"
  },
  "dependencies": {
    "csv-parse": "^5.4.0",
    "csv-stringify": "^6.3.0",
    "@fast-csv/parse": "^4.3.6",
    "lodash": "^4.17.21",
    "ramda": "^0.29.0",
    "ndjson": "^2.0.0"
  }
}
```

---

## CSV: Lecture et écriture

### csv-parse — Parser CSV robuste

```javascript
import fs from 'fs';
import { parse } from 'csv-parse';

// Lecture simple
const csvStream = fs.createReadStream('data.csv')
  .pipe(parse({ 
    columns: true,  // Headers dans objet
    delimiter: ',',
    encoding: 'utf8'
  }));

for await (const record of csvStream) {
  console.log(record);  // { id: '1', name: 'Alice', ... }
}
```

### Avec types et validation

```javascript
import { parse } from 'csv-parse';
import fs from 'fs';

const parser = parse({
  columns: true,
  cast: {
    // Conversion automatique
    id: (value) => parseInt(value),
    age: (value) => parseInt(value),
    salary: (value) => parseFloat(value),
    date: (value) => new Date(value),
    active: (value) => value === 'true'
  },
  skip_empty_lines: true,
  skip_records_with_error: true
});

fs.createReadStream('data.csv').pipe(parser);
parser.on('readable', function() {
  let record;
  while (record = parser.read()) {
    console.log(`User: ${record.name}, Age: ${record.age}`);
  }
});
```

### csv-stringify — Écriture CSV

```javascript
import { stringify } from 'csv-stringify';
import fs from 'fs';

const data = [
  { id: 1, name: 'Alice', email: 'alice@example.com' },
  { id: 2, name: 'Bob', email: 'bob@example.com' }
];

stringify(data, {
  header: true,
  columns: ['id', 'name', 'email'],
  encoding: 'utf8'
}, (err, output) => {
  if (err) throw err;
  fs.writeFileSync('output.csv', output);
  console.log('CSV written');
});
```

### Fast-CSV — Alternative performante

```javascript
import fs from 'fs';
import { parse, format } from '@fast-csv/parse';

// Lecture rapide
fs.createReadStream('data.csv')
  .pipe(parse({ headers: true }))
  .on('data', row => console.log(row))
  .on('error', err => console.error(err))
  .on('end', () => console.log('Done'));

// Écriture rapide
const output = fs.createWriteStream('output.csv');
format({ headers: true })
  .pipe(output);

format().write({ id: 1, name: 'Alice' });
format().write({ id: 2, name: 'Bob' });
format().end();
```

---

## JSON: Parsing et transformation

### JSON natif (bas niveau)

```javascript
// Parsing
const json = JSON.parse(fs.readFileSync('data.json', 'utf8'));
console.log(json);

// Stringify
const str = JSON.stringify(json, null, 2);  // Pretty-print
fs.writeFileSync('output.json', str);

// Problème: Charge tout en mémoire!
// Solution: Streaming (voir section Streaming)
```

### Lodash — Manipulation fonctionnelle

```javascript
import _ from 'lodash';

const users = [
  { id: 1, name: 'Alice', dept: 'IT', salary: 50000 },
  { id: 2, name: 'Bob', dept: 'HR', salary: 45000 },
  { id: 3, name: 'Charlie', dept: 'IT', salary: 60000 }
];

// Filtrer
const itDept = _.filter(users, { dept: 'IT' });

// Map (transformer)
const names = _.map(users, 'name');  // ['Alice', 'Bob', 'Charlie']

// GroupBy
const byDept = _.groupBy(users, 'dept');
// { IT: [...], HR: [...] }

// Sum
const totalSalary = _.sumBy(users, 'salary');  // 155000

// Chain (composable)
const result = _.chain(users)
  .filter(u => u.salary > 45000)
  .map(u => ({ name: u.name, dept: u.dept }))
  .orderBy(['salary'], ['desc'])
  .value();
```

### Ramda — Programmation fonctionnelle pure

```javascript
import * as R from 'ramda';

const users = [
  { id: 1, name: 'Alice', salary: 50000 },
  { id: 2, name: 'Bob', salary: 45000 },
  { id: 3, name: 'Charlie', salary: 60000 }
];

// Ramda = immutable, composable
const getNames = R.pluck('name');
const names = getNames(users);  // ['Alice', 'Bob', 'Charlie']

// Pipe (composition)
const highEarners = R.pipe(
  R.filter(u => u.salary > 48000),
  R.map(u => ({ name: u.name, salary: u.salary })),
  R.sortBy(u => -u.salary)
)(users);

// Lens (getter/setter fonctionnel)
const nameLens = R.lensProp('name');
const upperName = R.over(nameLens, R.toUpper);
console.log(upperName(users[0]));
// { id: 1, name: 'ALICE', salary: 50000 }
```

---

## Streaming: Pour gros fichiers

### JSONStream — Gros fichiers JSON

```javascript
import JSONStream from 'JSONStream';
import fs from 'fs';

// Fichier: [{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]

const stream = fs.createReadStream('huge.json')
  .pipe(JSONStream.parse('*'))  // Parse chaque élément
  .on('data', obj => {
    console.log('Processing:', obj);
  })
  .on('end', () => console.log('Done'));
```

### NDJSON — JSON Lines (un objet par ligne)

```javascript
import ndjson from 'ndjson';
import fs from 'fs';

// Fichier: huge.jsonl
// {"id":1,"name":"Alice"}
// {"id":2,"name":"Bob"}

fs.createReadStream('huge.jsonl')
  .pipe(ndjson.parse())
  .on('data', obj => console.log(obj))
  .on('error', err => console.error(err));

// Écrire NDJSON
const output = fs.createWriteStream('output.jsonl');
output.write(JSON.stringify({ id: 1, name: 'Alice' }) + '\n');
output.write(JSON.stringify({ id: 2, name: 'Bob' }) + '\n');
output.end();
```

### Transform Streams — Pipeline

```javascript
import { Transform } from 'stream';
import fs from 'fs';
import { parse } from 'csv-parse';
import { stringify } from 'csv-stringify';

// Transform: Filtrer et enrichir
const transformStream = new Transform({
  objectMode: true,
  transform(chunk, encoding, callback) {
    if (chunk.age > 25) {
      chunk.category = 'senior';
      this.push(chunk);
    }
    callback();
  }
});

// Pipeline: Read → Parse CSV → Transform → Stringify → Write
fs.createReadStream('input.csv')
  .pipe(parse({ columns: true }))
  .pipe(transformStream)
  .pipe(stringify({ header: true }))
  .pipe(fs.createWriteStream('output.csv'))
  .on('finish', () => console.log('Done'));
```

---

## Programmation fonctionnelle

### Async/Await + Streams

```javascript
import fs from 'fs';
import { parse } from 'csv-parse';

async function processCSV(filePath) {
  const data = [];
  
  const parser = fs.createReadStream(filePath)
    .pipe(parse({ columns: true }));
  
  return new Promise((resolve, reject) => {
    parser.on('readable', function() {
      let record;
      while (record = this.read()) {
        data.push(record);
      }
    });
    
    parser.on('end', () => resolve(data));
    parser.on('error', reject);
  });
}

// Usage
const users = await processCSV('data.csv');
console.log(`Loaded ${users.length} users`);
```

### Promisify avec util

```javascript
import { parseFile } from 'csv-parse';
import { promisify } from 'util';
import fs from 'fs';

const parseAsync = promisify(parseFile);

async function loadCSV(path) {
  const records = await parseAsync(fs.createReadStream(path), {
    columns: true
  });
  return records;
}

const data = await loadCSV('data.csv');
```

---

## Pipelines complets

### Pipeline 1 : Transformer CSV → JSON

```javascript
import fs from 'fs';
import { parse } from 'csv-parse';
import ndjson from 'ndjson';

// CSV → JSONL
fs.createReadStream('data.csv')
  .pipe(parse({ 
    columns: true,
    cast: {
      id: Number,
      age: Number,
      salary: Number
    }
  }))
  .pipe(ndjson.stringify())
  .pipe(fs.createWriteStream('output.jsonl'));
```

### Pipeline 2 : Filtrer + Agréger

```javascript
import fs from 'fs';
import { parse } from 'csv-parse';
import { Transform } from 'stream';
import _ from 'lodash';

const aggregator = new Transform({
  objectMode: true,
  async transform(chunk, encoding, callback) {
    // Accumulate (en vrai, utiliser reduce)
    this.records = this.records || [];
    this.records.push(chunk);
    callback();
  },
  async flush(callback) {
    const grouped = _.groupBy(this.records, 'department');
    const result = Object.entries(grouped).map(([dept, recs]) => ({
      department: dept,
      count: recs.length,
      avgSalary: _.meanBy(recs, 'salary')
    }));
    
    for (const item of result) {
      this.push(item);
    }
    callback();
  }
});

fs.createReadStream('employees.csv')
  .pipe(parse({ columns: true, cast: { salary: Number } }))
  .pipe(aggregator)
  .pipe(ndjson.stringify())
  .pipe(fs.createWriteStream('summary.jsonl'));
```

### Pipeline 3 : API → Validate → Store

```javascript
import fetch from 'node-fetch';
import { stringify } from 'csv-stringify';
import fs from 'fs';
import _ from 'lodash';

async function fetchAndProcess() {
  const response = await fetch('https://api.example.com/users');
  const data = await response.json();
  
  // Validate
  const validated = data.filter(u => 
    u.email && u.email.includes('@')
  );
  
  // Transform
  const rows = validated.map(u => ({
    id: u.id,
    name: u.name,
    email: u.email,
    created: new Date(u.created_at).toISOString()
  }));
  
  // Write CSV
  stringify(rows, {
    header: true,
    columns: ['id', 'name', 'email', 'created']
  }, (err, output) => {
    if (err) throw err;
    fs.writeFileSync('users.csv', output);
    console.log(`Exported ${rows.length} users`);
  });
}

await fetchAndProcess();
```

---

## 🎓 Exercices pratiques

### Exercice 8.1 : CSV basic
Lisez CSV, filtrez par colonne, écrivez résultat.

### Exercice 8.2 : JSON Transform
Chargez JSON, utilisez lodash/ramda pour transformer.

### Exercice 8.3 : Streaming
Créez fichier CSV 100MB, streamez-le.

### Exercice 8.4 : Pipeline complet
CSV → Filtrer → Agréger → JSONL.

---

## 📚 Références

- **csv-parse Docs** : https://csv.js.org/parse/
- **Lodash Docs** : https://lodash.com/
- **Ramda Docs** : https://ramdajs.com/
- **Node.js Streams** : https://nodejs.org/en/docs/guides/backpressuring-in-streams/

---

**Prêt pour Java/Scala? → [Chapitre 9 : Java/Scala](./09_Java_Scala.md)**
