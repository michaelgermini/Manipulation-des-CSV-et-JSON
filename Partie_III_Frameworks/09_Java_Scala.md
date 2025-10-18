# Chapitre 9 : Java / Scala — CSV, JSON & Spark DataFrame

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- Parser **CSV et JSON** avec **Jackson**
- Utiliser **OpenCSV** pour manipulation CSV avancée
- Construire **Spark DataFrames** (distribuées)
- Transformer données avec **Spark SQL**
- Optimiser performances sur gros volumes

---

## 📖 Table des matières

1. [Jackson — JSON ultra-rapide](#jackson--json-ultra-rapide)
2. [OpenCSV — CSV robuste](#opencsv--csv-robuste)
3. [Spark DataFrame — Distribué](#spark-dataframe--distribué)
4. [Transformations Spark SQL](#transformations-spark-sql)
5. [Performance et optimisation](#performance-et-optimisation)

---

## Jackson — JSON ultra-rapide

### Installation Maven

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.16.0</version>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-csv</artifactId>
    <version>2.16.0</version>
</dependency>
```

### Parsing JSON simple

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.*;

ObjectMapper mapper = new ObjectMapper();

// Lire JSON
String json = "{\"id\":1,\"name\":\"Alice\",\"email\":\"alice@example.com\"}";
Map<String, Object> user = mapper.readValue(json, Map.class);
System.out.println(user.get("name"));  // Alice

// Lire array JSON
String jsonArray = "[{\"id\":1},{\"id\":2}]";
List<Map> users = mapper.readValue(jsonArray, List.class);
System.out.println(users.size());  // 2
```

### Types structures

```java
// Classe modèle
public class User {
    public int id;
    public String name;
    public String email;
    public int age;
    
    // Getters/setters
}

// Parsing avec types
ObjectMapper mapper = new ObjectMapper();
User user = mapper.readValue(jsonString, User.class);

// Écriture
String output = mapper.writeValueAsString(user);

// Pretty-print
String pretty = mapper.writerWithDefaultPrettyPrinter()
                      .writeValueAsString(user);
```

### Streaming JSON

```java
import com.fasterxml.jackson.core.*;

// Pour gros fichiers
JsonFactory factory = new JsonFactory();
JsonParser parser = factory.createParser(new File("huge.json"));

while (parser.nextToken() != JsonToken.END_OBJECT) {
    if ("users".equals(parser.currentName())) {
        parser.nextToken();  // [
        while (parser.nextToken() != JsonToken.END_ARRAY) {
            User user = mapper.readValue(parser, User.class);
            // Process user
        }
    }
}
```

---

## OpenCSV — CSV robuste

### Installation Maven

```xml
<dependency>
    <groupId>com.opencsv</groupId>
    <artifactId>opencsv</artifactId>
    <version>5.8</version>
</dependency>
```

### Lecture simple

```java
import com.opencsv.CSVReader;
import java.io.FileReader;

try (CSVReader reader = new CSVReader(new FileReader("data.csv"))) {
    String[] line;
    while ((line = reader.readNext()) != null) {
        System.out.println(Arrays.toString(line));
    }
}
```

### Avec en-têtes

```java
import com.opencsv.bean.CsvToBeanBuilder;
import java.io.FileReader;
import java.util.List;

// Classe map
public class User {
    @CsvBindByName
    private int id;
    
    @CsvBindByName
    private String name;
    
    @CsvBindByName
    private String email;
    
    // Getters/setters
}

// Parse avec types
List<User> users = new CsvToBeanBuilder<User>(new FileReader("data.csv"))
    .withType(User.class)
    .withIgnoreLeadingWhiteSpace(true)
    .build()
    .parse();

users.forEach(u -> System.out.println(u.getName()));
```

### Écriture CSV

```java
import com.opencsv.bean.StatefulBeanToCsv;
import com.opencsv.bean.StatefulBeanToCsvBuilder;
import java.io.FileWriter;

List<User> users = new ArrayList<>();
users.add(new User(1, "Alice", "alice@example.com"));
users.add(new User(2, "Bob", "bob@example.com"));

try (FileWriter writer = new FileWriter("output.csv")) {
    StatefulBeanToCsv<User> csvWriter = 
        new StatefulBeanToCsvBuilder<User>(writer)
        .withSeparator(',')
        .build();
    csvWriter.write(users);
}
```

### Personnalisation

```java
import com.opencsv.CSVFormat;
import com.opencsv.CSVParser;
import com.opencsv.CSVReader;

// Délimiteur personnalisé
CSVFormat format = CSVFormat.DEFAULT.withDelimiter(';');

try (CSVReader reader = new CSVReader(
    new FileReader("data.csv"),
    CSVParser.DEFAULT_SEPARATOR,
    CSVParser.DEFAULT_QUOTE_CHARACTER,
    1  // Ignorer 1 ligne (headers)
)) {
    // ...
}
```

---

## Spark DataFrame — Distribué

### Installation (PySpark, mais Java aussi)

```xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-core_2.13</artifactId>
    <version>3.4.0</version>
</dependency>
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-sql_2.13</artifactId>
    <version>3.4.0</version>
</dependency>
```

### Setup Spark

```java
import org.apache.spark.sql.SparkSession;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;

SparkSession spark = SparkSession
    .builder()
    .appName("CSVApp")
    .master("local[4]")  // 4 cores localement
    .getOrCreate();

// Lecture CSV
Dataset<Row> df = spark.read()
    .option("header", "true")
    .option("inferSchema", "true")
    .csv("data.csv");

df.show();
df.printSchema();
```

### Transformations courantes

```java
// Sélectionner colonnes
df.select("name", "email").show();

// Filtrer
df.filter(df.col("age").gt(25)).show();

// Ajouter colonne
df.withColumn("salary_doubled", df.col("salary").multiply(2)).show();

// GroupBy + aggregate
df.groupBy("department")
  .agg(org.apache.spark.sql.functions.avg("salary"))
  .show();

// Trier
df.sort(df.col("salary").desc()).show();

// Joindre
Dataset<Row> df1 = spark.read().csv("users.csv");
Dataset<Row> df2 = spark.read().csv("orders.csv");
df1.join(df2, df1.col("id").equalTo(df2.col("user_id")))
   .show();
```

### Écriture

```java
// Parquet (recommandé pour data warehouse)
df.write()
  .mode(SaveMode.Overwrite)
  .parquet("output.parquet");

// CSV
df.write()
  .option("header", "true")
  .mode(SaveMode.Overwrite)
  .csv("output.csv");

// JSON
df.write()
  .mode(SaveMode.Overwrite)
  .json("output.json");
```

---

## Transformations Spark SQL

### SQL sur DataFrames

```java
// Créer vue temporaire
df.createOrReplaceTempView("users");

// SQL queries
Dataset<Row> result = spark.sql(
    "SELECT name, age FROM users WHERE age > 25 ORDER BY salary DESC"
);
result.show();

// JOIN SQL
spark.sql(
    "SELECT u.name, o.amount " +
    "FROM users u " +
    "JOIN orders o ON u.id = o.user_id " +
    "WHERE o.amount > 100"
).show();

// GROUP BY SQL
spark.sql(
    "SELECT department, AVG(salary) as avg_salary, COUNT(*) as count " +
    "FROM users " +
    "GROUP BY department " +
    "ORDER BY avg_salary DESC"
).show();
```

### Scala version (plus idiomatique)

```scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession
  .builder()
  .appName("CSVApp")
  .master("local[*]")
  .getOrCreate()

// Lecture
val df = spark.read
  .option("header", "true")
  .option("inferSchema", "true")
  .csv("data.csv")

// Transformations Scala
df.filter($"age" > 25)
  .select("name", "salary")
  .groupBy("department")
  .agg(avg("salary"))
  .show()

// Ou SQL
df.createOrReplaceTempView("users")
spark.sql("SELECT department, AVG(salary) FROM users GROUP BY department").show()
```

---

## Performance et optimisation

### Partitioning

```java
// Écriture partitionnée (pour big data)
df.write()
  .mode(SaveMode.Overwrite)
  .partitionBy("department", "year")
  .parquet("output.parquet");

// Lecture (Spark lit seulement partitions nécessaires)
Dataset<Row> df2 = spark.read()
  .parquet("output.parquet")
  .filter("department = 'IT' AND year = 2025");
```

### Caching

```java
// Cache DataFrame en mémoire (si accès multiple)
df.cache();
df.show();  // Première fois : évaluation
df.show();  // Deuxième fois : cache

// Remove cache
df.unpersist();
```

### Broadcast (petits tables)

```java
import org.apache.spark.sql.functions.broadcast;

Dataset<Row> smallDf = spark.read().csv("small.csv");
Dataset<Row> largeDf = spark.read().csv("large.csv");

// Broadcast smallDf (envoyer tous les workers)
largeDf.join(broadcast(smallDf), "id").show();
```

### Exemple : Pipeline complet

```java
// 1. Charger
Dataset<Row> df = spark.read()
    .option("header", "true")
    .option("inferSchema", "true")
    .csv("sales.csv");

// 2. Nettoyer
df = df.filter(df.col("amount").gt(0));

// 3. Transformer
df = df.withColumn("year", year(df.col("date")))
       .withColumn("profit", df.col("amount").multiply(0.15));

// 4. Agréger
Dataset<Row> summary = df.groupBy("year", "region")
    .agg(
        org.apache.spark.sql.functions.sum("amount").as("total_sales"),
        org.apache.spark.sql.functions.avg("profit").as("avg_profit"),
        org.apache.spark.sql.functions.count("*").as("transactions")
    );

// 5. Exporter
summary.write()
    .mode(SaveMode.Overwrite)
    .option("header", "true")
    .csv("summary.csv");
```

---

## 🎓 Exercices pratiques

### Exercice 9.1 : Jackson JSON
Parsez JSON complexe, mettez en types Java.

### Exercice 9.2 : OpenCSV
Lisez CSV, filtrez, écrivez résultat.

### Exercice 9.3 : Spark simple
Chargez CSV, faites transformations, affichez résultats.

### Exercice 9.4 : Spark SQL
Réalisez JOIN et GROUP BY en SQL.

### Exercice 9.5 : Pipeline Spark
CSV → Nettoyer → Transformer → Exporter Parquet.

---

## 📚 Références

- **Jackson Docs** : https://github.com/FasterXML/jackson
- **OpenCSV Docs** : https://opencsv.sourceforge.net/
- **Spark SQL** : https://spark.apache.org/docs/latest/sql-programming-guide.html
- **Spark Python (PySpark)** : https://spark.apache.org/docs/latest/api/python/

---

**Prêt pour Go? → [Chapitre 10 : Go](./10_Go.md)**
