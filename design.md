# Design Document

## Project Overview

**Project Name:** AI-Powered Mandal-Level Crop Advisory & Price Prediction System

**Architecture Style:** Serverless, Event-Driven, Microservices

**Cloud Provider:** Amazon Web Services (AWS)

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Interface Layer                     │
│                    (AWS Amplify - Web Frontend)                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ HTTPS
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API Gateway Layer                           │
│              (Amazon API Gateway + AWS WAF)                      │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ REST/GraphQL
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Application Layer                             │
│                     (AWS Lambda Functions)                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Query Handler│  │ LLM Agent    │  │ Data Fetcher │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└────────────────────────────┬────────────────────────────────────┘
                             │
                ┌────────────┼────────────┐
                │            │            │
                ▼            ▼            ▼
┌──────────────────┐ ┌──────────────┐ ┌──────────────────┐
│   ML Layer       │ │  LLM Layer   │ │  Data Layer      │
│  (SageMaker)     │ │  (Bedrock)   │ │  (S3 + Athena)   │
│                  │ │              │ │                  │
│ • Weather→Prod   │ │ • Claude/    │ │ • Raw Data (S3)  │
│ • Prod→Price     │ │   Llama      │ │ • DuckDB/Athena  │
│ • Weather→Market │ │ • Prompt Eng │ │ • Data Catalog   │
└──────────────────┘ └──────────────┘ └──────────────────┘
```

## Component Design

### 1. Frontend Layer (AWS Amplify)

**Technology:** React.js hosted on AWS Amplify

**Key Components:**
- **Conversational Interface:** Chat-like UI for farmer queries
- **Selection Widgets:** Dropdowns for mandal, crop, season selection
- **Visualization Dashboard:** Charts for price trends, yield forecasts
- **Explanation Panel:** Display AI reasoning and confidence scores

**Design Decisions:**
- Progressive Web App (PWA) for mobile accessibility
- Lightweight UI for low-bandwidth scenarios
- Offline capability for basic information
- Responsive design for various screen sizes

### 2. API Gateway Layer

**Technology:** Amazon API Gateway + AWS WAF

**Endpoints:**

```
POST /api/v1/predict/price
POST /api/v1/predict/yield
POST /api/v1/recommend/crop
POST /api/v1/query/conversational
GET  /api/v1/data/mandals
GET  /api/v1/data/crops
GET  /api/v1/data/seasons
```

**Features:**
- Request validation and sanitization
- Rate limiting (100 requests/minute per user)
- API key authentication
- CORS configuration
- Request/response logging

### 3. Application Layer (AWS Lambda)

#### Lambda Function: Query Handler

**Purpose:** Route and orchestrate user queries

**Input:**
```json
{
  "query": "What will be the price of paddy in Guntur mandal this Kharif season?",
  "mandal": "Guntur",
  "crop": "Paddy",
  "season": "Kharif 2026"
}
```

**Process:**
1. Parse and validate input
2. Determine query intent (price/yield/recommendation)
3. Fetch relevant data
4. Call appropriate ML models
5. Invoke LLM for explanation
6. Format and return response

**Output:**
```json
{
  "prediction": {
    "type": "price",
    "crop": "Paddy",
    "mandal": "Guntur",
    "season": "Kharif 2026",
    "price_range": {
      "min": 1850,
      "max": 2150,
      "expected": 2000
    },
    "confidence": 0.78,
    "unit": "INR per quintal"
  },
  "explanation": {
    "summary": "Expected paddy prices in Guntur for Kharif 2026 are around ₹2000/quintal based on normal monsoon predictions and stable production levels.",
    "factors": [
      "Normal rainfall expected (85-95% of average)",
      "Production estimated at 12,500 tonnes",
      "Historical price trend shows 5% annual increase"
    ],
    "weather_impact": "Moderate positive impact from good monsoon forecast",
    "confidence_note": "78% confidence based on 5 years of historical data"
  },
  "metadata": {
    "timestamp": "2026-02-15T10:30:00Z",
    "models_used": ["weather_prod_v2", "prod_price_v2", "weather_market_v1"],
    "data_sources": ["IMD", "AGMARKNET", "State Agri Dept"]
  }
}
```

#### Lambda Function: LLM Agent

**Purpose:** Generate natural language explanations

**Technology:** Amazon Bedrock (Claude 3 or Llama 3)

**Responsibilities:**
- Parse user natural language queries
- Generate plain-language explanations of predictions
- Provide contextual follow-up suggestions
- Format responses for readability

**Prompt Template:**
```
You are an agricultural advisor helping Indian farmers make crop decisions.

