# Chapitre 11 : Rust — Performance & Sécurité des types

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- Parser CSV/JSON avec **serde** et **csv crates**
- Gérer **erreurs fortement typées**
- Optimiser pour **maximum performance**
- Utiliser **pattern matching**
- Compiler pour **zero-cost abstractions**

---

## 📖 Table des matières

1. [Pourquoi Rust pour CSV/JSON?](#pourquoi-rust-pour-csvjson)
2. [Setup et dépendances](#setup-et-dépendances)
3. [CSV avec le crate csv](#csv-avec-le-crate-csv)
4. [JSON avec serde_json](#json-avec-serde_json)
5. [Gestion d'erreurs Rust](#gestion-derreurs-rust)
6. [Performance et benchmarks](#performance-et-benchmarks)

---

## Pourquoi Rust pour CSV/JSON?

### Avantages Rust

| Aspect | Rust | Go | Python | Java |
|--------|------|-----|---------|------|
| **Sécurité** | ⚡⚡⚡ Garanties compilateur | ⚠️ Runtime | ❌ Dynamique | ⚠️ Nullable |
| **Performance** | ⚡⚡⚡ Native | ⚡⚡ | 🐌 Interprété | ⚡ JVM |
| **Memory safety** | ✅ Ownership | ⚠️ GC | ⚠️ GC | ✅ GC |
| **Zero-cost abstractions** | ✅ Garanti | ✅ | ❌ | ⚠️ |
| **Apprentissage** | 🔴 Difficile | 🟢 Rapide | 🟢 Très rapide | 🟡 Modéré |

### Quand utiliser Rust?
- ✅ Applications critique (finance, embarqué)
- ✅ Gestion massive de données (100GB+ CSV)
- ✅ CLI tools production
- ✅ Besoin de concurrence lockfree
- ❌ Prototypage rapide

---

## Setup et dépendances

### Créer projet

```bash
cargo new csv_json_rust
cd csv_json_rust
```

### Cargo.toml

```toml
[package]
name = "csv_json_rust"
version = "0.1.0"
edition = "2021"

[dependencies]
csv = "1.3"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
anyhow = "1.0"  # Error handling
thiserror = "1.0"

[profile.release]
opt-level = 3
lto = true  # Link Time Optimization
```

---

## CSV avec le crate csv

### Lecture simple

```rust
use std::error::Error;
use std::fs::File;

fn main() -> Result<(), Box<dyn Error>> {
    let file = File::open("data.csv")?;
    let mut reader = csv::Reader::from_reader(file);

    // Lire chaque record
    for result in reader.records() {
        let record = result?;
        println!("{:?}", record);  // StringRecord
    }

    Ok(())
}
```

### Parsing en structs (recommandé)

```rust
use csv::ReaderBuilder;
use serde::Deserialize;
use std::fs::File;

#[derive(Debug, Deserialize)]
struct User {
    id: u32,
    name: String,
    email: String,
    age: u8,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let file = File::open("users.csv")?;
    let mut reader = csv::Reader::from_reader(file);

    for result in reader.deserialize() {
        let user: User = result?;
        println!("{}: {} ({} years)", user.id, user.name, user.age);
    }

    Ok(())
}
```

### Écriture CSV

```rust
use csv::Writer;
use serde::Serialize;

#[derive(Serialize)]
struct Product {
    id: u32,
    name: String,
    price: f64,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut writer = Writer::from_path("products.csv")?;

    writer.serialize(Product {
        id: 1,
        name: "Laptop".to_string(),
        price: 999.99,
    })?;

    writer.serialize(Product {
        id: 2,
        name: "Mouse".to_string(),
        price: 29.99,
    })?;

    writer.flush()?;
    Ok(())
}
```

### Parsing custom avec types

```rust
use csv::StringRecord;
use std::str::FromStr;

#[derive(Debug)]
struct ParseError(String);

#[derive(Debug)]
struct Product {
    id: u32,
    name: String,
    price: f64,
    in_stock: bool,
}

impl FromStr for Product {
    type Err = ParseError;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let parts: Vec<&str> = s.split(',').collect();
        if parts.len() < 4 {
            return Err(ParseError("Not enough fields".to_string()));
        }

        Ok(Product {
            id: parts[0].parse().map_err(|_| ParseError("Bad ID".to_string()))?,
            name: parts[1].to_string(),
            price: parts[2].parse().map_err(|_| ParseError("Bad price".to_string()))?,
            in_stock: parts[3] == "true",
        })
    }
}
```

---

## JSON avec serde_json

### Parsing simple

```rust
use serde_json::json;

fn main() {
    // Créer JSON
    let user = json!({
        "id": 1,
        "name": "Alice",
        "email": "alice@example.com"
    });

    // Accéder valeurs
    println!("Name: {}", user["name"]);  // "Alice"
    println!("Email: {}", user["email"]);  // "alice@example.com"
}
```

### Sérialisation struct

```rust
use serde::{Deserialize, Serialize};
use serde_json;

#[derive(Serialize, Deserialize, Debug)]
struct User {
    id: u32,
    name: String,
    email: String,
    active: bool,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let user = User {
        id: 1,
        name: "Alice".to_string(),
        email: "alice@example.com".to_string(),
        active: true,
    };

    // Sérialiser
    let json_string = serde_json::to_string(&user)?;
    println!("{}", json_string);
    // {"id":1,"name":"Alice","email":"alice@example.com","active":true}

    // Pretty print
    let json_pretty = serde_json::to_string_pretty(&user)?;
    println!("{}", json_pretty);

    Ok(())
}
```

### Désérialisation

```rust
use serde_json;

fn main() -> Result<(), serde_json::Error> {
    let json_str = r#"{
        "id": 1,
        "name": "Alice",
        "email": "alice@example.com",
        "active": true
    }"#;

    let user: User = serde_json::from_str(json_str)?;
    println!("{:?}", user);

    Ok(())
}
```

### Structs imbriquées

```rust
#[derive(Serialize, Deserialize, Debug)]
struct Company {
    id: String,
    name: String,
    employees: Vec<Employee>,
}

#[derive(Serialize, Deserialize, Debug)]
struct Employee {
    name: String,
    email: String,
    department: String,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let json = r#"{
        "id": "acme",
        "name": "ACME Corp",
        "employees": [
            {"name": "Alice", "email": "alice@acme.com", "department": "IT"},
            {"name": "Bob", "email": "bob@acme.com", "department": "HR"}
        ]
    }"#;

    let company: Company = serde_json::from_str(json)?;

    for emp in &company.employees {
        println!("{} ({})", emp.name, emp.department);
    }

    Ok(())
}
```

---

## Gestion d'erreurs Rust

### Pattern matching

```rust
use std::fs::File;
use std::io;

fn read_file(path: &str) -> Result<String, io::Error> {
    std::fs::read_to_string(path)
}

fn main() {
    match read_file("data.csv") {
        Ok(content) => println!("File: {}", content),
        Err(e) => eprintln!("Error: {}", e),
    }
}
```

### Custom error types

```rust
use thiserror::Error;

#[derive(Error, Debug)]
enum ParseError {
    #[error("Invalid CSV format: {0}")]
    CsvError(String),
    
    #[error("Invalid number: {0}")]
    ParseIntError(#[from] std::num::ParseIntError),
    
    #[error("IO error: {0}")]
    IoError(#[from] std::io::Error),
}

fn parse_csv(line: &str) -> Result<(u32, String), ParseError> {
    let parts: Vec<&str> = line.split(',').collect();
    if parts.len() < 2 {
        return Err(ParseError::CsvError("Not enough fields".to_string()));
    }

    let id = parts[0].parse()?;  // ? operator auto-converts
    let name = parts[1].to_string();

    Ok((id, name))
}

fn main() {
    match parse_csv("1,Alice") {
        Ok((id, name)) => println!("{}: {}", id, name),
        Err(e) => eprintln!("Error: {}", e),
    }
}
```

### Result chaining

```rust
fn process_csv(path: &str) -> Result<Vec<User>, Box<dyn std::error::Error>> {
    let file = File::open(path)?;
    let mut reader = csv::Reader::from_reader(file);
    
    let users = reader
        .deserialize()
        .collect::<Result<Vec<_>, _>>()?;
    
    Ok(users)
}
```

---

## Performance et benchmarks

### Optimisations CSV

```rust
use csv::ReaderBuilder;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let file = std::fs::File::open("huge.csv")?;
    
    // Buffer size important pour CSV
    let mut reader = ReaderBuilder::new()
        .buffer_capacity(128 * 1024)  // 128KB buffer
        .from_reader(file);

    let mut count = 0;
    for result in reader.records() {
        let _record = result?;
        count += 1;
    }

    println!("Processed {} records", count);
    Ok(())
}
```

### Parallélisation Rayon

```rust
use rayon::prelude::*;

fn process_parallel(records: Vec<String>) -> Vec<ProcessedRecord> {
    records
        .into_par_iter()  // Parallel iterator
        .map(|record| process_record(&record))
        .collect()
}

fn process_record(s: &str) -> ProcessedRecord {
    // Processing logic
    ProcessedRecord { /* ... */ }
}
```

### Benchmark avec Criterion

```rust
// Ajouter à Cargo.toml:
// [dev-dependencies]
// criterion = "0.5"

use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn benchmark_csv_read(c: &mut Criterion) {
    c.bench_function("read csv 10k", |b| {
        b.iter(|| {
            let file = std::fs::File::open("test_10k.csv").unwrap();
            let mut reader = csv::Reader::from_reader(file);
            let mut count = 0;
            for result in reader.records() {
                let _record = result.unwrap();
                count += 1;
            }
            black_box(count)
        })
    });
}

criterion_group!(benches, benchmark_csv_read);
criterion_main!(benches);
```

### Run benchmark

```bash
cargo bench
```

---

## 🎓 Exercices pratiques

### Exercice 11.1 : Lire CSV
Lisez CSV en struct, affichez statistiques.

### Exercice 11.2 : Sérialiser JSON
Créez struct, convertissez en JSON, re-parsez.

### Exercice 11.3 : Erreurs
Implémentez custom error type avec thiserror.

### Exercice 11.4 : Performance
Benchmark Rust vs Go pour même CSV.

### Exercice 11.5 : Parallélisation
Traitez CSV gros en parallèle avec Rayon.

---

## 📚 Références

- **csv crate** : https://docs.rs/csv/
- **serde** : https://serde.rs/
- **serde_json** : https://docs.rs/serde_json/
- **Rust Book** : https://doc.rust-lang.org/book/
- **rayon** : https://docs.rs/rayon/

---

**Prêt pour R? → [Chapitre 12 : R & Data Science](./12_R_Data_Science.md)**
