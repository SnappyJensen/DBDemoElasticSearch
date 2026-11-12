Denne guide tager udgangspunkt i Elasticsearch quick start guide. https://www.elastic.co/docs/deploy-manage/deploy/self-managed/local-development-installation-quickstart

DEL1:
Brug Linux. I.E. hvis du på windows brug WSL.

DEL2:
Hent , in Linux terminalen, elasticsearch quick start image: curl -fsSL https://elastic.co/start-local | sh

DEL3:
Notere det username og password terminalen spytter ude.

DEL4:
Elasticsearch køre på port 9200 og Kibana port 5601. Får at komme ind på nogen af dem skal du bruge det username og password der blev genreret.

DEL5:
Åben Kibana og log ind.

DEL6:
Don't panic. Klik på console lavest på siden.

DEL7:
Sample time. Lav et indeks. Indeks er et slags skabalon som vi laver vores dokumenter ud fra.

PUT foods
{
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "category": { "type": "keyword" },
      "cuisine": { "type": "keyword" },
      "description": { "type": "text" },
      "ingredients": { "type": "keyword" },
      "price": { "type": "float" },
      "calories": { "type": "integer" },
      "vegetarian": { "type": "boolean" },
      "vegan": { "type": "boolean" },
      "gluten_free": { "type": "boolean" },
      "rating": { "type": "float" },
      "created_at": { "type": "date" }
    }
  }
}

DEL8:
Nu hvor vi har et indeks kan vi lave noget data.
POST _bulk
{"index":{"_index":"foods","_id":"1"}}
{"name":"Margherita Pizza","category":"Pizza","cuisine":"Italian","description":"Classic Italian pizza with tomato sauce, fresh mozzarella and basil.","ingredients":["tomato","mozzarella","basil","flour","olive oil"],"price":12.50,"calories":850,"vegetarian":true,"vegan":false,"gluten_free":false,"rating":4.8,"created_at":"2026-01-10"}
{"index":{"_index":"foods","_id":"2"}}
{"name":"Classic Cheeseburger","category":"Burger","cuisine":"American","description":"Juicy beef burger with cheddar cheese, lettuce, tomato and special sauce.","ingredients":["beef","cheddar","lettuce","tomato","onion","bread"],"price":14.00,"calories":950,"vegetarian":false,"vegan":false,"gluten_free":false,"rating":4.5,"created_at":"2026-01-11"}
{"index":{"_index":"foods","_id":"3"}}
{"name":"Vegan Buddha Bowl","category":"Salad","cuisine":"Asian","description":"Healthy bowl with quinoa, avocado, chickpeas and fresh vegetables.","ingredients":["quinoa","avocado","chickpeas","spinach","carrot"],"price":11.00,"calories":420,"vegetarian":true,"vegan":true,"gluten_free":true,"rating":4.9,"created_at":"2026-01-12"}
{"index":{"_index":"foods","_id":"4"}}
{"name":"Chicken Tikka Masala","category":"Curry","cuisine":"Indian","description":"Tender chicken in a creamy tomato-based curry sauce served with rice.","ingredients":["chicken","tomato","cream","ginger","garlic","rice"],"price":16.50,"calories":720,"vegetarian":false,"vegan":false,"gluten_free":true,"rating":4.7,"created_at":"2026-01-13"}
{"index":{"_index":"foods","_id":"5"}}
{"name":"Sushi Salmon Roll","category":"Sushi","cuisine":"Japanese","description":"Fresh salmon and avocado wrapped in seasoned sushi rice and seaweed.","ingredients":["salmon","rice","avocado","seaweed"],"price":13.50,"calories":380,"vegetarian":false,"vegan":false,"gluten_free":true,"rating":4.6,"created_at":"2026-01-14"}
{"index":{"_index":"foods","_id":"6"}}
{"name":"Greek Salad","category":"Salad","cuisine":"Greek","description":"Traditional Greek salad with feta cheese, cucumber, olives and tomatoes.","ingredients":["feta","cucumber","tomato","olives","red onion"],"price":9.50,"calories":320,"vegetarian":true,"vegan":false,"gluten_free":true,"rating":4.4,"created_at":"2026-01-15"}
{"index":{"_index":"foods","_id":"7"}}
{"name":"Beef Tacos","category":"Mexican","cuisine":"Mexican","description":"Soft tacos filled with seasoned beef, salsa, avocado and coriander.","ingredients":["beef","tortilla","avocado","tomato","coriander"],"price":10.50,"calories":560,"vegetarian":false,"vegan":false,"gluten_free":false,"rating":4.6,"created_at":"2026-01-16"}
{"index":{"_index":"foods","_id":"8"}}
{"name":"Chocolate Brownie","category":"Dessert","cuisine":"American","description":"Rich chocolate brownie with a soft centre and crispy edges.","ingredients":["chocolate","butter","sugar","flour","eggs"],"price":6.00,"calories":450,"vegetarian":true,"vegan":false,"gluten_free":false,"rating":4.9,"created_at":"2026-01-17"}
{"index":{"_index":"foods","_id":"9"}}
{"name":"Vegetable Pad Thai","category":"Noodles","cuisine":"Thai","description":"Stir-fried rice noodles with vegetables, peanuts and tamarind sauce.","ingredients":["rice noodles","tofu","peanuts","bean sprouts","lime"],"price":12.00,"calories":590,"vegetarian":true,"vegan":true,"gluten_free":true,"rating":4.5,"created_at":"2026-01-18"}
{"index":{"_index":"foods","_id":"10"}}
{"name":"Grilled Ribeye Steak","category":"Steak","cuisine":"French","description":"Premium grilled ribeye steak served with roasted potatoes and herbs.","ingredients":["beef","potato","rosemary","garlic","butter"],"price":24.00,"calories":880,"vegetarian":false,"vegan":false,"gluten_free":true,"rating":4.8,"created_at":"2026-01-19"}

DEL9:
Nu kan vi gøre search delen af search database. Denne leder efter documents hvor vegetarian er true

GET foods/_search
{
  "query": {
    "term": {
      "vegetarian": true
    }
  }
}

DEL10:
Den her leder efter kylling.

GET foods/_search
{
  "query": {
    "term": {
      "ingredients": "chicken"
    }
  }

DEL11:
Leder på price

GET foods/_search
{
  "query": {
    "range": {
      "price": {
        "lt": 15
      }
    }
  }
}

DEL12:
Kigger på description

GET foods/_search
{
  "query": {
    "match": {
      "description": "Italian tomato mozzarella"
    }
  }
}

DEL13:
kigger på size og cuisine

GET foods/_search
{
  "size": 0,
  "aggs": {
    "cuisines": {
      "terms": {
        "field": "cuisine"
      }
    }
  }
}

DEL14:
Enderlig, hvis du vil delete et dokument via ID,

DELETE foods/_doc/5

DEL15:
Fjerner index.

DELETE foods