Context:
- Mandal: {mandal}
- Crop: {crop}
- Season: {season}

Prediction Data:
- Expected Yield: {yield} tonnes
- Expected Price: ₹{price} per quintal
- Weather Impact: {weather_summary}
- Confidence: {confidence}%

Task: Explain this prediction to a farmer in simple, clear language. Focus on:
1. What the numbers mean for their income
2. Why this prediction was made (key factors)
3. What weather conditions are expected
4. How confident we are and why

Rules:
- Use simple language (avoid technical jargon)
- Be honest about uncertainty
- Never make up numbers
- Provide actionable insights
```

**Anti-Hallucination Measures:**
- Structured output format
- Numeric values passed as context, not generated
- Validation of LLM output against input data
- Confidence thresholds for explanations

#### Lambda Function: Data Fetcher

**Purpose:** Retrieve and validate data from storage

**Responsibilities:**
- Query DuckDB/Athena for historical data
- Validate data quality
- Apply sanity checks
- Handle missing data with fallbacks

**Data Validation Logic:**
```python
def validate_production_data(data):
    # Yield sanity check
    if data['area'] > 0:
        yield_per_hectare = data['production'] / data['area']
        if yield_per_hectare < MIN_YIELD or yield_per_hectare > MAX_YIELD:
            return False, "Yield out of expected range"
    
    # Cross-year consistency
    if check_historical_deviation(data) > THRESHOLD:
        return False, "Unusual deviation from historical pattern"
    
    return True, "Valid"

def apply_fallback(mandal, crop, season):
    # Try district-level data
    district_data = get_district_data(mandal.district, crop, season)
    if district_data:
        return district_data
    
    # Try state-level data
    state_data = get_state_data(mandal.state, crop, season)
    return state_data
```

### 4. ML Model Layer (Amazon SageMaker)

#### Model 1: Weather → Production Model

**Algorithm:** Gradient Boosting (XGBoost) or Random Forest

**Input Features:**
- Seasonal rainfall (mm)
- Average temperature (°C)
- Humidity (%)
- Wind speed (km/h)
- Crop type (encoded)
- Mandal (encoded)
- Season (encoded)
- Historical production (lagged features)

**Output:** Predicted crop production (tonnes)

**Training Data:** 5+ years of mandal-level production + weather data

**Model Architecture:**
```
Input Layer (15 features)
    ↓
Feature Engineering
    ↓
XGBoost Regressor
    ↓
Output: Production (tonnes) + Confidence Interval
```

#### Model 2: Production → Market Price Model

**Algorithm:** Time Series Forecasting (LSTM or Prophet)

**Input Features:**
- Expected production (from Model 1)
- Historical prices (time series)
- Season
- Market proximity
- Demand indicators

**Output:** Predicted market price (INR per quintal)

**Model Architecture:**
```
Input Layer (Production + Historical Prices)
    ↓
LSTM Layers (2 layers, 128 units)
    ↓
Dense Layer (64 units)
    ↓
Output: Price + Confidence Interval
```

#### Model 3: Weather → Market Model

**Algorithm:** Ensemble (XGBoost + Neural Network)

**Purpose:** Capture early price signals from weather anomalies

**Input Features:**
- Weather forecasts
- Historical weather-price correlations
- Crop type
- Season

**Output:** Price adjustment factor

**Integration Logic:**
```python
def combine_predictions(weather_prod, prod_price, weather_market):
    # Base prediction from production-price model
    base_price = prod_price.predict()
    
    # Weather adjustment
    weather_adjustment = weather_market.predict()
    
    # Weighted combination
    final_price = (
        0.6 * base_price +
        0.3 * weather_adjustment +
        0.1 * historical_average
    )
    
    # Confidence calculation
    confidence = calculate_ensemble_confidence([
        weather_prod.confidence,
        prod_price.confidence,
        weather_market.confidence
    ])
    
    return final_price, confidence
```

**Model Deployment:**
- SageMaker endpoints with auto-scaling
- Model versioning and A/B testing
- Real-time inference (< 1 second latency)
- Batch inference for bulk predictions

### 5. Data Layer

#### Data Storage (Amazon S3)

**Bucket Structure:**
```
s3://agri-advisory-data/
├── raw/
│   ├── production/
│   │   ├── year=2021/
│   │   ├── year=2022/
│   │   └── year=2023/
│   ├── weather/
│   │   ├── year=2021/
│   │   └── year=2022/
│   └── prices/
│       ├── year=2021/
│       └── year=2022/
├── processed/
│   ├── mandal_features/
│   ├── seasonal_aggregates/
│   └── validated_data/
└── models/
    ├── weather_prod/
    ├── prod_price/
    └── weather_market/
