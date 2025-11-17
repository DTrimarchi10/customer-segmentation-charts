# **BEP Segmentation – Data Input & Output Overview**

This document briefly explains the end‑to‑end data flow used in the BEP Segmentation API.  

---

# **1. SearchAPI Overview**

The segmentation workflow uses **SearchAPI** to retrieve structured business information from Google.

🔗 https://www.searchapi.io/


SearchAPI offers several engines. Only two are used in segmentation API (Google Maps and Google).


---


### **google_maps Engine**

<details>
<summary><strong>Click to expand</strong></summary>

The `google_maps` engine returns structured business listings:

```
search_metadata  
search_parameters  
search_information  
local_results: [ ... ]
people_also_search_for: [ ... ] - optional
```

Local Results Includes:

- Business name  
- Address  
- Categories  
- Operational status  
- Ratings / reviews  
- Hours  
- Phone / website  
- Menus  
- GPS coordinates  
- Review summaries  

**Raw Examples (linked files):**  
- sample_google_maps_search_result.rtf  
- sample_google_maps_searchapi_result_bad_business_name.rtf  

</details>

---

### **google Engine**

<details>
<summary><strong>Click to expand</strong></summary>

The `google` engine returns:

```
search_metadata  
search_parameters  
search_information 
knowledge_graph: { ... }
```

Knowledge Graph / Organic Results Includes:

- Business name  
- Category  
- Description  
- Operations / closure  
- Reviews  
- Menu links  
- Organic results [ ... ]

Sometimes the organic results are in the main result dictionary without the knowledge graph. 

**Raw Example:**  
- sample_google_seardchapi_result_bad_business_name.rtf  

</details>

---

# **2. How the Segmentation API Uses SearchAPI Data**

## **2.1 Simplified Process:**

### **Step 1 — Receive Raw SearchAPI Data**

Retrieve google_maps result from Search API. 

*Note*: Loop back and repeat process with google result if google_maps info is insufficient for openai segementation (some businesses do not have a google place set up or have very limited information on google maps). 


### **Step 2 — Clean & Normalize**

<details>
<summary><strong>Click to expand cleanup details</strong></summary>

Removed:

- Reviewer URLs  
- Reviewer photos  
- Deep image structures  
- Google place IDs  
- Large metadata blocks  

Retained:

- Business name  
- Address  
- Categories  
- Description / snippets  
- Review summaries  
- Phone  
- Website  
- Hours  
- Operational status  
- is_entity_closed  

</details>

### **Step 3 — OpenAI Segmentation**

OpenAI uses:

- Sector/Segment hierarchy  
- Multi-sector rules  
- Restaurant attribute rules  
- Cleaned search data  

---

## **2.2 Classification Code Snippet**

```python
import requests
url_base = "https://prod-api.customersegmentation.buyersedgeplatform.com"

headers = {
    "x-api-key": API_KEY,    #API KEY for BEP Segmentation API
    "Content-Type": "application/json"
}

url = url_base + "/classify"
payload = {
    "platformclientid": "CL-1099217",
    "name": "Bushwallers",
    "address": "209 N Market St, Frederick MD 21701 US",
    "new_search": False,
    "new_segmentation": False
}

response = requests.post(url, json=payload, headers=headers)
```

## **2.3 Final Output Example**

<details>
<summary><strong>Click to expand example output</strong></summary>

```json
{
  "platformclientid": "CL-1099217",
  "name": "Bushwallers",
  "address": "209 N Market St, Frederick MD 21701 US",
  "sector": "Restaurant",
  "segment": "Full Service",
  "attributes": {
    "Service Type": "Full Service",
    "Cuisine Type": "British & Irish",
    "Franchise": "No",
    "Alcohol Served": "Yes",
    "Restaurant Type": "Bar / Pub"
  },
  "is_entity_closed": "",
  "operational_status": "",
  "business_description": "Bushwaller's is a longstanding Irish pub in Frederick, MD, offering a cozy atmosphere...",
  "search_data_date": "2025-08-28",
  "segmentation_date": "2025-11-10",
  "user": "testing",
  "status": "Success"
}
```

</details>

---

## **2.4 Field Descriptions (API Fields)**

<details>
<summary><strong>Click to view API field descriptions</strong></summary>

| Field | Description |
|-------|-------------|
| platformclientid | The platform ID passed in during the API call. |
| name | The business name passed into the API call. |
| address | The address passed in during the API call. |
| sector | Selected sector from hierarchy. |
| segment | Selected segment from hierarchy. |
| attributes | Restaurant-only attributes. |
| business_description | One-sentence model-generated description. |
| is_entity_closed | Boolean or empty. |
| operational_status | “temporarily closed”, “permanently closed”, or blank. |
| search_data_date | Date SearchAPI query was performed. |
| segmentation_date | Date segmentation was performed. |
| user | The API user determined from x-api-key. |
| status | “Success” or error state. |

`segmentation_api_fields.xlsx`  

</details>


---

# **3. Retrieving Segmentation Data From S3**

Data is stored in S3 as JSON after each segmentation run.  
You can retrieve this data using the `/get_data` endpoint.

---

## **3.1 Fields Stored in S3**

<details>
<summary><strong>Click to view S3 fields</strong></summary>

| Field |
|-------|
| platformclientid |
| name |
| address |
| status |
| sector |
| segment |
| attributes |
| business_description |
| is_entity_closed |
| operational_status |
| search_engine |
| new_search |
| search_date |
| new_segmentation |
| segmentation_date |
| data_passed_to_openai |
| search_result_idx |
| searchapi_data_google_maps |
| searchapi_data_google |
| user |

</details>

---

## **3.2 Retrieve Full S3 Data Row Code Snippet**

```python
import requests
url_base = "https://prod-api.customersegmentation.buyersedgeplatform.com"

headers = {
    "x-api-key": API_KEY,
    "Content-Type": "application/json"
}

url = url_base + "/get_data"
payload = {
    "platform_ids": ["CL-4182492", ...], #multiple locations allowed
    "full_search_data": True
}

response = requests.post(url=url, json=payload, headers=headers)
```

## **3.3 Final Output Example**

<details>
<summary><strong>Click to expand example output</strong></summary>

