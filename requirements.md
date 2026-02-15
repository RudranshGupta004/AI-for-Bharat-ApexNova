# Requirements Document

## Project Overview

**Project Name:** AI-Powered Mandal-Level Crop Advisory & Price Prediction System

**Target Hackathon:** AI for Bharat by AWS

**One-Line Summary:** An AI-powered, mandal-level agricultural decision system that predicts crop prices and yields using validated data, seasonal intelligence, and explainable AI — built on AWS for Bharat-scale impact.

## Problem Statement

Indian farmers face critical challenges in making informed crop decisions:

- Limited visibility into future crop prices and market trends
- Insufficient understanding of climate impact on yields
- Lack of localized production trend data at operational scale
- Existing advisory systems operate at state/district level (too coarse for mandal-level decisions)
- Weather variability directly impacts both yield and market prices
- Fragmented and noisy government datasets
- Absence of trustworthy, explainable AI recommendations

## Target Users

### Primary Users
- **Smallholder Farmers:** Operating at mandal level across India
- **Agricultural Extension Workers:** Providing advisory services to farmers
- **Farmer Producer Organizations (FPOs):** Making collective crop decisions

### Secondary Users
- **Agricultural Policy Makers:** Understanding ground-level trends
- **Agricultural Researchers:** Analyzing crop patterns and climate impact

## Functional Requirements

### FR1: Crop Price Prediction
- **FR1.1:** System shall predict future crop prices at mandal level
- **FR1.2:** Predictions shall be season-aware (aligned with sowing → harvest cycles)
- **FR1.3:** Price predictions shall account for expected supply and weather patterns
- **FR1.4:** System shall provide price range (min, max, expected) with confidence scores
- **FR1.5:** Historical price trends shall be available for comparison

### FR2: Crop Yield Estimation
- **FR2.1:** System shall estimate crop production/yield based on seasonal weather patterns
- **FR2.2:** Yield estimates shall be mandal-specific
- **FR2.3:** System shall factor in rainfall, humidity, wind, and temperature data
- **FR2.4:** Yield predictions shall include confidence intervals

### FR3: Crop Recommendation
- **FR3.1:** System shall recommend optimal crops for a given mandal and season
- **FR3.2:** Recommendations shall consider both yield potential and price forecasts
- **FR3.3:** System shall rank top 3-5 crop options with rationale
- **FR3.4:** Recommendations shall account for historical crop patterns in the region

### FR4: Explainable AI
- **FR4.1:** Every prediction shall include plain-language explanation
- **FR4.2:** System shall explain key factors influencing the prediction
- **FR4.3:** Weather impact on predictions shall be clearly communicated
- **FR4.4:** Confidence scores shall be provided for all predictions
- **FR4.5:** System shall never hallucinate numbers or fabricate data

### FR5: Conversational Interface
- **FR5.1:** Users shall interact via natural language queries
- **FR5.2:** System shall support queries in English (extensible to regional languages)
- **FR5.3:** Interface shall guide users through mandal, crop, and season selection
- **FR5.4:** System shall provide contextual follow-up suggestions

### FR6: Data Validation & Quality
- **FR6.1:** System shall validate agricultural production data using yield sanity checks
- **FR6.2:** Cross-year crop consistency checks shall be performed
- **FR6.3:** District-level aggregation shall serve as fallback for missing mandal data
- **FR6.4:** System shall filter out aggregate entries (totals, "others" categories)
- **FR6.5:** Data quality indicators shall be surfaced to users

### FR7: Seasonal Intelligence
- **FR7.1:** System shall align predictions with agricultural seasons (Kharif, Rabi, Zaid)
- **FR7.2:** Crops spanning across calendar years shall be handled correctly
- **FR7.3:** Prices shall be matched to actual selling seasons, not reporting years
- **FR7.4:** Season-specific weather patterns shall be aggregated appropriately

### FR8: Multi-Model Architecture
- **FR8.1:** Weather → Production Model shall predict yield from weather data
- **FR8.2:** Production → Market Price Model shall predict prices from supply data
- **FR8.3:** Weather → Market Model shall capture early price signals from climate anomalies
- **FR8.4:** Models shall operate independently before integration
- **FR8.5:** Model outputs shall be combined for final predictions

## Non-Functional Requirements

### NFR1: Performance
- **NFR1.1:** API response time shall be < 3 seconds for predictions
- **NFR1.2:** System shall support concurrent queries from 10,000+ users
- **NFR1.3:** ML model inference latency shall be < 1 second

### NFR2: Scalability
- **NFR2.1:** System shall scale to cover all mandals across India (5000+ mandals)
- **NFR2.2:** Architecture shall be serverless and auto-scaling
- **NFR2.3:** Data storage shall handle multi-year historical data efficiently