```

#### Data Processing (AWS Glue / Lambda)

**ETL Pipeline:**
1. **Ingestion:** Fetch data from government APIs/portals
2. **Validation:** Apply quality checks
3. **Transformation:** 
   - Remove aggregates (totals, "others")
   - Standardize crop names
   - Map markets to mandals
   - Aggregate weather data seasonally
4. **Storage:** Write to S3 in Parquet format

**Seasonal Alignment Logic:**
```python
def align_to_season(crop, date, production, price):
    """
    Align data to agricultural seasons, not calendar years
    """
    season_config = {
        'Paddy': {
            'Kharif': {'sowing': 'Jun-Jul', 'harvest': 'Oct-Nov'},
            'Rabi': {'sowing': 'Nov-Dec', 'harvest': 'Mar-Apr'}
        },
        # ... other crops
    }
    
    # Determine season from date
    season = determine_season(crop, date)
    
    # Map production to harvest season
    production_season = season_config[crop][season]['harvest']
    
    # Map price to selling season (post-harvest)
    price_season = add_months(production_season, 1)
    
    return {
        'crop': crop,
        'season': season,
        'production': production,
        'price': price,
        'aligned_date': price_season
    }
```

#### Query Engine (DuckDB / Amazon Athena)

**Purpose:** Fast analytical queries on S3 data

**Sample Queries:**
```sql
-- Get mandal-level production for a crop and season
SELECT 
    mandal,
    crop,
    season,
    AVG(production) as avg_production,
    AVG(area) as avg_area,
    AVG(production/area) as avg_yield
FROM production_data
WHERE crop = 'Paddy' 
  AND season = 'Kharif'
  AND year >= 2021
GROUP BY mandal, crop, season;

-- Get seasonal weather patterns
SELECT 
    mandal,
    season,
    AVG(rainfall) as avg_rainfall,
    AVG(temperature) as avg_temp,
    AVG(humidity) as avg_humidity
FROM weather_data
WHERE year >= 2021
GROUP BY mandal, season;
```

### 6. LLM Layer (Amazon Bedrock)

**Model Selection:** Claude 3 Sonnet or Llama 3 70B

**Use Cases:**
- Query understanding and intent classification
- Natural language explanation generation
- Conversational follow-ups
- Context-aware responses

**Integration Pattern:**
```python
import boto3

bedrock = boto3.client('bedrock-runtime')

def generate_explanation(prediction_data, user_context):
    prompt = build_prompt(prediction_data, user_context)
    
    response = bedrock.invoke_model(
        modelId='anthropic.claude-3-sonnet-20240229-v1:0',
        body=json.dumps({
            'anthropic_version': 'bedrock-2023-05-31',
            'max_tokens': 500,
            'messages': [{
                'role': 'user',
                'content': prompt
            }],
            'temperature': 0.3,  # Low temperature for consistency
            'top_p': 0.9
        })
    )
    
    # Validate output
    explanation = parse_response(response)
    validate_no_hallucination(explanation, prediction_data)
    
    return explanation
```

## Data Flow

### End-to-End Flow: Price Prediction Query

```
1. User Input
   ↓
   "What will be the price of paddy in Guntur this Kharif?"
   ↓
2. API Gateway
   ↓
   Validate, authenticate, route to Lambda
   ↓
3. Query Handler Lambda
   ↓
   Parse query → Extract: mandal=Guntur, crop=Paddy, season=Kharif
   ↓
4. Data Fetcher Lambda
   ↓
   Fetch: weather forecast, historical production, historical prices
   ↓
5. ML Models (SageMaker)
   ↓
   Model 1: Weather → Production (12,500 tonnes)
   Model 2: Production → Price (₹2000/quintal)
   Model 3: Weather → Market (adjustment: +2%)
   ↓
   Combine: Final Price = ₹2040/quintal, Confidence = 78%
   ↓
6. LLM Agent (Bedrock)
   ↓
   Generate explanation in simple language
   ↓
7. Response Formatting
   ↓
   JSON response with prediction + explanation
   ↓
8. API Gateway → Frontend
   ↓
