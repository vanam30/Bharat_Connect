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



// API service functions for PayrollPayments

// Import your auth functions
import { getAuth } from './auth'; // Adjust path as needed

const API_BASE_URL = 'http://localhost:3001/api'; // Adjust your backend URL

// API service class for batch operations
class PayrollBatchService {
  
  // Get authentication headers
  static getAuthHeaders() {
    const auth = getAuth();
    return {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${auth.accessToken}`,
      'x-user-id': auth.userId,
      'x-session-token': auth.sessionToken
    };
  }

  // Create a new batch
  static async createBatch(batchData) {
    try {
      const response = await fetch(`${API_BASE_URL}/batches`, {
        method: 'POST',
        headers: this.getAuthHeaders(),
        body: JSON.stringify(batchData)
      });
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      
      return await response.json();
    } catch (error) {
      console.error('Error creating batch:', error);
      throw error;
    }
  }

  // Add transaction to existing batch
  static async addTransactionToBatch(batchId, transactionData) {
    try {
      const auth = getAuth();
      const payload = {
        batchId: batchId,
        creationTime: new Date().toISOString(),
        createdByUserId: parseInt(auth.userId),
        status: "draft", // default status
        transactions: [
          {
            transactionId: transactionData.transactionId,
            transactionDate: transactionData.date,
            debitAccount: transactionData.debitAccount,
            creditAccount: transactionData.creditAccount,
            debitCurrency: transactionData.debitCurrency,
            debitAmount: parseFloat(transactionData.debitAmount),
            creditCurrency: transactionData.creditCurrency,
            employeeName: transactionData.employeeName,
            bankId: transactionData.bankId,
            remarks: transactionData.remarks || "Testing",
            version: 0
          }
        ]
      };

      const response = await fetch(`${API_BASE_URL}/batches/${batchId}/transactions`, {
        method: 'POST',
        headers: this.getAuthHeaders(),
        body: JSON.stringify(payload)
      });
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      
      return await response.json();
    } catch (error) {
      console.error('Error adding transaction to batch:', error);
      throw error;
    }
  }

  // Submit batch (either as draft or for approval)
  static async submitBatch(batchId, status = "draft", transactions = []) {
    try {
      const auth = getAuth();
      const payload = {
        batchId: batchId,
        creationTime: new Date().toISOString(),
        createdByUserId: parseInt(auth.userId),
        status: status, // "draft" or "submitted_for_approval"
        transactions: transactions.map(transaction => ({
          transactionId: transaction.transactionId,
          transactionDate: transaction.date,
          debitAccount: transaction.debitAccount,
          creditAccount: transaction.creditAccount,
          debitCurrency: transaction.debitCurrency,
          debitAmount: parseFloat(transaction.debitAmount),
          creditCurrency: transaction.creditCurrency,
          employeeName: transaction.employeeName,
          bankId: transaction.bankId,
          remarks: transaction.remarks || "Testing",
          version: 0
        }))
      };

      const response = await fetch(`${API_BASE_URL}/batches/submit`, {
        method: 'POST',
        headers: this.getAuthHeaders(),
        body: JSON.stringify(payload)
      });
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      
      return await response.json();
    } catch (error) {
      console.error('Error submitting batch:', error);
      throw error;
    }
  }

  // Get batch details
  static async getBatch(batchId) {
    try {
      const response = await fetch(`${API_BASE_URL}/batches/${batchId}`, {
        method: 'GET',
        headers: this.getAuthHeaders()
      });
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      
      return await response.json();
    } catch (error) {
      console.error('Error fetching batch:', error);
      throw error;
    }
  }

  // Get all batches for user
  static async getUserBatches() {
    try {
      const response = await fetch(`${API_BASE_URL}/batches/user`, {
        method: 'GET',
        headers: this.getAuthHeaders()
      });
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      
      return await response.json();
    } catch (error) {
      console.error('Error fetching user batches:', error);
      throw error;
    }
  }

  // Update batch status
  static async updateBatchStatus(batchId, status) {
    try {
      const response = await fetch(`${API_BASE_URL}/batches/${batchId}/status`, {
        method: 'PUT',
        headers: this.getAuthHeaders(),
        body: JSON.stringify({ status })
      });
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      
      return await response.json();
    } catch (error) {
      console.error('Error updating batch status:', error);
      throw error;
    }
  }
}

// Updated PayrollPayments component with backend integration
import React, { useState, useEffect } from "react";
import { useNavigate } from "react-router-dom";
import {
  Grid, Card, CardContent, Typography, TextField, Button,
  Box, Table, TableBody, TableCell, TableContainer, TableHead,
  TableRow, Tabs, Tab, IconButton, MenuItem, Paper
} from "@mui/material";
import { Add, Edit, Delete, Save, Send, Person, Receipt } from "@mui/icons-material";

export default function PayrollPayments() {
  const [setTab] = useState(0);
  const [batchId, setBatchId] = useState('');
  const [form, setForm] = useState({
    debitAccount: '',
    creditAccount: '',
    debitCurrency: '',
    debitAmount: '',
    creditCurrency: '',
    employeeName: '',
    bankId: '',
    remarks: ''
  });
  const [errors, setErrors] = useState({});
  const navigate = useNavigate();
  const [employees, setEmployees] = useState([]);
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [currentBatchTransactions, setCurrentBatchTransactions] = useState([]);

  const currencies = ['INR', 'USD', 'EUR', 'GBP', 'AED'];

  // Generate unique transaction ID
  const getTransactionId = () => `BATCH-${batchId}-TXN-${Date.now().toString(36).toUpperCase()}`;
  
  // Get current date
  const getCurrentDate = () => new Date().toISOString().split('T')[0];

  // Validation functions (keeping existing ones)
  const validateAccountNumber = (val) => /^[0-9]{8}$/.test(val);
  const validateBankId = (val) => /^[0-9]{7}$/.test(val);
  const validateNumeric = (val) => !isNaN(val) && val.trim() !== "";
  const validateAlphaNumeric = (val) => /^[a-zA-Z0-9 ]+$/.test(val);

  // Handle input changes with validation
  const handleInputChange = (e) => {
    const { name, value } = e.target;
    setForm({ ...form, [name]: value });
    
    let errorMsg = "";
    if (name === "debitAccount" || name === "creditAccount") {
      if (!validateAccountNumber(value)) errorMsg = "Account number must be 8 digits";
    } else if (name === "bankId") {
      if (!validateBankId(value)) errorMsg = "Bank ID must be 7 digits";
    } else if (name === "debitAmount") {
      if (!validateNumeric(value)) errorMsg = "Amount must be numeric";
    } else if (name === "employeeName") {
      if (!validateAlphaNumeric(value)) errorMsg = "Only letters, numbers and space allowed";
    }
    
    setErrors(prev => ({ ...prev, [name]: errorMsg }));
  };

  // Add payment to current batch
  const handleAddPayment = async () => {
    if (Object.values(errors).some(e => e !== "")) {
      alert("Please fix errors before adding payment");
      return;
    }

    if (!batchId || 
        !form.debitAccount || 
        !form.creditAccount || 
        !form.debitCurrency || 
        !form.debitAmount || 
        !form.creditCurrency || 
        !form.employeeName || 
        !form.bankId) {
      alert("Fill all required fields including Batch ID");
      return;
    }

    setIsSubmitting(true);
    try {
      const transactionData = {
        ...form,
        transactionId: getTransactionId(),
        date: getCurrentDate()
      };

      // Add transaction to backend
      await PayrollBatchService.addTransactionToBatch(batchId, transactionData);
      
      // Update local state
      setCurrentBatchTransactions(prev => [...prev, transactionData]);
      
      // Reset form
      setForm({
        debitAccount: '',
        creditAccount: '',
        debitCurrency: '',
        debitAmount: '',
        creditCurrency: '',
        employeeName: '',
        bankId: '',
        remarks: ''
      });

      alert("Payment added successfully!");
      
    } catch (error) {
      console.error('Error adding payment:', error);
      alert("Error adding payment. Please try again.");
    } finally {
      setIsSubmitting(false);
    }
  };

  // Submit batch for approval
  const handleSubmitBatch = async () => {
    if (currentBatchTransactions.length === 0) {
      alert("No transactions to submit");
      return;
    }

    setIsSubmitting(true);
    try {
      await PayrollBatchService.submitBatch(batchId, "submitted_for_approval", currentBatchTransactions);
      alert("Batch submitted for approval successfully!");
      
      // Reset everything after successful submission
      setBatchId('');
      setCurrentBatchTransactions([]);
      
    } catch (error) {
      console.error('Error submitting batch:', error);
      alert("Error submitting batch. Please try again.");
    } finally {
      setIsSubmitting(false);
    }
  };

  // Save batch as draft
  const handleSaveAsDraft = async () => {
    if (currentBatchTransactions.length === 0) {
      alert("No transactions to save");
      return;
    }

    setIsSubmitting(true);
    try {
      await PayrollBatchService.submitBatch(batchId, "draft", currentBatchTransactions);
      alert("Batch saved as draft successfully!");
      
    } catch (error) {
      console.error('Error saving draft:', error);
      alert("Error saving draft. Please try again.");
    } finally {
      setIsSubmitting(false);
    }
  };

  // Load existing batch
  const handleLoadBatch = async (selectedBatchId) => {
    if (!selectedBatchId) return;
    
    try {
      const batchData = await PayrollBatchService.getBatch(selectedBatchId);
      setBatchId(selectedBatchId);
      setCurrentBatchTransactions(batchData.transactions || []);
    } catch (error) {
      console.error('Error loading batch:', error);
      alert("Error loading batch");
    }
  };

  return (
    <Box sx={{ minWidth: '1300px', ml: 3, display: 'flex', gap: 1 }}>
      <Typography variant="h4" sx={{ fontWeight: 'bold', color: 'primary.main' }}>
        Payroll Payments
      </Typography>
      
      <Button variant="contained" color="primary" startIcon={<Send />} onClick={() => navigate('/quick-payments')}>
        Quick Payment
      </Button>

      <Grid container spacing={2}>
        <Grid item xs={7}>
          <Card sx={{ height: '100%', maxWidth: '720px' }}>
            <CardContent>
              <Typography variant="h6" sx={{ mb: 2, fontWeight: 600, color: 'primary.main' }}>
                CREATE PAYMENT
              </Typography>
              
              <Grid container spacing={2}>
                <Grid item xs={12} sm={6}>
                  <TextField
                    fullWidth
                    label="Batch ID"
                    name="batchId"
                    value={batchId}
                    onChange={(e) => setBatchId(e.target.value)}
                    error={!!errors.batchId}
                    helperText={errors.batchId}
                  />
                </Grid>
                
                {/* Rest of your form fields remain the same */}
                <Grid item xs={12} sm={6}>
                  <TextField
                    fullWidth
                    label="Employee Name"
                    name="employeeName"
                    value={form.employeeName}
                    onChange={handleInputChange}
                    error={!!errors.employeeName}
                    helperText={errors.employeeName}
                  />
                </Grid>

                {/* Add all other form fields similarly... */}
                
                <Grid item xs={12}>
                  <Button 
                    variant="contained" 
                    color="primary" 
                    startIcon={<Add />} 
                    onClick={handleAddPayment}
                    disabled={isSubmitting}
                    sx={{ p: 1.8 }}
                  >
                    Add Payment
                  </Button>
                </Grid>
              </Grid>
            </CardContent>
          </Card>
        </Grid>

        {/* Action buttons */}
        <Box sx={{ mt: 2, display: 'flex', gap: 1 }}>
          <Button 
            variant="contained" 
            color="primary" 
            startIcon={<Send />} 
            onClick={handleSubmitBatch}
            disabled={isSubmitting}
          >
            Submit
          </Button>
          <Button 
            variant="outlined" 
            startIcon={<Save />} 
            onClick={handleSaveAsDraft}
            disabled={isSubmitting}
          >
            Save as Draft
          </Button>
        </Box>

        {/* Display current batch transactions */}
        {currentBatchTransactions.length > 0 && (
          <Box sx={{ mt: 3 }}>
            <Typography variant="h6">Current Batch Transactions</Typography>
            <TableContainer component={Paper}>
              <Table size="small">
                <TableHead>
                  <TableRow>
                    <TableCell>Transaction ID</TableCell>
                    <TableCell>Employee</TableCell>
                    <TableCell>Amount</TableCell>
                    <TableCell>Actions</TableCell>
                  </TableRow>
                </TableHead>
                <TableBody>
                  {currentBatchTransactions.map((transaction, index) => (
                    <TableRow key={index}>
                      <TableCell>{transaction.transactionId}</TableCell>
                      <TableCell>{transaction.employeeName}</TableCell>
                      <TableCell>{transaction.debitCurrency} {transaction.debitAmount}</TableCell>
                      <TableCell>
                        <IconButton size="small" color="error">
                          <Delete />
                        </IconButton>
                      </TableCell>
                    </TableRow>
                  ))}
                </TableBody>
              </Table>
            </TableContainer>
          </Box>
        )}
      </Grid>
    </Box>
  );
}

export { PayrollBatchService };