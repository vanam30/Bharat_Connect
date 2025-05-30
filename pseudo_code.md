# 1. AI Prompt Configuration

AI_PROMPT = """Analyze this travel query and extract key details. Follow these steps:
1. Identify parameters:
- Origin city (required)
- Destination city (required)
- Travel date (required)
- Budget preference (optional: "cheapest", "balanced", "richie rich")
- Time preference (optional: "fastest", "flexible")

2. Hub detection logic:
- If origin is Tier 2/3 city without airport:
  - Find nearest Tier 1 city with airport using geographical knowledge
  - Example: Surat → Mumbai, Coimbatore → Chennai
- If origin has airport, use origin directly

3. Output JSON format:
{{
  "origin": "City Name",
  "destination": "City Name",
  "date": "YYYY-MM-DD",
  "preference": {{"type": "cost/time", "priority": "low/medium/high"}},
  "nearest_hub": {{"city": "Hub Name", "required": true/false}},
  "route_types": ["direct_train", "direct_flight", "multi_modal"] 
}}

4. Special cases:
- Handle relative dates: "tomorrow, next few days" → actual date
- Infer implicit preferences: "cheapest way" → cost priority high

Example queries:
1. "Need to reach Delhi from Indore by 15th morning cheapest option"
2. "Book flight from Nagpur to Bangalore for next Friday"
3. "Surat to Kolkata tomorrow fastest route under 5000"
"""

# 2. Core AI Integration

def ai_process_query(user_query):

    """Send query to AI model and parse structured response"""
    formatted_prompt = AI_PROMPT.format(user_query=user_query)
    
    # AI API call (OpenAI/Perplexity/Gemini)
    response = requests.post(
        "https://api.openai.com/v1/chat/completions",
        headers={"Authorization": f"Bearer {OPENAI_API_KEY}"},
        json={
            "model": "gpt-4", # any model works
            "messages": [{"role": "user", "content": formatted_prompt}]
        }
    )
    
    ai_response = json.loads(response.json()["choices"][0]["message"]["content"])
    
    return ai_response

# 3. API Integration Layer

def get_live_train_schedule(origin, destination, date):

    """Fetch real-time train data from Rail API"""
    
    try:
        response = requests.get(
            "https://railapi.com/v3/search",
            params={
                "origin": origin,
                "dest": destination,
                "date": date,
                "apikey": RAIL_API_KEY
            }
        )
        return [{
            "id": t["number"],
            "name": t["name"],
            "departure": t["departure_time"],
            "arrival": t["arrival_time"],
            "fare": t["fare"]["sleeper"],
            "duration": t["duration_min"],
            "class": t["class_type"],
            "mode": "train"
        } for t in response.json()["trains"]]
    except Exception as e:
        print(f"Rail API Error: {e}")
        return []

def get_live_flight_schedule(origin, destination, date):

    """Fetch real-time flight data from Flight API"""
    try:
        response = requests.get(
            "https://flightapi.com/v2/search",
            headers={"Authorization": f"Bearer {FLIGHT_API_KEY}"},
            params={
                "origin": origin,
                "dest": destination,
                "date": date,
                "cabin_class": "economy"
            }
        )
        return [{
            "id": f["flight_number"],
            "airline": f["airline"],
            "departure": f["departure_time"],
            "arrival": f["arrival_time"],
            "fare": f["price"],
            "duration": f["duration_min"],
            "class": "Economy",
            "mode": "flight"
        } for f in response.json()["flights"]]
    except Exception as e:
        print(f"Flight API Error: {e}")
        return []

# 4. Transportation Data Fetching

def fetch_transport_data(parsed_query):

    """Retrieve transportation options based on AI-identified route types"""
    results = {}
    origin = parsed_query["origin"]
    destination = parsed_query["destination"]
    date = parsed_query["date"]
    route_types = parsed_query["route_types"]
    
    if "direct_train" in route_types:
        results["trains"] = get_live_train_schedule(origin, destination, date)
        
    if "direct_flight" in route_types:
        results["flights"] = get_live_flight_schedule(origin, destination, date)
        
    if "multi_modal" in route_types:
        hub = parsed_query["nearest_hub"]["city"]
        results["hub_trains"] = get_live_train_schedule(origin, hub, date)
        results["hub_flights"] = get_live_flight_schedule(hub, destination, date)
    
    return results

# 5. Route Combination Engine 

def process_direct_routes(data):

    """Process direct train and flight routes"""
    routes = []
    
    # Direct trains
    for train in data.get("trains", []):
        routes.append({
            "type": "direct_train",
            "segments": [train],
            "total_fare": train["fare"],
            "total_time": train["duration"],
            "layover": 0
        })
    
    # Direct flights
    for flight in data.get("flights", []):
        routes.append({
            "type": "direct_flight",
            "segments": [flight],
            "total_fare": flight["fare"],
            "total_time": flight["duration"],
            "layover": 0
        })
    
    return routes

def create_multi_modal_route(train, flight):

    """Create multi-modal route with layover calculation"""
    layover = calculate_layover(
        parse_time(train["arrival_time"]),
        parse_time(flight["departure_time"])
    )
    
    return {
        "type": "multi_modal",
        "segments": [train, flight],
        "total_fare": train["fare"] + flight["fare"],
        "total_time": train["duration"] + flight["duration"] + layover,
        "layover": layover
    }

def validate_connection(train, flight):

    """Enhanced validation with layover tracking"""
    min_layover = 90 
    layover = calculate_layover(
        parse_time(train["arrival_time"]),
        parse_time(flight["departure_time"])
    )
    return layover >= min_layover

def calculate_layover(train_arrival, flight_departure):

    """Calculate layover time in minutes"""
    return int((flight_departure - train_arrival).total_seconds() / 60)

# 6. Enhanced Route Processing

def generate_routes(data, sort_preference='fastest'):

    """Generate and sort routes with layover analysis"""
    routes = []
    min_layover = float('inf')
    best_layover_route = None
    
    # Process direct routes
    routes += process_direct_routes(data)
    
    # Process multi-modal routes
    for train in data.get("hub_trains", []):
        for flight in data.get("hub_flights", []):
            if validate_connection(train, flight):
                route = create_multi_modal_route(train, flight)
                
                # Track minimum layover
                if route["layover"] < min_layover and route["layover"] > 0:
                    min_layover = route["layover"]
                    best_layover_route = route
                
                routes.append(route)
    
    # Initial sorting
    sorted_routes = sort_routes(routes, sort_preference)
    
    # Add layover metadata
    return {
        "routes": sorted_routes,
        "best_layover": best_layover_route,
        "sort_preference": sort_preference
    }


# 7. Dynamic Sorting System

def sort_routes(routes, preference):

    """Sort routes with additional metadata"""
    if preference == 'fastest':
        return sorted(routes, key=lambda x: x["total_time"])
    elif preference == 'cheapest':
        return sorted(routes, key=lambda x: x["total_fare"])
    else:  # optimal approach if priority not defined
        return sorted(routes, key=lambda x: (x['total_fare'] * 0.7) + (x['total_time'] * 0.3))

# 8. Main Workflow

def handle_travel_request(user_query, sort_preference='fastest'):

    """End-to-end processing with dynamic sorting"""
    # Step 1: AI Analysis
    parsed = ai_process_query(user_query)
    
    # Step 2: Data Collection
    transport_data = fetch_transport_data(parsed)
    
    # Step 3: Route Generation
    result = generate_routes(transport_data, sort_preference)
    
    # Step 4: Result Formatting
    return format_results(result, parsed)