9. Display to User
```

## Security Design

### Authentication & Authorization
- API Gateway with API keys
- AWS IAM roles for service-to-service communication
- Rate limiting per user/IP

### Data Security
- S3 bucket encryption (SSE-S3)
- Data in transit: TLS 1.2+
- VPC for Lambda functions (if needed)
- Secrets Manager for API keys

### Input Validation
- Schema validation at API Gateway
- SQL injection prevention
- XSS protection in frontend

## Monitoring & Observability

### Metrics (Amazon CloudWatch)
- API latency (p50, p95, p99)
- Lambda execution time
- Model inference latency
- Error rates
- Data freshness

### Logging
- API Gateway access logs
- Lambda function logs
- Model prediction logs (input/output)
- Data pipeline logs

### Alerting
- High error rates (> 5%)
- Slow response times (> 5 seconds)
- Model accuracy degradation
- Data pipeline failures

## Deployment Strategy

### Infrastructure as Code
- AWS CDK or Terraform for infrastructure
- CI/CD pipeline (AWS CodePipeline)
- Automated testing before deployment

### Deployment Stages
1. **Dev:** Development and testing
2. **Staging:** Pre-production validation
3. **Production:** Live system

### Model Deployment
- Blue-green deployment for models
- Canary releases (10% → 50% → 100%)
- Rollback capability

## Cost Optimization

### Estimated Monthly Costs (1000 users, 10K queries/month)

| Service | Usage | Cost |
|---------|-------|------|
| API Gateway | 10K requests | $0.04 |
| Lambda | 10K invocations, 1GB, 3s avg | $0.20 |
| SageMaker | 3 endpoints, ml.t3.medium | $150 |
| Bedrock | 10K invocations, 500 tokens avg | $15 |
| S3 | 100GB storage, 50GB transfer | $5 |
| Athena | 10GB scanned | $0.05 |
| **Total** | | **~$170/month** |

**Cost per prediction:** ~$0.017 (₹1.40)

### Optimization Strategies
- Use Lambda reserved concurrency for predictable workloads
- SageMaker Serverless Inference for variable traffic
- S3 Intelligent-Tiering for data storage
- Athena query result caching
- Bedrock prompt optimization (reduce tokens)

## Scalability Design

### Horizontal Scaling
- Lambda auto-scales automatically
- SageMaker endpoints with auto-scaling policies
- API Gateway handles high throughput

### Data Scaling
- Partitioned S3 data (by year, state, crop)
- Athena query optimization with partitions
- DuckDB for fast local analytics

### Geographic Scaling
- CloudFront for frontend distribution
- Multi-region deployment (future)

## Testing Strategy

### Unit Testing
- Lambda function logic
- Data validation functions
- Model preprocessing/postprocessing

### Integration Testing
- API endpoint testing
- End-to-end flow testing
- Model inference testing

### Model Testing
- Accuracy metrics (MAE, RMSE, MAPE)
- Backtesting on historical data
- A/B testing in production

### User Acceptance Testing
- Pilot with 50-100 farmers
- Feedback collection
- Usability testing

## Disaster Recovery

### Backup Strategy
- S3 versioning enabled
- Cross-region replication for critical data
- Model artifacts backed up

### Recovery Plan
- RTO (Recovery Time Objective): 4 hours
- RPO (Recovery Point Objective): 24 hours
- Automated failover for critical services

## Future Enhancements

### Phase 2 Features
- Real-time market price integration
- Soil health data integration
- Mobile app (React Native)
- Regional language support

### Phase 3 Features
- Personalized recommendations
- Community features
- Direct market linkage
- Government scheme integration

## Technology Alternatives Considered

| Component | Chosen | Alternative | Reason |
|-----------|--------|-------------|--------|
| Frontend | Amplify | EC2 + Nginx | Serverless, easier deployment |
| ML Hosting | SageMaker | EC2 + Docker | Managed service, auto-scaling |
| LLM | Bedrock | Self-hosted | Managed, cost-effective |
| Database | S3 + Athena | RDS | Analytical workload, cost |
| Compute | Lambda | ECS/EKS | Event-driven, serverless |

## Design Principles

1. **Simplicity First:** Avoid over-engineering
2. **Data Quality Over Quantity:** Validate before use
3. **Explainability:** Every prediction must be explainable
4. **Fail Gracefully:** Fallbacks for missing data
5. **Cost-Conscious:** Optimize for low operational cost
6. **Farmer-Centric:** Design for end-user needs
7. **Scalable by Default:** Serverless architecture

## Conclusion

This design provides a robust, scalable, and cost-effective solution for mandal-level agricultural advisory. The multi-model AI approach ensures accurate predictions, while the LLM layer provides explainability and trust. The serverless AWS architecture enables Bharat-scale deployment with minimal operational overhead.