### NFR3: Reliability
- **NFR3.1:** System uptime shall be ≥ 99.5%
- **NFR3.2:** Graceful degradation when mandal-level data is unavailable
- **NFR3.3:** Fallback mechanisms for model failures

### NFR4: Accuracy
- **NFR4.1:** Price prediction accuracy shall be within ±15% of actual prices
- **NFR4.2:** Yield prediction accuracy shall be within ±20% of actual yields
- **NFR4.3:** Model performance shall be monitored and reported

### NFR5: Security
- **NFR5.1:** API endpoints shall be authenticated and rate-limited
- **NFR5.2:** User data shall be encrypted in transit and at rest
- **NFR5.3:** Compliance with data privacy regulations

### NFR6: Maintainability
- **NFR6.1:** Models shall be retrained quarterly with new data
- **NFR6.2:** System shall support A/B testing of model versions
- **NFR6.3:** Comprehensive logging and monitoring shall be implemented

### NFR7: Cost Efficiency
- **NFR7.1:** Serverless architecture to minimize idle costs
- **NFR7.2:** Efficient data storage and query optimization
- **NFR7.3:** Target cost: < ₹0.10 per prediction

## Data Requirements

### DR1: Agricultural Production Data
- **Source:** Government agricultural departments, AGMARKNET
- **Granularity:** Mandal-level
- **Attributes:** Crop name, area (hectares), production (tonnes), season, year
- **Quality:** Validated, aggregates removed

### DR2: Weather Data
- **Source:** IMD, weather APIs
- **Granularity:** Mandal/district level
- **Attributes:** Rainfall, temperature, humidity, wind speed
- **Temporal:** Daily data aggregated seasonally

### DR3: Market Price Data
- **Source:** AGMARKNET, state agricultural marketing boards
- **Granularity:** Market/AMC level, mapped to mandals
- **Attributes:** Commodity, price (min/max/modal), date, market
- **Temporal:** Daily/weekly prices

### DR4: Geospatial Data
- **Mandal boundaries and mappings**
- **Market to mandal proximity mapping**
- **District-level fallback mappings**

## Technical Constraints

### TC1: AWS Technology Stack
- Must use AWS services for deployment
- Leverage Amazon Bedrock for LLM capabilities
- Use Amazon SageMaker for ML model hosting

### TC2: LLM Usage Boundaries
- LLM shall NOT predict prices or yields directly
- LLM acts only as conversational agent and explainer
- All numeric predictions from ML models only

### TC3: Data Freshness
- Weather data updated daily
- Price data updated weekly
- Production data updated seasonally

## Success Criteria

### Quantitative Metrics
- **Adoption:** 1000+ farmer queries in pilot phase
- **Accuracy:** Price predictions within ±15%, yield within ±20%
- **Performance:** < 3 second response time for 95% of queries
- **Coverage:** Support for top 20 crops across 100+ mandals in pilot

### Qualitative Metrics
- **Trust:** Farmers understand and trust explanations
- **Usability:** Non-technical users can interact without training
- **Impact:** Farmers report improved decision confidence

## Out of Scope (Phase 1)

- Real-time market integration
- Soil health data integration
- Pest and disease prediction
- Direct market linkage/e-commerce
- Mobile app (web interface only)
- Regional language support beyond English
- Personalized recommendations based on individual farm data

## Assumptions

- Government agricultural data is accessible via APIs or bulk downloads
- Weather data APIs are available and reliable
- Farmers have basic smartphone/internet access
- Agricultural extension workers can facilitate system access
- Historical data for at least 3-5 years is available for model training

## Dependencies

- Access to government agricultural datasets
- AWS account with necessary service quotas
- Weather data API subscriptions
- Domain expertise for model validation
- User testing with actual farmers

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Data quality issues | High | Robust validation pipeline, fallback mechanisms |
| Model accuracy below threshold | High | Multi-model ensemble, continuous monitoring |
| Low farmer adoption | Medium | Simple UX, extension worker partnerships |
| API rate limits/costs | Medium | Caching, efficient queries, cost monitoring |
| Weather data unavailability | Medium | Multiple data sources, historical fallbacks |

## Compliance & Ethics

- No fabrication of predictions (anti-hallucination measures)
- Transparent about confidence levels and limitations
- Data privacy for user queries
- Responsible AI practices (fairness, accountability)
- Clear disclaimers about prediction uncertainty

## Future Enhancements (Post-Hackathon)

- Regional language support (Hindi, Telugu, Tamil, etc.)
- Mobile application (Android/iOS)
- Soil health integration
- Pest and disease alerts
- Direct market linkage
- Personalized farm-level recommendations
- Integration with government subsidy schemes
- Community features (farmer forums, success stories)