```json
{"data": [{"platformclientid": "CL-4182492",
   "name": "Seoul Garden",
   "address": "4701 Atlantic Ave, Raleigh NC 27604 US",
   "status": "Success",
   "sector": "Restaurant",
   "segment": "Full Service",
   "attributes": {
      "Service Type": "Full Service", 
      "Cuisine Type": "Korean", 
      "Franchise": "No", 
      "Alcohol Served": "Yes", 
      "Restaurant Type": "Standard", 
      "Primary Menu Focus": ["BBQ", "Seafood"]
      },
   "business_description": "Seoul Garden is a highly-rated Korean restaurant in Raleigh, NC, offering a casual dining experience with a variety of traditional Korean dishes including stews, noodles, seafood, tofu, and BBQ.",
   "is_entity_closed": "",
   "operational_status": "",
   "search_engine": "google_maps",
   "new_search": "True",
   "search_date": "2025-09-09",
   "new_segmentation": "True",
   "segmentation_date": "2025-09-09",
   "data_passed_to_openai": {"id": "CL-4182492", "name": "Seoul Garden", "address": "4701 Atlantic Ave, Raleigh NC 27604 US", "websearch": {"title": "Seoul Garden", "description": "Casual option for Korean specialties. Traditional Korean stews, noodles, seafood, tofu & BBQ fare in a simply appointed atmosphere.", "address": "4701 Atlantic Ave, Raleigh, NC 27604", "located_in": "Millbrook Collection", "phone": "+1 919-850-9984", "price": "$10\\u201320", "price_description": "$10 to $20", "rating": 4.5, "reviews": 1600, "reviews_histogram": {"1": 81, "2": 39, "3": 83, "4": 261, "5": 1136}, "website": "http://seoulgardennc.com/", "domain": "seoulgardennc.com", "type": "Korean restaurant", "types": ["Korean restaurant", "Korean barbecue restaurant", "Restaurant"], "menu": {"link": "http://seoulgardennc.com/", "source": "seoulgardennc.com"}, "open_hours": {"tuesday": "Closed", "wednesday": "11\\u202fAM\\u20139:30\\u202fPM", "thursday": "11\\u202fAM\\u20139:30\\u202fPM", "friday": "11\\u202fAM\\u20139:30\\u202fPM", "saturday": "11\\u202fAM\\u20139:30\\u202fPM", "sunday": "12\\u20139:30\\u202fPM", "monday": "11\\u202fAM\\u20139:30\\u202fPM"}, "review_results": ["I would like to give five stars but I give four because of the toilets. The food is delicious, one of the best Korean food I have ever had. But the restroom!!!!!! You may have put the urinal and hand washing sink side by side!!!!!! but you should think of a barrier between them. I think this is unhy...", "Absolutely Unacceptable Experience \\u2013 Vegetarian Given Beef\\n\\nI am beyond disappointed and frankly appalled by this restaurant. I placed an order clearly marked as vegetarian and requested NO BEEF. Despite this, what arrived was unmistakably beef. This is not just a mistake\\u2014it\\u2019s a complete disregard f...", "A lovely korean restaurant with amazing bbq. If you haven\'t tried korean bbq definitely give it a try here! Super friendly staff. I had some amazing haemul bokkeum, which was incredible. Their rice cakes were phenomenal. The fried chicken was also yummy. Lots of good side dishes as well.", "My first time doing Korean bbq. The restaurant is clean, large and nicely decorated. The experience was fun. The server was polite, he brought us to a table immediately, brought menus, took order and food arrived quickly. He forgot the rice so we went for it ourselves. We waited a little to get the ...", "The food at Seoul Garden is truly wonderful, but the customer service ruins the experience. During lunch hour, it is a disaster. They are always understaffed, usually leaving one waitress to handle dine-in tables and to-go orders at the same time. If you have a lunch break, do not expect to get back..."], "business_tags": ["alcohol", "beer", "catering", "comfort food", "delivery", "dessert", "dine-in", "dinner", "dinner reservations recommended", "fast service", "good for groups", "good for kids", "good for solo dining", "great tea selection", "hard liquor", "healthy options", "high chairs available", "kids\' menu", "lunch", "popular for dinner", "popular for lunch", "popular with tourists", "quiet", "reservations", "small plates", "table service", "takeout", "trendy", "vegetarian dishes", "wine"], "segment_hint": "Korean restaurant with dine-in, takeout"}},
   "search_result_idx": "0", 
   "searchapi_data_google_maps": {"search_metadata": {"id": "search_JBAO2bkaoPuNRK5VW4n7XajE", "status": "Success", "created_at": "2025-09-09T10:59:02Z", "request_time_taken": 0.74, "parsing_time_taken": 0.02, "total_time_taken": 0.76, "request_url": "https://www.google.com/maps/search/Seoul+Garden%2C+4701+Atlantic+Ave%2C+Raleigh+NC+27604+US?hl=en", "html_url": "https://www.searchapi.io/api/v1/searches/search_JBAO2bkaoPuNRK5VW4n7XajE.html", "json_url": "https://www.searchapi.io/api/v1/searches/search_JBAO2bkaoPuNRK5VW4n7XajE"}, "search_parameters": {"engine": "google_maps", "q": "Seoul Garden, 4701 Atlantic Ave, Raleigh NC 27604 US", "google_domain": "google.com", "hl": "en"}, "search_information": {"query_displayed": "Seoul Garden, 4701 Atlantic Ave, Raleigh NC 27604 US", "state": "Showing results for original spelling."}, "local_results": [{"position": 1, "ludocid": "14609391802476021868", "place_id": "ChIJLxceeAdZrIkRbEgWw3D8vso", "kgmid": "/g/1tdbrr2c", "data_id": "0x89ac5907781e172f:0xcabefc70c316486c", "title": "Seoul Garden", "description": "Casual option for Korean specialties. Traditional Korean stews, noodles, seafood, tofu & BBQ fare in a simply appointed atmosphere.", "address": "4701 Atlantic Ave, Raleigh, NC 27604", "located_in": "Millbrook Collection", "phone": "+1 919-850-9984", "plus_code": "R9XW+JQ Raleigh, North Carolina", "price": "$10\\u201320", "price_description": "$10 to $20", "rating": 4.5, "reviews": 1600, "reviews_histogram": {"1": 81, "2": 39, "3": 83, "4": 261, "5": 1136}, "reviews_link": "https://search.google.com/local/reviews?placeid=ChIJLxceeAdZrIkRbEgWw3D8vso&q=Seoul+Garden,+4701+Atlantic+Ave,+Raleigh+NC+27604+US&authuser=0&hl=en&gl=US", "website": "http://seoulgardennc.com/", "domain": "seoulgardennc.com", "gps_coordinates": {"latitude": 35.84901, "longitude": -78.603061}, "type": "Korean restaurant", "types": ["Korean restaurant", "Korean barbecue restaurant", "Restaurant"], "menu": {"link": "http://seoulgardennc.com/", "source": "seoulgardennc.com"}, "open_state": "Closed", "hours": "Closed \\u22c5 Opens 11\\u202fAM Wed", "open_hours": {"tuesday": "Closed", "wednesday": "11\\u202fAM\\u20139:30\\u202fPM", "thursday": "11\\u202fAM\\u20139:30\\u202fPM", "friday": "11\\u202fAM\\u20139:30\\u202fPM", "saturday": "11\\u202fAM\\u20139:30\\u202fPM", "sunday": "12\\u20139:30\\u202fPM", "monday": "11\\u202fAM\\u20139:30\\u202fPM"}, "reservation_links": [{"text": "Order online", "link": "https://www.google.com/viewer/chooseprovider?mid=/g/1tdbrr2c&g2lbs=AO8LyOLEytp6DWPQrARd82XDGlh6j00odCLCy6B6zx4jjQQhz7RjbJULUr3mJiysUO62F3MoI_WucHOBFe8JXzrs4kN9zcjD3w%3D%3D&hl=en-US&gl=us&fo_m=MfohQo559jFvMUOzJVpjPL1YMfZ3bInYwBDuMfaXTPp5KXh-&utm_source=tactile&gei=dgjAaM7iJZSWwbkPx96UKA&ei=dgjAaM7iJZSWwbkPx96UKA&fo_s=OA,SOE&opi=79508299&ebb=1&cs=0&foub=mcpp"}], "thumbnail": "https://lh3.googleusercontent.com/gps-cs-s/AC9h4novdlREhW5liMBHRvJvWsrfPnDwKMOHIGhC1rnbxUG2ynD4mQJVYzu0DzTsqW3Td7lCTA7UngR_PoFebtTy23Q1ahLenaeQgoBlPwXH6pJUANUwXYlt5laP6Wb185IB0WJZXfVI=w114-h86-k-no", "images": [{"title": "All", "thumbnail": "https://lh3.googleusercontent.com/gps-cs-s/AC9h4novdlREhW5liMBHRvJvWsrfPnDwKMOHIGhC1rnbxUG2ynD4mQJVYzu0DzTsqW3Td7lCTA7UngR_PoFebtTy23Q1ahLenaeQgoBlPwXH6pJUANUwXYlt5laP6Wb185IB0WJZXfVI=w397-h298-k-no"}, {"title": "Latest", "thumbnail": "https://lh3.googleusercontent.com/gps-cs-s/AC9h4npD--QngImJubYk-XgyECVBCGVykXREr-iBojlwwjoL0au2gkJwYGkjz3pez0zn0Hd6wMExPj3JjXfsqD5oLjSta1BNqMedDw4YSxo347EYJTHt9zWLFa0vG8mScz33kRwyJalQ13S_CMs=w224-h298-k-no"}, {"title": "Videos", "thumbnail": "https://lh3.googleusercontent.com/gps-cs-s/AC9h4nqVdhunG3CDsEFmRujqzD7p3SQ8f_R-vtueXe3vhV2eTxGH8TYbVW8-jcutkf61XlMQC6B5t7m9gMZkWbMdEVYju_-mbNV3745XXSLD8Xzgu3pq6dBGi6x6gG9Bwa6RtOjCYuoU7A=w529-h298-k-no"}, {"title": "Menu", "thumbnail": "https://lh3.googleusercontent.com/p/AF1QipPjb3QBwJHCnC6x4PIaw-hzXMACvqGbTQrxESc2=w224-h302-k-no"}, {"title": "Food & drink", "thumbnail": "https://lh3.googleusercontent.com/p/AF1QipPPW09qm3BdqKH6hgBPezF-zdXTHOqXAGlfcbTx=w447-h298-k-no"}, {"title": "Vibe", "thumbnail": "https://lh3.googleusercontent.com/gps-cs-s/AC9h4nqmNNpxKOj3doB-JHqEDfmmKyATdzIFuOBx3713s7xKZBGFJwbQzQG_B4miLNWIgxGGS0gzjzPUnvYtjyOQBhO5gyqn8bjd1X_Q3q7vrcKfW0U67mo51MYlfGlSvutivHTgy_fgJg=w645-h298-k-no"}, {"title": "Empanada", "thumbnail": "https://lh3.googleusercontent.com/gps-cs-s/AC9h4nqarGqlAf26H21_3OtP7usgVUnxQOZDtFaWbsXzurL54Z0dk9BuWWzn7ZXe7tbSBiJ1RYM5DuCoGVEXWNTxp334yH-GufNQvSn0vyaj7KFEU3ldtuCZIOQCgELiXkhMxtn5VwrX=w397-h298-k-no"}, {"title": "Meat", "thumbnail": "https://lh3.googleusercontent.com/p/AF1QipPDw9Qu9a_6QHQXEdJ1Z8AXhA7xUcC-hc0K92Mo=w447-h298-k-no"}, {"title": "Bibimbap", "thumbnail": "https://lh3.googleusercontent.com/gps-cs-s/AC9h4noCLwuH8rMVNFBuuLTHArU0dRwcYchpPEA5UBlPRYi1WK7vVCJhBHijY_vNePJa-w5-hOVYU7emQ1Ql3hw-vj76ndFl-1NX5iTPdvA79cT1uFqvhvlYb8C4tJ7eQb7aL9JrXTU=w397-h298-k-no"}, {"title": "Mochi", "thumbnail": "https://lh3.googleusercontent.com/p/AF1QipNg4FadMLVjb5het07od3d8UT0VMFNx1gS5FtbB=w447-h298-k-no"}, {"title": "Bento", "thumbnail": "https://lh3.googleusercontent.com/gps-cs-s/AC9h4nqWdgLLxbTZNbeyh1ak4wzpJjPM2lZ-Gz1Fhj083KoMXldcgAVnuon45VEbVL9LAxrNfVaBnDvarsCsF6tfZhKBwepTvkv0qW8u3-uE5-l_RB0SIT_4PvH6ANgY8Gjt1fqW-AQ=w224-h298-k-no"}, {"title": "Pajeon", "thumbnail": "https://lh3.googleusercontent.com/gps-cs-s/AC9h4nrTO4uke6gbZdCWVSwx1GDkFmqWAodMmqKvM3Y_OmNJu_7O2whTz5Rbjs_n_ieraCOPkk5oAEmSMIGJMySf2SR9BeSs3d-6v90KBefJQ7qKoashvFD7756xuCVK47JzlQEEri8=w397-h298-k-no"}, {"title": "Soup", "thumbnail": "https://lh3.googleusercontent.com/p/AF1QipP1sMOYf72AbwSLITU6iwGAUUkWGKAbOxMitZ5n=w447-h298-k-no"}, {"title": "Bulgogi", "thumbnail": "https://lh3.googleusercontent.com/p/AF1QipOkjX5mhEJhR5aHhIKE7OIwTr1NgOse2PK2VTXL=w447-h298-k-no"}, {"title": "Kimchi", "thumbnail": "https://lh3.googleusercontent.com/gps-cs-s/AC9h4nrw6awa5Enjwaxw0p6wqPIJBRBEXkIO2R3Zzqgh37A5e3nL8IFVs7T72MapyseRdmRIAQxQciukJQh3nxqX0T25cKt1XBNHFZ5JTSfQdMYxVaautmye-0F16Uyz9wzVX05crnY=w397-h298-k-no"}, {"title": "Orange chicken", "thumbnail": "https://lh3.googleusercontent.com/gps-cs-s/AC9h4np3HphOESBK3NOElz8ZI6UKUvaIydEyy5xLXmhAqoeWJf63TCAfkIfOTvcE6gTOOgoRMdG_ouX-bGQ3-J-tfVo1SsRwYrERYb2Z3zvMg3tIKw4wAudEFNypSlIHsMf2c2dhfa0=w397-h298-k-no"}, {"title": "By owner", "thumbnail": "https://lh3.googleusercontent.com/p/AF1QipM3aBmANxQujHYqOBvKxUwx3mxpCF7ufjquQt2u=w447-h298-k-no"}, {"title": "Street View & 360\\u00b0", "thumbnail": "https://lh3.googleusercontent.com/gps-cs-s/AC9h4noY86Nowc7mdxCXgYu_eS3qIFSwt4s0Ybd4DTdr_KQ7k-RsU7h2fG9zrL1r-1bvBegaqxH_MBSjhbHMXl7QRQtJSBoJHlHe7i_xinKNtydvMTrsVQoFvPrdW-SKcSN9dNxCG9HC=w224-h298-k-no-pi0-ya340-ro0-fo100"}], "extensions": [{"title": "Service options", "items": [{"title": "Delivery", "value": "Offers delivery"}, {"title": "Takeout", "value": "Offers takeout"}, {"title": "Dine-in", "value": "Serves dine-in"}]}, {"title": "Highlights", "items": [{"title": "Fast service", "value": "Has fast service"}, {"title": "Great tea selection", "value": "Has great tea selection"}]}, {"title": "Popular for", "items": [{"title": "Lunch", "value": "Popular for lunch"}, {"title": "Dinner", "value": "Popular for dinner"}, {"title": "Solo dining", "value": "Good for solo dining"}]}, {"title": "Accessibility", "items": [{"title": "Wheelchair accessible entrance", "value": "Has wheelchair accessible entrance"}, {"title": "Wheelchair accessible parking lot", "value": "Has wheelchair accessible parking lot"}, {"title": "Wheelchair accessible restroom", "value": "Has wheelchair accessible restroom"}, {"title": "Wheelchair accessible seating", "value": "Has wheelchair accessible seating"}]}, {"title": "Offerings", "items": [{"title": "Alcohol", "value": "Serves alcohol"}, {"title": "Beer", "value": "Serves beer"}, {"title": "Comfort food", "value": "Serves comfort food"}, {"title": "Hard liquor", "value": "Serves hard liquor"}, {"title": "Healthy options", "value": "Serves healthy options"}, {"title": "Small plates", "value": "Serves small plates"}, {"title": "Vegetarian options", "value": "Serves vegetarian dishes"}, {"title": "Wine", "value": "Serves wine"}]}, {"title": "Dining options", "items": [{"title": "Lunch", "value": "Serves lunch"}, {"title": "Dinner", "value": "Serves dinner"}, {"title": "Catering", "value": "Has catering"}, {"title": "Dessert", "value": "Serves dessert"}, {"title": "Seating", "value": "Has seating"}, {"title": "Table service", "value": "Has table service"}]}, {"title": "Amenities", "items": [{"title": "Restroom", "value": "Has restroom"}, {"title": "Wi-Fi", "value": "Has Wi-Fi"}]}, {"title": "Atmosphere", "items": [{"title": "Casual", "value": "Casual"}, {"title": "Cozy", "value": "Cozy"}, {"title": "Quiet", "value": "Quiet"}, {"title": "Trendy", "value": "Trendy"}]}, {"title": "Crowd", "items": [{"title": "Groups", "value": "Good for groups"}, {"title": "Tourists", "value": "Popular with tourists"}]}, {"title": "Planning", "items": [{"title": "Dinner reservations recommended", "value": "Dinner reservations recommended"}, {"title": "Accepts reservations", "value": "Accepts reservations"}]}, {"title": "Payments", "items": [{"title": "Credit cards", "value": "Accepts credit cards"}, {"title": "Debit cards", "value": "Accepts debit cards"}, {"title": "NFC mobile payments", "value": "Accepts NFC mobile payments"}]}, {"title": "Children", "items": [{"title": "Good for kids", "value": "Good for kids"}, {"title": "High chairs", "value": "High chairs available"}, {"title": "Kids\' menu", "value": "Has kids\' menu"}]}, {"title": "Parking", "items": [{"title": "Free parking lot", "value": "Free parking lot"}, {"title": "Free street parking", "value": "Free street parking"}]}], "questions_and_answers": {"question": {"text": ",do you serve lunch boxes?", "date": "7 years ago", "language": "en"}, "answer": {"text": "Yes. And they\'re very good.", "date": "7 years ago", "language": "en"}, "total_answers": 5}, "review_results": {"summaries": ["\\"This place has the best Korean food in the area and the service is amazing.\\"", "\\"We ordered fried dumplings and dim sum as apps and 3 different meats to try.\\"", "\\"It was just 8 barley cooked shrimp on a bed of onion with white rice for $32.\\""], "reviews": [{"rating": 4, "description": "I would like to give five stars but I give four because of the toilets. The food is delicious, one of the best Korean food I have ever had. But the restroom!!!!!! You may have put the urinal and hand washing sink side by side!!!!!! but you should think of a barrier between them. I think this is unhygienic for health.\\nI hope the restaurant owner will fix this soon.\\nThe food and staff are great \\u263a\\ufe0f", "link": "https://www.google.com/maps/reviews/data=!4m8!14m7!1m6!2m5!1sChZDSUhNMG9nS0VNeVc2SW42d3ZEaEZBEAE!2m1!1s0x0:0xcabefc70c316486c!3m1!1s2@1:CIHM0ogKEMyW6In6wvDhFA%7C%7C?hl=en-US", "date": "3 months ago", "images": ["https://lh3.googleusercontent.com/geougc-cs/AB3l90AV0NTvUyaC7vjgHXwd6jiHrp8MGYAYNACkAmlzPTbHi-5lsYhVs-vsff9WTMTj3P46aaTos-dG_BUHbhX38zpzIQAZtAiIFHuDYeFHkToWlaiBgZRpMPqyfyOx4zLJGu_2gAGg9Q"]}, {"rating": 1, "description": "Absolutely Unacceptable Experience \\u2013 Vegetarian Given Beef\\n\\nI am beyond disappointed and frankly appalled by this restaurant. I placed an order clearly marked as vegetarian and requested NO BEEF. Despite this, what arrived was unmistakably beef. This is not just a mistake\\u2014it\\u2019s a complete disregard for dietary restrictions, personal beliefs, and basic attention to orders.\\n\\nWhether the issue was carelessness in the kitchen or a failure in communication, there is no excuse for serving meat to someone who avoids it for ethical, religious, or health reasons. This was not just poor service\\u2014it was deeply offensive and unacceptable.\\n\\nI tried reaching out for an explanation or apology, and the response was indifferent at best. If a restaurant can\\u2019t respect something as fundamental as dietary restrictions, they don\\u2019t deserve anyone\\u2019s trust.\\n\\nIf you\'re vegetarian, vegan, or have any dietary restriction at all, avoid this place. They clearly don\\u2019t care.", "link": "https://www.google.com/maps/reviews/data=!4m8!14m7!1m6!2m5!1sCi9DQUlRQUNvZENodHljRjlvT2pFMWMwcENRVVJYYVVWaVpGWlJkbTFsYTJOa2VrRRAB!2m1!1s0x0:0xcabefc70c316486c!3m1!1s2@1:CAIQACodChtycF9oOjE1c0pCQURXaUViZFZRdm1la2NkekE%7C%7C?hl=en-US", "date": "2 months ago", "images": ["https://lh3.googleusercontent.com/geougc-cs/AB3l90AkoleAdKuhT1WkPDSBs1297JUutSkIj-Q5jOBHWHR0k1jmw0hVE_sOILM9616CEN81UUYCCLyc2z0edjTUnBWftY9_jW5zIAdckFroJVoc6VHs8PpWTp_HkwMZKm1UbU21AG3u4RdAojf7", "https://lh3.googleusercontent.com/geougc-cs/AB3l90BYWbqE-ariWNXv68TcW2H2Ub1AeZ2_B8NOfakhozXtjhToJRxp6znwmKy5UtKm6waulfIemqHe9Y1wUqjJqlEzYXhC_dM7V30xiA0Dx3JUmt4yFyIQReClqpEzuyq0x7hRh1ywPQX7nR5k"]}, {"rating": 5, "description": "A lovely korean restaurant with amazing bbq. If you haven\'t tried korean bbq definitely give it a try here! Super friendly staff. I had some amazing haemul bokkeum, which was incredible. Their rice cakes were phenomenal. The fried chicken was also yummy. Lots of good side dishes as well.", "link": "https://www.google.com/maps/reviews/data=!4m8!14m7!1m6!2m5!1sChdDSUhNMG9nS0VJQ0FnSUNiaTg2SGtnRRAB!2m1!1s0x0:0xcabefc70c316486c!3m1!1s2@1:CIHM0ogKEICAgICbi86HkgE%7C%7C?hl=en-US", "date": "a year ago", "images": ["https://lh3.googleusercontent.com/geougc-cs/AB3l90DwyXjvUT4RQTfTODP48q97h9DynP3FKJ-8AFkEi-tXKO2CuSHSuzEuLpyALgPnNi8F7iqHKQCOUzKi_ftBJxG02VK57PNPFgJgx0Li7asB_4RIM_WJYEezSyCK54BPI7E7c7pk", "https://lh3.googleusercontent.com/geougc-cs/AB3l90DK1aSQXzwtCwYytAHfSWCI7LmbysuS6VtYNu_TfKKR7tf0bB1Xk3Cx7ajmFpf1Y5i6Aqa8kAA9-ZVhND_5VhjdBiWOG5lw50Ownub-zun4ouI1T64M3wfs6Zr9axZHCGIyoTDX2g", "https://lh3.googleusercontent.com/geougc-cs/AB3l90CYjT9am3GmmzOMEcPRxZTeuxAUj1_9L3MkS9_jzbmaqVzG--FkCoMFl_qIySlueBTYGyZjxO_0nyh8Jb5wSbIk9Lpmg3x1yUEuIdVd-eYJdVbGMHNgyAjahdjJmKtYIA1esMo", "https://lh3.googleusercontent.com/geougc-cs/AB3l90Cb8XEXm6o7FU2uTwODNLfSo4RRiO4F7qOIOzbzIp8qvycACUWtfXiiQgDKbkt00dRvvihCnC7BwuV7dB_Sw36IvCJMHbnU9BlhLXlRqNqyF1yWF1LTe2q7wAOKwvEtHBq0KnbwaA", "https://lh3.googleusercontent.com/geougc-cs/AB3l90BCaahazqZ6qdRZyROvoN1JnnbLilCBJCkrSp_XgP-Zk6k51GuJdvvq2GAeaT7Ks4k5wPruxSZRtjU6YA3DNinbK7TGfiCIaLI0DBbU0kotXsV6ejGT1oaKlMUgh4Ek0-gpPlzK", "https://lh3.googleusercontent.com/geougc-cs/AB3l90CQDr1LpiNl838-dOFr_7n1ywVTapBA5JZJiKFyT3XXFGD1rQgNmYEJQ9rnuya9PJkE0TYkxAbg4lvbMggYZhlACBoBZm9JLZisEGf3BLs-QEU6bN9oP88ZZS1aPmxr75jDy_9m", "https://lh3.googleusercontent.com/geougc-cs/AB3l90CkATYGOGxD3SAFX4iq5OtPPlnXJYgHwOFT0ON5KJBz-lmz1cCyremh2dbn9scRp3Gr6oWoYfuKjBg6f_G04aqUBrxZX0hZeQcUUBk8nuJ8qj7kN9c7XNhJXRV444y766R81Lo", "https://lh3.googleusercontent.com/geougc-cs/AB3l90AIbSXuEtruXeQaSiMVphUFk52o4om0lFrPibb-XgqoQy2W1Tx85y7tkZKnFkJ7P8XcyddwSeOyiETqgSS3QCjIMaSxjeg52rv9isvftrVhqvhVHsHkRD6MiybcWlp1nQMxKQmfNQ", "https://lh3.googleusercontent.com/geougc-cs/AB3l90Di7vMXgVO0jnSS3GyqJgQz_ACD-LldzoyOl9hxFJN1hyNZaaYmfFoskjX29VBMsStrbNKe65dlb46WAHZsiz2HXnasYSlALQvX5ZR_Ret0-2SvXoBvQyKXK1tFK5LKfQS2g2o", "https://lh3.googleusercontent.com/geougc-cs/AB3l90BvWfF79hCGeWzSBJAbqQCQJ_arNuIQ9NBMgVLIZEvsqVqspg4j5WvtnLpYCplYELprPzoo2KhTJ8CxVfv4l7IMHahb5MvN--nISLNt0SOh4VCX51TkuYqleldv6GJvoPhedHLWpg", "https://lh3.googleusercontent.com/geougc-cs/AB3l90AIkXhdUtmALn9JH6-y5bfJJuWHG9bn3v2SKW-Z-KxXgjHq8p3avX_D01A7qQahvXf90xLOYwlov3qs8dFnRzSSDhEvz8XUF6Hv79jscWwysZDWM6ontm8r2c6ECCX1WhKAwds3", "https://lh3.googleusercontent.com/geougc-cs/AB3l90ABcRmASOKyk-1FY4pz8JzmLL2cNtyICOapOl5HfDnUoeTzZTA-4JfcGNTkr05NtGsAh7KZQO3T5FC4nlSaPDiXoJoBy74k48BNAr9Hb8TSfkGOlQUSIk-Iy_77uflzNT8dDa-x", "https://lh3.googleusercontent.com/geougc-cs/AB3l90DWYXpYoBP-XE5MAaulA6-Uunyq4UIfrMNp-BLxBWXmxjJ3HXq4JWWf5qqZk3crQS9KGkQSyuy8uqKKqJd7Rhl6_FZ1FyTrplRakbH6iacKWqbx8zAQGLDvgPnQLIZ20MwIQsce", "https://lh3.googleusercontent.com/geougc-cs/AB3l90BwDPFWh_0-L-lVpS2CE9DR1h178KX85b_5YImsTtmJllPtWU_AXr9wD-IveRJ8GcV53ss8M0e63ni8XK5ct-eEhfVhaLE8FbkXy5juHxwMchic9NFRY8P5cU8W8bRQV57j1NmU", "https://lh3.googleusercontent.com/geougc-cs/AB3l90CwF5nq-KfCKiGMaClLKokT0bQC0w9Zru8MciFQCn_JUezC7ha2rKzsAAkrUyDtDO6dtKIposXE8oHxozZFwIB3_jbm76y463y4LqwZByGfcURUtPsmr4pouhEJD51pgYNW4pQa", "https://lh3.googleusercontent.com/geougc-cs/AB3l90BG_z4ds2X-o9H6cdonrdBP3brYOlbACFJNpBSfK-MGh87CWecABSpOLkbU5EGqghWVtjxiF_2hDRmb4ohQEj9ztj0pBSe6fNGixmx_ezlBvZ8uei_iboWBC7PdsjGmbjOxBkX1qQ"]}, {"rating": 4, "description": "My first time doing Korean bbq. The restaurant is clean, large and nicely decorated. The experience was fun. The server was polite, he brought us to a table immediately, brought menus, took order and food arrived quickly. He forgot the rice so we went for it ourselves. We waited a little to get the bill. It wasn\\u2019t that the service was slow (NY standards), it was they weren\\u2019t rushing us. Most importantly the food we cooked ourselves was well seasoned, flavorful and delicious. Meat was thin enough it didn\\u2019t take long to cook properly. I don\\u2019t know if I\\u2019d do it again but I\\u2019d probably eat there again and check out cooked food options. Both my kids ate it and felt full. It was a little pricy but serving size was appropriate. Three of us shared two dishes. Bathroom was clean too.", "link": "https://www.google.com/maps/reviews/data=!4m8!14m7!1m6!2m5!1sChdDSUhNMG9nS0VJQ0FnSUNMZ3JpS3BRRRAB!2m1!1s0x0:0xcabefc70c316486c!3m1!1s2@1:CIHM0ogKEICAgICLgriKpQE%7C%7C?hl=en-US", "date": "a year ago", "images": ["https://lh3.googleusercontent.com/geougc-cs/AB3l90DTPy3q0NjCp99YiG9JvzBl4hM8IdLDetQqz4iMWXTxEKGO6cCYZ8aL6xFUPeXZtLfuYLmbrxX40ehCfKAmnmND8eFUsuDb07JDtIzOfjsCh3gCdVsneJbwd_z1SSgaig89oqDL", "https://lh3.googleusercontent.com/geougc-cs/AB3l90Al8Zz8pADHXCmxaDq8iiq4EVNeu-ICB0V8FyLHAcUlmG0k2YMWHhLIEVPwLap61fWJFXwQi4wYUed7cjRSlwO7eW6heQX8HHLYBKl2jjXnF0YmsrpY2hTNiQ93Fwl11zqPUR58Mw", "https://lh3.googleusercontent.com/geougc-cs/AB3l90D33F9h2FtJVCunFQSGFSJ07EtG2vBOZMFOVIJgex0Yx8pwR0gllP82P3-ineDcdquy0DKjlLgzQHZJCVIhBkA0_ZsJPBboYYbAJaIbcudoD-Oh79F0nZvzvLisEkZf4724aUlq", "https://lh3.googleusercontent.com/geougc-cs/AB3l90DLZJM4-i5sRKiXqkcmoiaNJ2YaYlAq-XR-KtgCB29zwPe9rAKE24yOJoW8gd1I1CyKQGbQLjepgB84oe7aAx5gGnPVMIiFqsmTbovr39hhB28lr1WSV1gVBO8aywT2x9tuMEW8rw", "https://lh3.googleusercontent.com/geougc-cs/AB3l90Bfmh5QHGLaervA4vx8_mdK41rPHwHWIyL4iBOKtYi8OBMbx1sq0Cm8qHiEDLNszD2j9F26jngUO0geo1eiGZChZ36SPh66ckZP8L3fsGye1OmGyZAJ-pemvPHZ6B55yW-rADXK", "https://lh3.googleusercontent.com/geougc-cs/AB3l90DClD8_20pTAVkNoAF0vm2dksv6umE2XCLx-FE19vfd6C9cCF7l0mHlkblAQBpg2eQWt_ImpxoOu_xemnZIbLeJ4i1-DWiXVAABdLIxrKLESGnLAn_oRL5c9-GBC0iqJsPVr1LBbg", "https://lh3.googleusercontent.com/geougc-cs/AB3l90DSsmDI129MeQcI_zanZCF9QDaMcHud8-kzCIo8bITleYzKUq3AbGCGlvlLzVgGrY93RZ20swXVzTVlv9zS36t7K56uc9t1scXC-YxSikqsyHF2wY6xMRIrK6LVQ_hUX7CjPRs", "https://lh3.googleusercontent.com/geougc-cs/AB3l90Dy7qYZko86IOq0DmBEd5GRlRcEsUtFVOPlPkJ8YF2WnuJIirRVsq3bOBqbCncN-oVS5ih5i-Q2xataJrs8yk0K0HDOfNmrTwnUFS9obJHQ_FL_zRL8lukOdRgoqL9yW_H4Hnci2A", "https://lh3.googleusercontent.com/geougc-cs/AB3l90DUgE9_g_w31qhi5pKbtpxzGPFv0NZcJqizQ_NUYhg0TYyaIGXFH1yV0KcoFzji_VZ5GxopGdyp0TwZUicD5kTC4l97bqLxBvsv3nNyM_I7uYMVqtXv2ioKltp32Tk0P_Mk15c", "https://lh3.googleusercontent.com/geougc-cs/AB3l90DsePG6Kd1dRBJFN0fXBNx_RwqYrFAZEj-RptGqEz8jSyztIZkRsEKoM6TbiM2rnTdLFPWubhXSu1NhN-6sZ2KxE0Yw6_KPcgNIPAq3c_yUniUGqdTeiLU0XvfIpYtVHKHXSta0-g", "https://lh3.googleusercontent.com/geougc-cs/AB3l90CLDLD2P95mzhXc6jKXEtu1dSDMGA-IH-uq773Rvo3CRJaZXPYRBNwzFfYKJaIPfcE8XppD2oZgPkmpdGCsHA83Coqgmu9OmBAf2zwxcocoyxueRXKFe2PVN5pZGIlbyS-GJ7WU", "https://lh3.googleusercontent.com/geougc-cs/AB3l90DmoP3MIY7NAzAmimVPCQIrrSCUqTv_NKkNiN-rbLjEhaTQA0RT99OGM3I4uu--3f9BSFfVGeIRgYRb_nCpqORc6q7qUgzTBp66h40T_HIzv-PbljZABGkdgJzC_U02ZaU7GsUjrA", "https://lh3.googleusercontent.com/geougc-cs/AB3l90DZTKeHCBFr8X3dP5IpKOp3S_oXp8avPe44GE-FKdpYwAqcwneokx7NoFNtlXIW42JXrOQGFs4m3Iyola_7X7sBPmhYTIGfDLzS6cwEIbFnYAAZmHp8HY6RKOw8ZqTTPmEQmeLC", "https://lh3.googleusercontent.com/geougc-cs/AB3l90AB7Zuc6MIWXkagKH6dbTuIW43_dXCZBIwLYsoVYUJGdPhoDlPpLpB6mM2hWl3M0_ef_rXkV3QVO5jcVdvwPWmhi30V-dH8CRVaHOhHjfIqcGKuDPCytaYWgxQS5tkmmW0rrxSo", "https://lh3.googleusercontent.com/geougc-cs/AB3l90CqdZ3IbpMriyyt9e3Ku7mYNKXaJ_etrQHrpGIaw1HRlFpx8dwj0EkICyzk3JXQg4U9uq4WM7nXe1kX_UlmukShoT_5M-X0TmLKIoM4dLglqONLnR51jUd_uFRCOpKQW5w31KF3", "https://lh3.googleusercontent.com/geougc-cs/AB3l90C5O57QfxR8epshEVdgH_xKzwyEvmZeS31goyWElPInAW3pNc4ItNB1SwwDxJqRkoWYKZXiYC2HU3eoq7DQOLQMknza_4tEJ9Cn-kONzxpvmzsgysDU8oeFKsTRJzdZx8pkPqHpSA", "https://lh3.googleusercontent.com/geougc-cs/AB3l90BXQ_NzGm7ZGPkOuzh1xo3f3qGN8ET8TgqB6XTtpCYCAfZVX9sHRFixMBIhhJ6xAIVJJ9uRqSWE1-9uxJ_8krBqSPE05qQOE63ZZKsoA2SP5RwrVCXX5KsjSoBCMEfNphE0DPTW", "https://lh3.googleusercontent.com/geougc-cs/AB3l90Dkne9hfDTPXj7y3XFkFVbNPq6LKdV3AN3LtpncgHo3MQ0MDR34-SNyvyxZdcopDzLmid7CABRO6qBRo_Q4tSKDta_dH-k3KTQTn32x8xopotAGAQndM0o4vdaeU0YQXQrc1KKPSQ", "https://lh3.googleusercontent.com/geougc-cs/AB3l90C44-Cxs-nSoQXPUlh1hpKfovmol5MIiAyVzt7oNToKo2FDElHucb3aafB0U2SpPUimL01fvcdC2vaHKMjWZg9oW0_MeQMphUMtSMta3qdjJfSY867EGlNrajq4k-LTmdD1RqDU"]}, {"rating": 2, "description": "The food at Seoul Garden is truly wonderful, but the customer service ruins the experience. During lunch hour, it is a disaster. They are always understaffed, usually leaving one waitress to handle dine-in tables and to-go orders at the same time. If you have a lunch break, do not expect to get back to work on time.\\n\\nOnce your food arrives, do not expect to see your server again. Refills are rare, and if you need anything\\u2014like your check or to-go boxes\\u2014you will have to hunt someone down. It feels like an afterthought rather than a service.\\n\\nYes, the food tastes great, but you can absolutely go elsewhere and get both exceptional customer service and amazing food without the frustration.", "link": "https://www.google.com/maps/reviews/data=!4m8!14m7!1m6!2m5!1sCi9DQUlRQUNvZENodHljRjlvT2tsNmJTMVhWM05ITkd4Q1dXSnVWRE5WTlVzeVdHYxAB!2m1!1s0x0:0xcabefc70c316486c!3m1!1s2@1:CAIQACodChtycF9oOkl6bS1XV3NHNGxCWWJuVDNVNUsyWGc%7C%7C?hl=en-US", "date": "3 weeks ago"}, {"rating": 3, "description": "Pretty good Korean food in Raleigh, NC.  We were there around 2pm on a weekday.  The food, in my humble opinion, was a bit bland, but it was still good.  The service wasn\'t horrible, but the server barely spoke to us and the guy at the register, who may have also been the cook, literally grunted at me when I asked how he was doing and spoke no other words.  It was weird, we kinda felt like we were imposing on them or something for being there.  Maybe they were tired from a recent lunch rush or something.  The atmosphere was cool enough though.  I would definitely try it if you\'re in the area, but I wouldn\'t go out of your way to get there.  It\'s a good, but fairly forgettable Korean dining experience.", "link": "https://www.google.com/maps/reviews/data=!4m8!14m7!1m6!2m5!1sChdDSUhNMG9nS0VJQ0FnSUNack15ZXBBRRAB!2m1!1s0x0:0xcabefc70c316486c!3m1!1s2@1:CIHM0ogKEICAgICZrMyepAE%7C%7C?hl=en-US", "date": "2 years ago", "images": ["https://lh3.googleusercontent.com/geougc-cs/AB3l90CqJYSKE_LCwCvILGYMA12SzsXHJZ9hCZ9Tf21Oj6-1Vo400lSNx4-MoknsBI_kY2s_TPWXW0sYtZDdbwpl7mtP0yvHGEnAYYrsaCOLGvEtqlQCKXfyBnWfL93Tni189Ts8AHNn", "https://lh3.googleusercontent.com/geougc-cs/AB3l90DjObZmc9vPJZqpVHD7TE_9_QXSHDsatQ6MzMm51BkTh59t-_Pq9iY-CsEYEl__28XEXqksKVuBX-rLOYpltL6SO1C3KQJTwWnrHsLtcLEiz-JbUFnv0KSxKMX71tFJ-4WQlV7X8g", "https://lh3.googleusercontent.com/geougc-cs/AB3l90AA_Jm3FPXfafjTNpnd16S8-Xk6RrGvHg-dLuYTe79fP_vBXkF0t6ElP0N-NqAOaVlF5nxfmhsS3g1s99pHqFBHQPYwX1lrruYOWlPibi4I1B_2A7DehitegwbAf61IvCFCUXY", "https://lh3.googleusercontent.com/geougc-cs/AB3l90DuhZxRzbK7BEWSiYiZ3bqtsa6R-0EjTOQMVU1ZRj3WfsJAuF4sVVu6uxzNPzmMIluPzhhpazURO2j9ZqMh4Y0_EBhIVg_Wz8kCWgKm1qBXOB6g0eVQNsIz9482_fCHzmYsYld8Hw"]}, {"rating": 5, "description": "Everything was great!!\\n\\nWe came for dinner there was no line the vibe was nice and it smelled great!\\nWe didn\'t sit at a bbq table so the bbq was prepared for us.\\n\\n\\ud83e\\udd5fFried dumplings: nice size and great filling, the sauce goes nicely with it as well.\\n\\ud83e\\udd69Bulgogi: amazing just amazing!!!\\n\\ud83c\\udf57Fried chicken: a tad too much pepper powder for my preference, it overtakes the chicken flavor a bit but still good.\\n\\nOur server was very nice and friendly, he checked up on us here and there to make sure we\'re doing alright. We were seated quickly at dinner time and the food was served fast.\\n\\nOverall, nothing to complain about and we will definitely return to try their tteokbokki and kalbijjim and bibimbap and japchae and soondubu and many more!!!", "link": "https://www.google.com/maps/reviews/data=!4m8!14m7!1m6!2m5!1sChZDSUhNMG9nS0VJQ0FnSUQxMFpfbFJREAE!2m1!1s0x0:0xcabefc70c316486c!3m1!1s2@1:CIHM0ogKEICAgID10Z_lRQ%7C%7C?hl=en-US", "date": "a year ago", "images": ["https://lh3.googleusercontent.com/geougc-cs/AB3l90Corlze-O1TX6j2AuNYidOv6VT8wCRKYSZ2nQxSUDTqGIF8O2e5ZgcpYfEqOOji3hlOK-o65rag4uY-k4y070Tr6k0QCdJY5_HhymV_6ozV1mlMJMyt6H6qUTxjDOe5AlIUCnTh", "https://lh3.googleusercontent.com/geougc-cs/AB3l90BMV2PpRY7V3Yt1vkR5b-51xby3o_jcX7bbGQCsJWl2SHxu6on0MLKMAI7N6sL8UHhhhC0P0wZRPvXUFncPFSt9STAnRv87EDDPML5FBGiz0eHUDDuP6f-oKzpO6adWCfB4K8R-", "https://lh3.googleusercontent.com/geougc-cs/AB3l90Av6lvAI8vSB1zCK8cr3h8EvKO3xsyffaeuoUc7lFTSXpGVXL8qoupJ7jRwYJyG56CxPGnRZCHDoL2Ev4HMPQeGPDbcA-hNuv0Qq_I2d35a_eK0puUHv4TqNvh4nm1Mnq7ew0miKw", "https://lh3.googleusercontent.com/geougc-cs/AB3l90AjuR2GJR_18Ew6eTwhwyfCFnj2XavClF_2p_e5YfMglGV--cRaPQPbyqKgvYNGLmde45S1bFMNdLEwf1UXaYCwz-NysuTcOEQgceyxleWsg7nJkBIFs9XD44H3i5VFJwd9n5F0", "https://lh3.googleusercontent.com/geougc-cs/AB3l90DuQ4pFhwPZvBosRU8YTKGip3uJiYc_1WrRJxXfuzSj0dkqicuFJpSLnLlpV9XW1DwOeud2JUpkVLpEnXt0VmuUPdGjEd32qGWWUdQoxGS9oFIRpZm0gT7KxcKd3AGtn8zwfHrwcQ"]}, {"rating": 5, "description": "Oh my goodness!!! My server was super kind and helpful as I made my selections! The setting is sooo relaxed and mellow. I just loved the beautifully arranged appetizers that arrived. They were an amazing precursor to my fantastic meal. The food here exceeded any expectation I could think of!", "link": "https://www.google.com/maps/reviews/data=!4m8!14m7!1m6!2m5!1sChZDSUhNMG9nS0VJQ0FnSUN4NGFET0hBEAE!2m1!1s0x0:0xcabefc70c316486c!3m1!1s2@1:CIHM0ogKEICAgICx4aDOHA%7C%7C?hl=en-US", "date": "Edited 2 years ago", "images": ["https://lh3.googleusercontent.com/geougc-cs/AB3l90DkrapVjM8S4FNDGPRS9a7VH8mK0vNMfSn6wTBsjo79QJY6b59GSa8ZKbtxH9VvZ_pCJ-cZMLcEMzTllnm0oh9Nrh7aKypqBCC9l5nb2ZYa63lBr_5ObWTi8nXy2tOq8px1Ra1p", "https://lh3.googleusercontent.com/geougc-cs/AB3l90CPcUs08hGOCMqbXALnYPByNpMJze_g-pJo4jXEq1QOEmQo2TMc4dsrlzy1GskwRA7KoJFwt6QTqrKCWzDxPfo5x-G2m1g7Cv3m3RyIWriL4DJ_dcKuXaBBHye_RzIrsns6ZyAX", "https://lh3.googleusercontent.com/geougc-cs/AB3l90Ayw6GlE32kyDnZlxp-bXOyrYG23jfxvM2HQBIDUrxwO6YRrrzgo8kMQqCLGfBBXDu0N6gdA5SZ-tGjQOhIw8Lr9_HhjCqKmmjCT60nveHaDbXdkFATcgkBG8_5isISnsPD3hqn", "https://lh3.googleusercontent.com/geougc-cs/AB3l90AI_JW8eSMsSLiEsteaykKbphtG1ujuIOKmLJLfhKAXuB09HpV4POX-DFm-ZOyMObPv4_l05eg1biiLbk2DDQb62ZwZkTiVjrBO1kRElloHInXjwe9K9aDJ4-UIv7RQSsIozz7V", "https://lh3.googleusercontent.com/geougc-cs/AB3l90CGfyywrIdm2niysAhL-Vig-l8NnrQpvS2LTJiVpHW5fwKtVvLKKts1T8UA4EaNCUT5gA50FCb3xLx4vctT0Zj2SKHAjykE5zyQHn5TRvraviwwc1fTvlrJ0WcXDoZmYG01wAYhGw"]}]}, "popular_times": {"live": {"typical_time_spent": "People typically spend up to 1.5 hours here"}, "chart": {"sunday": [{"time": "6\\u202fAM", "busyness_score": 0}, {"time": "7\\u202fAM", "busyness_score": 0}, {"time": "8\\u202fAM", "busyness_score": 0}, {"time": "9\\u202fAM", "busyness_score": 0}, {"time": "10\\u202fAM", "busyness_score": 0}, {"time": "11\\u202fAM", "busyness_score": 0}, {"time": "12\\u202fPM", "busyness_score": 58, "info": "Usually a little busy"}, {"time": "1\\u202fPM", "busyness_score": 76, "info": "Usually not too busy"}, {"time": "2\\u202fPM", "busyness_score": 72, "info": "Usually not too busy"}, {"time": "3\\u202fPM", "busyness_score": 73, "info": "Usually not too busy"}, {"time": "4\\u202fPM", "busyness_score": 81, "info": "Usually as busy as it gets"}, {"time": "5\\u202fPM", "busyness_score": 99, "info": "Usually as busy as it gets"}, {"time": "6\\u202fPM", "busyness_score": 100, "info": "Usually as busy as it gets"}, {"time": "7\\u202fPM", "busyness_score": 85, "info": "Usually as busy as it gets"}, {"time": "8\\u202fPM", "busyness_score": 65, "info": "Usually not too busy"}, {"time": "9\\u202fPM", "busyness_score": 40, "info": "Usually not too busy"}, {"time": "10\\u202fPM", "busyness_score": 0}, {"time": "11\\u202fPM", "busyness_score": 0}], "monday": [{"time": "6\\u202fAM", "busyness_score": 0}, {"time": "7\\u202fAM", "busyness_score": 0}, {"time": "8\\u202fAM", "busyness_score": 0}, {"time": "9\\u202fAM", "busyness_score": 0}, {"time": "10\\u202fAM", "busyness_score": 0}, {"time": "11\\u202fAM", "busyness_score": 32, "info": "Usually not too busy"}, {"time": "12\\u202fPM", "busyness_score": 50, "info": "Usually a little busy"}, {"time": "1\\u202fPM", "busyness_score": 49, "info": "Usually not too busy"}, {"time": "2\\u202fPM", "busyness_score": 45, "info": "Usually not too busy"}, {"time": "3\\u202fPM", "busyness_score": 35, "info": "Usually not too busy"}, {"time": "4\\u202fPM", "busyness_score": 39, "info": "Usually not too busy"}, {"time": "5\\u202fPM", "busyness_score": 53, "info": "Usually a little busy"}, {"time": "6\\u202fPM", "busyness_score": 67, "info": "Usually a little busy"}, {"time": "7\\u202fPM", "busyness_score": 65, "info": "Usually a little busy"}, {"time": "8\\u202fPM", "busyness_score": 45, "info": "Usually not too busy"}, {"time": "9\\u202fPM", "busyness_score": 26, "info": "Usually not too busy"}, {"time": "10\\u202fPM", "busyness_score": 0}, {"time": "11\\u202fPM", "busyness_score": 0}], "wednesday": [{"time": "6\\u202fAM", "busyness_score": 0}, {"time": "7\\u202fAM", "busyness_score": 0}, {"time": "8\\u202fAM", "busyness_score": 0}, {"time": "9\\u202fAM", "busyness_score": 0}, {"time": "10\\u202fAM", "busyness_score": 0}, {"time": "11\\u202fAM", "busyness_score": 56, "info": "Usually a little busy"}, {"time": "12\\u202fPM", "busyness_score": 68, "info": "Usually a little busy"}, {"time": "1\\u202fPM", "busyness_score": 59, "info": "Usually a little busy"}, {"time": "2\\u202fPM", "busyness_score": 49, "info": "Usually not too busy"}, {"time": "3\\u202fPM", "busyness_score": 41, "info": "Usually not too busy"}, {"time": "4\\u202fPM", "busyness_score": 44, "info": "Usually not too busy"}, {"time": "5\\u202fPM", "busyness_score": 50, "info": "Usually a little busy"}, {"time": "6\\u202fPM", "busyness_score": 57, "info": "Usually a little busy"}, {"time": "7\\u202fPM", "busyness_score": 54, "info": "Usually a little busy"}, {"time": "8\\u202fPM", "busyness_score": 41, "info": "Usually not too busy"}, {"time": "9\\u202fPM", "busyness_score": 27, "info": "Usually not too busy"}, {"time": "10\\u202fPM", "busyness_score": 0}, {"time": "11\\u202fPM", "busyness_score": 0}], "thursday": [{"time": "6\\u202fAM", "busyness_score": 0}, {"time": "7\\u202fAM", "busyness_score": 0}, {"time": "8\\u202fAM", "busyness_score": 0}, {"time": "9\\u202fAM", "busyness_score": 0}, {"time": "10\\u202fAM", "busyness_score": 0}, {"time": "11\\u202fAM", "busyness_score": 39, "info": "Usually not too busy"}, {"time": "12\\u202fPM", "busyness_score": 50, "info": "Usually a little busy"}, {"time": "1\\u202fPM", "busyness_score": 50, "info": "Usually not too busy"}, {"time": "2\\u202fPM", "busyness_score": 45, "info": "Usually not too busy"}, {"time": "3\\u202fPM", "busyness_score": 39, "info": "Usually not too busy"}, {"time": "4\\u202fPM", "busyness_score": 39, "info": "Usually not too busy"}, {"time": "5\\u202fPM", "busyness_score": 48, "info": "Usually not too busy"}, {"time": "6\\u202fPM", "busyness_score": 52, "info": "Usually a little busy"}, {"time": "7\\u202fPM", "busyness_score": 60, "info": "Usually a little busy"}, {"time": "8\\u202fPM", "busyness_score": 53, "info": "Usually a little busy"}, {"time": "9\\u202fPM", "busyness_score": 37, "info": "Usually not too busy"}, {"time": "10\\u202fPM", "busyness_score": 0}, {"time": "11\\u202fPM", "busyness_score": 0}], "friday": [{"time": "6\\u202fAM", "busyness_score": 0}, {"time": "7\\u202fAM", "busyness_score": 0}, {"time": "8\\u202fAM", "busyness_score": 0}, {"time": "9\\u202fAM", "busyness_score": 0}, {"time": "10\\u202fAM", "busyness_score": 0}, {"time": "11\\u202fAM", "busyness_score": 54, "info": "Usually a little busy"}, {"time": "12\\u202fPM", "busyness_score": 71, "info": "Usually a little busy"}, {"time": "1\\u202fPM", "busyness_score": 73, "info": "Usually a little busy"}, {"time": "2\\u202fPM", "busyness_score": 62, "info": "Usually a little busy"}, {"time": "3\\u202fPM", "busyness_score": 53, "info": "Usually a little busy"}, {"time": "4\\u202fPM", "busyness_score": 53, "info": "Usually a little busy"}, {"time": "5\\u202fPM", "busyness_score": 63, "info": "Usually a little busy"}, {"time": "6\\u202fPM", "busyness_score": 72, "info": "Usually a little busy"}, {"time": "7\\u202fPM", "busyness_score": 67, "info": "Usually a little busy"}, {"time": "8\\u202fPM", "busyness_score": 53, "info": "Usually a little busy"}, {"time": "9\\u202fPM", "busyness_score": 35, "info": "Usually not too busy"}, {"time": "10\\u202fPM", "busyness_score": 0}, {"time": "11\\u202fPM", "busyness_score": 0}], "saturday": [{"time": "6\\u202fAM", "busyness_score": 0}, {"time": "7\\u202fAM", "busyness_score": 0}, {"time": "8\\u202fAM", "busyness_score": 0}, {"time": "9\\u202fAM", "busyness_score": 0}, {"time": "10\\u202fAM", "busyness_score": 0}, {"time": "11\\u202fAM", "busyness_score": 31, "info": "Usually not too busy"}, {"time": "12\\u202fPM", "busyness_score": 43, "info": "Usually not too busy"}, {"time": "1\\u202fPM", "busyness_score": 53, "info": "Usually a little busy"}, {"time": "2\\u202fPM", "busyness_score": 54, "info": "Usually a little busy"}, {"time": "3\\u202fPM", "busyness_score": 51, "info": "Usually a little busy"}, {"time": "4\\u202fPM", "busyness_score": 54, "info": "Usually a little busy"}, {"time": "5\\u202fPM", "busyness_score": 62, "info": "Usually a little busy"}, {"time": "6\\u202fPM", "busyness_score": 68, "info": "Usually a little busy"}, {"time": "7\\u202fPM", "busyness_score": 77, "info": "Usually a little busy"}, {"time": "8\\u202fPM", "busyness_score": 70, "info": "Usually a little busy"}, {"time": "9\\u202fPM", "busyness_score": 53, "info": "Usually a little busy"}, {"time": "10\\u202fPM", "busyness_score": 0}, {"time": "11\\u202fPM", "busyness_score": 0}]}}}]},
   "searchapi_data_google": "",
   "user": "testing"}],
 "status": "Success"
 }
```
</details>

---

# **4. End‑to‑End Data Flow**

```
Customer Name + Address
           ↓
     SearchAPI Query
           ↓
   Raw SearchAPI Output
           ↓
 Segmentation API Cleanup
           ↓
     OpenAI Classification
           ↓
 Final Segmentation Output
```

---

