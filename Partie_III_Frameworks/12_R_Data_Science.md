# Chapitre 12 : R & Data Science — dplyr, ggplot2, Shiny

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- Charger et manipuler CSV/JSON avec **tidyverse**
- Visualiser avec **ggplot2**
- Créer dashboards interactifs avec **Shiny**
- Faire du statistique avec **R**
- Exporter pour publication

---

## 📖 Table des matières

1. [Setup R](#setup-r)
2. [tidyverse & dplyr](#tidyverse--dplyr)
3. [Visualisations avec ggplot2](#visualisations-avec-ggplot2)
4. [Dashboard Shiny](#dashboard-shiny)
5. [JSON & APIs](#json--apis)
6. [R Markdown](#r-markdown)

---

## Setup R

### Installation packages

```r
# Install core packages
install.packages(c("tidyverse", "jsonlite", "readr"))

# tidyverse inclut:
# - dplyr (data manipulation)
# - ggplot2 (visualization)
# - tidyr (reshaping)
# - readr (CSV reading)
# - purrr (functional programming)

library(tidyverse)
```

### Charger CSV/JSON

```r
library(readr)
library(jsonlite)

# CSV
df <- read_csv("data.csv")

# CSV avec paramètres
df <- read_csv(
  "data.csv",
  skip = 1,          # Skip first row
  col_types = cols(
    id = col_integer(),
    name = col_character(),
    price = col_double(),
    active = col_logical()
  )
)

# JSON
json_data <- fromJSON("data.json")
df <- as_tibble(json_data)

# JSON Lines
json_lines <- readLines("data.jsonl") %>%
  map_df(fromJSON)
```

---

## tidyverse & dplyr

### Sélectionner et filtrer

```r
library(dplyr)

df %>%
  # Sélectionner colonnes
  select(id, name, email) %>%
  # Filtrer lignes
  filter(age > 18) %>%
  # Trier
  arrange(desc(salary))

# Sélection avancée
df %>%
  select(starts_with("price_")) %>%  # Colonnes commençant par
  select(where(is.numeric))           # Colonnes numériques
```

### Transformation et création

```r
df %>%
  # Ajouter colonne
  mutate(
    full_name = paste(first_name, last_name),
    age_group = case_when(
      age < 18 ~ "Child",
      age < 65 ~ "Adult",
      TRUE ~ "Senior"
    ),
    created_year = year(created_date)
  ) %>%
  # Renommer
  rename(
    customer_id = id,
    customer_name = name
  )
```

### Agrégations et groupage

```r
df %>%
  group_by(country) %>%
  summarise(
    count = n(),
    avg_salary = mean(salary, na.rm = TRUE),
    total_revenue = sum(revenue),
    .groups = 'drop'
  ) %>%
  arrange(desc(total_revenue))

# Multi-level aggregation
df %>%
  group_by(year, month, country) %>%
  summarise(
    sales = sum(amount),
    transactions = n(),
    avg_transaction = mean(amount)
  ) %>%
  ungroup()
```

### Jointures

```r
# Inner join
combined <- left_join(users, orders, by = c("id" = "user_id"))

# Multiple keys
combined <- left_join(
  sales,
  customers,
  by = c("customer_id" = "id", "region" = "sales_region")
)

# Anti-join (find non-matches)
orphaned <- anti_join(orders, users, by = c("user_id" = "id"))
```

---

## Visualisations avec ggplot2

### Graphiques basiques

```r
library(ggplot2)

# Scatter plot
ggplot(df, aes(x = age, y = salary)) +
  geom_point(aes(color = country), alpha = 0.6) +
  geom_smooth(method = "lm", se = FALSE) +
  labs(
    title = "Salary by Age",
    x = "Age (years)",
    y = "Salary ($)",
    color = "Country"
  ) +
  theme_minimal()

# Histogram
ggplot(df, aes(x = salary)) +
  geom_histogram(bins = 30, fill = "steelblue") +
  facet_wrap(~country) +
  labs(title = "Salary Distribution")

# Box plot
ggplot(df, aes(x = country, y = salary)) +
  geom_boxplot() +
  geom_jitter(width = 0.1, alpha = 0.3) +
  labs(title = "Salary by Country")
```

### Time series

```r
df %>%
  mutate(date = as.Date(created_date)) %>%
  group_by(date) %>%
  summarise(daily_revenue = sum(amount)) %>%
  ggplot(aes(x = date, y = daily_revenue)) +
    geom_line() +
    geom_smooth(span = 0.3) +
    scale_x_date(date_labels = "%Y-%m") +
    labs(title = "Daily Revenue Trend")
```

### Faceted visualization

```r
ggplot(df, aes(x = value)) +
  geom_histogram(fill = "coral") +
  facet_grid(country ~ year) +  # 2D facet
  theme(strip.text = element_text(size = 10))
```

---

## Dashboard Shiny

### Simple app

```r
# app.R
library(shiny)
library(tidyverse)

# Load data
df <- read_csv("sales.csv")

# UI
ui <- fluidPage(
  titlePanel("Sales Dashboard"),
  
  sidebarLayout(
    sidebarPanel(
      selectInput(
        "country_input",
        "Select Country:",
        choices = unique(df$country)
      ),
      sliderInput(
        "price_range",
        "Price Range:",
        min = 0,
        max = 10000,
        value = c(0, 10000)
      )
    ),
    
    mainPanel(
      tabsetPanel(
        tabPanel("Summary", 
          textOutput("summary_text"),
          plotOutput("plot_sales")
        ),
        tabPanel("Details",
          dataTableOutput("table_data")
        )
      )
    )
  )
)

# Server
server <- function(input, output) {
  
  # Reactive data
  filtered_data <- reactive({
    df %>%
      filter(country == input$country_input) %>%
      filter(price >= input$price_range[1], price <= input$price_range[2])
  })
  
  # Summary stats
  output$summary_text <- renderText({
    data <- filtered_data()
    paste("Total sales:", nrow(data), "| Avg price:", round(mean(data$price), 2))
  })
  
  # Plot
  output$plot_sales <- renderPlot({
    filtered_data() %>%
      ggplot(aes(x = category, y = price)) +
      geom_boxplot(fill = "steelblue") +
      labs(title = "Price by Category")
  })
  
  # Table
  output$table_data <- renderDataTable({
    filtered_data() %>% head(100)
  })
}

# Run
shinyApp(ui, server)
```

Run with: `shiny::runApp()`

---

## JSON & APIs

### Parsing JSON avancé

```r
library(jsonlite)

# Parse avec validation
json_str <- '{
  "users": [
    {"id": 1, "name": "Alice", "email": "alice@example.com"},
    {"id": 2, "name": "Bob", "email": "bob@example.com"}
  ]
}'

data <- fromJSON(json_str)
users_df <- as_tibble(data$users)

# API calls
library(httr)

response <- GET("https://api.example.com/users")
content <- content(response, as = "text")
users <- fromJSON(content)
```

---

## R Markdown

### Report template

```r
---
title: "CSV Analysis Report"
author: "Data Team"
date: "`r format(Sys.Date(), '%Y-%m-%d')`"
output:
  html_document:
    toc: true
    theme: bootstrap
  pdf_document:
    latex_engine: xelatex
---

# Introduction

This report analyzes sales data.

# Data Loading

\`\`\`{r message=FALSE}
library(tidyverse)
df <- read_csv("sales.csv")
head(df)
\`\`\`

# Summary Statistics

\`\`\`{r}
df %>%
  summarise(
    count = n(),
    avg_sales = mean(amount),
    total = sum(amount)
  )
\`\`\`

# Visualization

\`\`\`{r, fig.width=10}
df %>%
  ggplot(aes(x = date, y = amount)) +
  geom_line() +
  labs(title = "Sales Over Time")
\`\`\`

\`\`\`

Generate: `rmarkdown::render("report.Rmd")`

---

## 🎓 Exercices pratiques

### Exercice 12.1 : Charger & transformer
Chargez CSV, filtrez et agrégez.

### Exercice 12.2 : Visualisation
Créez 5 graphiques différents avec ggplot2.

### Exercice 12.3 : Shiny app
Créez dashboard interactif simple.

### Exercice 12.4 : JSON
Parsez API JSON et créez visualisations.

### Exercice 12.5 : Report
Générez R Markdown report complet.

---

## 📚 Références

- **tidyverse** : https://www.tidyverse.org/
- **ggplot2** : https://ggplot2.tidyverse.org/
- **Shiny** : https://shiny.rstudio.com/
- **R for Data Science** : https://r4ds.had.co.nz/

---

**Prêt pour multi-langages? → [Chapitre 13 : Multi-langages](./13_Multi_Langages.md)**
