# Sample Data for default_suggestions

## Add sample documents using curl:

```powershell
# Sample 1: Washing Machine
curl -X POST "http://localhost:8983/solr/default_suggestions/update?commit=true" -H "Content-Type: application/json" -d '[
  {
    "id": "1",
    "suggest_text": "Washing Machine",
    "type": "product",
    "weight": 100
  },
  {
    "id": "2",
    "suggest_text": "Washer Dryer Combo",
    "type": "product",
    "weight": 85
  },
  {
    "id": "3",
    "suggest_text": "Wash Basin",
    "type": "product",
    "weight": 40
  },
  {
    "id": "4",
    "suggest_text": "Washing Powder",
    "type": "product",
    "weight": 60
  },
  {
    "id": "5",
    "suggest_text": "Refrigerator Double Door",
    "type": "product",
    "weight": 95
  },
  {
    "id": "6",
    "suggest_text": "Fridge Single Door",
    "type": "product",
    "weight": 70
  },
  {
    "id": "7",
    "suggest_text": "Microwave Oven",
    "type": "product",
    "weight": 90
  },
  {
    "id": "8",
    "suggest_text": "Dishwasher Built-in",
    "type": "product",
    "weight": 50
  }
]'
```

## Example Queries:

### 1. Prefix search for "wash"
```
http://localhost:8983/solr/default_suggestions/suggest?q=wash*&rows=10
```

### 2. Search with minimum match
```
http://localhost:8983/solr/default_suggestions/suggest?q=washing machine&rows=10
```

### 3. Filter by type
```
http://localhost:8983/solr/default_suggestions/suggest?q=wash*&fq=type:product&rows=10
```

### 4. Sorted by weight (default behavior)
```
http://localhost:8983/solr/default_suggestions/suggest?q=fridge&rows=10
```

All queries automatically sort by weight (descending) then score.
