# Quick Start Guide - default_suggestions

## 🚀 Example Query URLs

### Basic Autocomplete Queries

#### 1. Prefix Search: "wash"
```
http://localhost:8983/solr/default_suggestions/suggest?q=wash*&rows=10
```
**Returns:** Washing Machine, Washer Dryer Combo, Washing Powder, Wash Basin (sorted by weight)

---

#### 2. Prefix Search: "fridge"
```
http://localhost:8983/solr/default_suggestions/suggest?q=fridge&rows=10
```
**Returns:** Fridge Single Door

---

#### 3. Prefix Search: "micro"
```
http://localhost:8983/solr/default_suggestions/suggest?q=micro*&rows=10
```
**Returns:** Microwave Oven

---

#### 4. Multi-word Search: "washing machine"
```
http://localhost:8983/solr/default_suggestions/suggest?q=washing%20machine&rows=10
```
**Returns:** Washing Machine (phrase matching)

---

#### 5. Filter by Type
```
http://localhost:8983/solr/default_suggestions/suggest?q=wash*&fq=type:product&rows=10
```
**Returns:** Only products matching "wash" prefix

---

#### 6. Limit Results to 5
```
http://localhost:8983/solr/default_suggestions/suggest?q=wash*&rows=5
```
**Returns:** Top 5 results by weight

---

### Advanced Queries

#### 7. Get All Documents (Testing)
```
http://localhost:8983/solr/default_suggestions/select?q=*:*&rows=100
```

#### 8. Search Without Prefix Wildcard
```
http://localhost:8983/solr/default_suggestions/suggest?q=wash&rows=10
```
**Note:** edgeNGram allows matching even without asterisk

#### 9. Custom Sort (Override Default)
```
http://localhost:8983/solr/default_suggestions/suggest?q=wash*&sort=suggest_text%20asc&rows=10
```
**Returns:** Results sorted alphabetically

#### 10. Return All Fields
```
http://localhost:8983/solr/default_suggestions/suggest?q=wash*&fl=*,score&rows=10
```

---

## 📊 Response Format

All queries return JSON by default:

```json
{
  "responseHeader": {
    "status": 0,
    "QTime": 2
  },
  "response": {
    "numFound": 4,
    "start": 0,
    "docs": [
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
      }
    ]
  }
}
```

---

## 🔧 PowerShell Testing Commands

### Test Autocomplete
```powershell
# Test "wash" prefix
$response = Invoke-RestMethod "http://localhost:8983/solr/default_suggestions/suggest?q=wash*&rows=10"
$response.response.docs | ForEach-Object { "[$($_.weight)] $($_.suggest_text)" }

# Output:
# [100] Washing Machine
# [85] Washer Dryer Combo
# [60] Washing Powder
# [40] Wash Basin
```

### Add New Suggestion
```powershell
$newDoc = @{
    id = "9"
    suggest_text = "Water Purifier"
    type = "product"
    weight = 80
} | ConvertTo-Json

Invoke-RestMethod -Uri "http://localhost:8983/solr/default_suggestions/update?commit=true" `
    -Method Post -ContentType "application/json" -Body $newDoc
```

### Delete Document
```powershell
$deleteDoc = @{ delete = @{ id = "9" } } | ConvertTo-Json
Invoke-RestMethod -Uri "http://localhost:8983/solr/default_suggestions/update?commit=true" `
    -Method Post -ContentType "application/json" -Body $deleteDoc
```

---

## ✅ Verification

**Core Status:**
```
http://localhost:8983/solr/admin/cores?action=STATUS&core=default_suggestions&wt=json
```

**Admin UI:**
```
http://localhost:8983/solr/#/default_suggestions/query
```

---

## 📝 Current Test Data

| ID | Suggest Text | Type | Weight |
|----|--------------|------|--------|
| 1 | Washing Machine | product | 100 |
| 2 | Washer Dryer Combo | product | 85 |
| 3 | Wash Basin | product | 40 |
| 4 | Washing Powder | product | 60 |
| 5 | Refrigerator Double Door | product | 95 |
| 6 | Fridge Single Door | product | 70 |
| 7 | Microwave Oven | product | 90 |
| 8 | Dishwasher Built-in | product | 50 |

---

## 🎯 Key Features Demonstrated

✓ **Prefix Matching** - Instant results as user types  
✓ **Weight-Based Sorting** - Most important suggestions first  
✓ **Fast Response** - Typically <10ms query time  
✓ **Type Filtering** - Filter by category/type  
✓ **Multi-Word Support** - Handles phrases correctly  
✓ **Production Ready** - Minimal config, maximum performance
