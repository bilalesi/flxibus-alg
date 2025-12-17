# AI Integration Roadmap & Implementation Strategy

## Executive Summary

This document outlines a phased approach to integrating AI capabilities into the Algeria Transportation Platform, transforming it from a booking platform into an intelligent travel assistant. AI will enhance customer experience, optimize operations, and create competitive moats through data-driven insights.

## Phase 1: Foundation (Months 1-6)

### 1.1 Intelligent Search & Recommendations

**Use Cases**:
- Natural language search queries
- Personalized route recommendations
- Price prediction and alerts

**Implementation**:

```typescript
// AI-Powered Search Service
import Anthropic from '@anthropic-ai/sdk';
import { db } from '../db';
import { routes, bookings, users } from '../db/schema';
import { eq } from 'drizzle-orm';

export class AISearchService {
  private client: Anthropic;

  constructor() {
    this.client = new Anthropic({
      apiKey: process.env.ANTHROPIC_API_KEY!,
    });
  }

  async naturalLanguageSearch(query: string, userId?: number) {
    // Get user context
    const userContext = userId 
      ? await this.getUserTravelProfile(userId)
      : null;

    // Use Claude to understand intent
    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      messages: [{
        role: 'user',
        content: `Parse this travel query and extract search parameters in JSON format.
        
Query: "${query}"
${userContext ? `User Context: ${JSON.stringify(userContext)}` : ''}

Return JSON with: origin, destination, date (YYYY-MM-DD), passengers, preferences.
If any field is unclear, make reasonable assumptions based on context.`
      }],
    });

    const searchParams = JSON.parse(
      message.content[0].type === 'text' 
        ? message.content[0].text 
        : '{}'
    );

    // Execute database search
    const results = await this.searchRoutes(searchParams);

    // Rank and personalize results
    return this.personalizeResults(results, userContext);
  }

  async recommendRoutes(userId: number) {
    // Get user's booking history
    const history = await db.query.bookings.findMany({
      where: eq(bookings.userId, userId),
      with: { route: true },
      orderBy: [desc(bookings.createdAt)],
      limit: 10,
    });

    // Analyze patterns
    const travelProfile = this.analyzeTravelPatterns(history);

    // Use Claude for intelligent recommendations
    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 2048,
      messages: [{
        role: 'user',
        content: `Based on this user's travel history, recommend 5 routes they might be interested in.
        
Travel History: ${JSON.stringify(travelProfile)}
Available Routes: ${JSON.stringify(await this.getAllActiveRoutes())}

Consider: frequency, seasonality, price sensitivity, destination patterns.
Return JSON array with route IDs and reasoning.`
      }],
    });

    return this.parseRecommendations(message);
  }

  private async getUserTravelProfile(userId: number) {
    const [user, bookings, preferences] = await Promise.all([
      db.query.users.findFirst({ where: eq(users.id, userId) }),
      db.query.bookings.findMany({
        where: eq(bookings.userId, userId),
        with: { route: true },
        limit: 20,
      }),
      this.getUserPreferences(userId),
    ]);

    return {
      preferredOrigins: this.extractFrequent(bookings, 'origin'),
      preferredDestinations: this.extractFrequent(bookings, 'destination'),
      averagePrice: this.calculateAverage(bookings, 'price'),
      bookingLeadTime: this.calculateLeadTime(bookings),
      preferences,
    };
  }
}
```

**Features**:
- "Find me a bus from Algiers to Oran next Friday"
- "What's the cheapest way to get to Constantine?"
- "Show me weekend trips under 2000 DZD"

### 1.2 Smart Chatbot (WhatsApp & In-App)

**Use Cases**:
- Booking assistance
- FAQ automation
- Real-time support
- Multi-language (Arabic/French/English)

**Implementation**:

```typescript
// AI Chatbot Service
export class AIChatbotService {
  private client: Anthropic;
  private conversationMemory: Map<string, Message[]> = new Map();

  async chat(userId: string, message: string, context: ConversationContext) {
    // Get conversation history
    const history = this.conversationMemory.get(userId) || [];

    // System prompt with business knowledge
    const systemPrompt = `You are a helpful travel assistant for an Algeria bus transportation platform.

CAPABILITIES:
- Search routes and check availability
- Book tickets
- Modify/cancel bookings
- Answer questions about routes, prices, policies
- Provide travel recommendations

TOOLS AVAILABLE:
- search_routes(origin, destination, date)
- check_availability(routeId, date)
- create_booking(routeId, passengers)
- get_booking(bookingRef)
- cancel_booking(bookingRef)

CONTEXT:
- User ID: ${userId}
- Current bookings: ${await this.getUserBookings(userId)}
- Language preference: ${context.language}

Always be helpful, friendly, and clear. Respond in ${context.language}.`;

    const response = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 2048,
      system: systemPrompt,
      messages: [
        ...history,
        { role: 'user', content: message },
      ],
      tools: [
        {
          name: 'search_routes',
          description: 'Search for available bus routes',
          input_schema: {
            type: 'object',
            properties: {
              origin: { type: 'string' },
              destination: { type: 'string' },
              date: { type: 'string', format: 'date' },
              passengers: { type: 'integer', default: 1 },
            },
            required: ['origin', 'destination', 'date'],
          },
        },
        {
          name: 'create_booking',
          description: 'Create a new booking',
          input_schema: {
            type: 'object',
            properties: {
              routeId: { type: 'integer' },
              departureTime: { type: 'string' },
              seats: { type: 'integer' },
              passengers: { type: 'array' },
            },
            required: ['routeId', 'departureTime', 'passengers'],
          },
        },
        // ... other tools
      ],
    });

    // Handle tool use
    if (response.stop_reason === 'tool_use') {
      const toolResults = await this.executeTools(response.content);
      
      // Continue conversation with tool results
      const finalResponse = await this.client.messages.create({
        model: 'claude-sonnet-4-20250514',
        max_tokens: 2048,
        system: systemPrompt,
        messages: [
          ...history,
          { role: 'user', content: message },
          { role: 'assistant', content: response.content },
          { role: 'user', content: toolResults },
        ],
      });

      return this.formatResponse(finalResponse);
    }

    // Update conversation memory
    history.push(
      { role: 'user', content: message },
      { role: 'assistant', content: response.content }
    );
    this.conversationMemory.set(userId, history.slice(-10)); // Keep last 10 messages

    return this.formatResponse(response);
  }

  private async executeTools(content: ContentBlock[]) {
    const results = [];

    for (const block of content) {
      if (block.type === 'tool_use') {
        const result = await this.executeTool(block.name, block.input);
        results.push({
          type: 'tool_result',
          tool_use_id: block.id,
          content: JSON.stringify(result),
        });
      }
    }

    return results;
  }

  private async executeTool(name: string, input: any) {
    switch (name) {
      case 'search_routes':
        return await this.routeService.search(input);
      case 'create_booking':
        return await this.bookingService.create(input);
      case 'get_booking':
        return await this.bookingService.getByRef(input.bookingRef);
      case 'cancel_booking':
        return await this.bookingService.cancel(input.bookingRef);
      default:
        throw new Error(`Unknown tool: ${name}`);
    }
  }
}
```

**Channels**:
- WhatsApp Business API integration
- In-app chat widget
- Facebook Messenger
- SMS (fallback)

### 1.3 Automated Customer Support

**Use Cases**:
- Ticket deflection (reduce support load 40-60%)
- Sentiment analysis
- Priority routing
- Auto-responses for common queries

**Implementation**:

```typescript
// Support Ticket Classifier
export class SupportAIService {
  async classifyTicket(ticket: SupportTicket) {
    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 512,
      messages: [{
        role: 'user',
        content: `Classify this support ticket:

Subject: ${ticket.subject}
Message: ${ticket.message}
User: ${ticket.userName}
Booking: ${ticket.bookingRef || 'N/A'}

Return JSON with:
- category: [booking, payment, route, complaint, inquiry, other]
- priority: [low, medium, high, urgent]
- sentiment: [positive, neutral, negative]
- suggestedResponse: brief response if simple query
- requiresHuman: boolean (true if complex/sensitive)`
      }],
    });

    const classification = JSON.parse(
      message.content[0].type === 'text' 
        ? message.content[0].text 
        : '{}'
    );

    // Auto-respond if simple
    if (!classification.requiresHuman && classification.suggestedResponse) {
      await this.sendAutoResponse(ticket, classification.suggestedResponse);
    }

    return classification;
  }

  async generateResponse(ticket: SupportTicket, context: string) {
    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      messages: [{
        role: 'user',
        content: `Generate a professional, helpful response to this support ticket:

Ticket: ${JSON.stringify(ticket)}
Context: ${context}

Guidelines:
- Be empathetic and professional
- Provide clear, actionable steps
- Include relevant links/references
- Match the customer's language (Arabic/French)
- Acknowledge their frustration if negative sentiment`
      }],
    });

    return message.content[0].type === 'text' 
      ? message.content[0].text 
      : '';
  }
}
```

## Phase 2: Intelligence (Months 7-12)

### 2.1 Dynamic Demand Forecasting

**Use Cases**:
- Predict booking demand per route
- Optimize fleet allocation
- Suggest new routes
- Dynamic pricing refinement

**Implementation**:

```typescript
// Demand Forecasting Service
import * as tf from '@tensorflow/tfjs-node';

export class DemandForecastingService {
  private model: tf.LayersModel;

  async predictDemand(routeId: number, date: Date, horizon: number = 30) {
    // Get historical data
    const historical = await this.getHistoricalBookings(routeId, 365);

    // Feature engineering
    const features = this.extractFeatures(historical, date);

    // Use both ML and AI
    const mlPrediction = await this.mlPredict(features);
    const aiInsights = await this.aiAnalyze(routeId, historical, date);

    return {
      predicted: mlPrediction,
      insights: aiInsights,
      confidence: this.calculateConfidence(mlPrediction, aiInsights),
    };
  }

  private async aiAnalyze(routeId: number, data: any[], date: Date) {
    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 2048,
      messages: [{
        role: 'user',
        content: `Analyze this route's booking patterns and predict demand:

Route: ${routeId}
Historical Data: ${JSON.stringify(data.slice(-90))}
Target Date: ${date.toISOString()}

Consider:
- Seasonal trends
- Day of week patterns
- Holiday impacts
- Recent trends
- External factors (events, weather, etc.)

Return JSON with:
- predictedBookings: number
- confidence: 0-100
- factors: array of key influencing factors
- recommendations: operational suggestions`
      }],
    });

    return JSON.parse(
      message.content[0].type === 'text' 
        ? message.content[0].text 
        : '{}'
    );
  }

  async suggestNewRoutes() {
    // Analyze booking patterns for underserved routes
    const patterns = await this.analyzeBookingPatterns();
    const competitorData = await this.getCompetitorRoutes();
    const demographicData = await this.getDemographicData();

    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 4096,
      messages: [{
        role: 'user',
        content: `Suggest profitable new bus routes for Algeria:

Current Network: ${JSON.stringify(patterns.currentRoutes)}
Booking Patterns: ${JSON.stringify(patterns.demandHotspots)}
Competitor Routes: ${JSON.stringify(competitorData)}
Demographics: ${JSON.stringify(demographicData)}

Analyze:
- Underserved city pairs
- High search-to-booking failure (demand without supply)
- Seasonal opportunities
- University/corporate corridors

Return top 10 route suggestions with:
- origin/destination
- estimated annual demand
- competitive analysis
- required frequency
- pricing recommendation
- rationale`
      }],
    });

    return this.parseRouteSuggestions(message);
  }
}
```

### 2.2 Personalization Engine

**Use Cases**:
- Personalized homepage
- Smart notifications
- Custom offers
- Loyalty program optimization

**Implementation**:

```typescript
// Personalization Engine
export class PersonalizationService {
  async personalizeExperience(userId: number) {
    const profile = await this.buildUserProfile(userId);

    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 3072,
      messages: [{
        role: 'user',
        content: `Create personalized content for this user:

Profile: ${JSON.stringify(profile)}

Generate:
1. Homepage hero message (personalized greeting)
2. 3 route recommendations with reasoning
3. 2 promotional offers tailored to their behavior
4. Smart notification schedule (when to send reminders)
5. Loyalty reward suggestions

Return structured JSON.`
      }],
    });

    return JSON.parse(
      message.content[0].type === 'text' 
        ? message.content[0].text 
        : '{}'
    );
  }

  async optimizePricing(userId: number, routeId: number, date: string) {
    const userSensitivity = await this.calculatePriceSensitivity(userId);
    const routeDemand = await this.getRouteDemand(routeId, date);

    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      messages: [{
        role: 'user',
        content: `Optimize pricing for this booking:

User Price Sensitivity: ${userSensitivity}
Route Demand: ${JSON.stringify(routeDemand)}
Base Price: ${routeDemand.basePrice}

Calculate optimal price that:
- Maximizes revenue
- Maintains conversion probability >70%
- Stays within acceptable range (±30% of base)

Return JSON with: price, discount (if any), reasoning`
      }],
    });

    return JSON.parse(
      message.content[0].type === 'text' 
        ? message.content[0].text 
        : '{}'
    );
  }
}
```

### 2.3 Intelligent Routing Optimization

**Use Cases**:
- Multi-city trip planning
- Connection optimization
- Alternative route suggestions
- Travel time optimization

**Implementation**:

```typescript
// Route Optimization Service
export class RouteOptimizationService {
  async findOptimalJourney(
    origin: string,
    destination: string,
    preferences: TravelPreferences
  ) {
    // Get all possible routes (direct + connections)
    const possibleRoutes = await this.findAllPossibleRoutes(
      origin,
      destination
    );

    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 4096,
      messages: [{
        role: 'user',
        content: `Find the optimal journey:

Origin: ${origin}
Destination: ${destination}
Preferences: ${JSON.stringify(preferences)}

Available Routes: ${JSON.stringify(possibleRoutes)}

Consider:
- Total travel time
- Number of connections
- Connection time (min 30min, max 3hours)
- Price
- Comfort level
- Departure/arrival times

Rank top 5 options with trade-offs explained.
Return JSON array with full itinerary details.`
      }],
    });

    return JSON.parse(
      message.content[0].type === 'text' 
        ? message.content[0].text 
        : '[]'
    );
  }
}
```

## Phase 3: Advanced Features (Months 13-18)

### 3.1 Voice Assistant Integration

**Channels**:
- Mobile app voice commands
- Phone booking hotline (IVR with AI)
- Smart speaker integration (future)

**Implementation**:

```typescript
// Voice Assistant Service
import { Deepgram } from '@deepgram/sdk';

export class VoiceAssistantService {
  private deepgram: Deepgram;
  private chatbot: AIChatbotService;

  async processVoiceCommand(audioBuffer: Buffer, language: string) {
    // Speech to text
    const transcript = await this.speechToText(audioBuffer, language);

    // Process with chatbot
    const response = await this.chatbot.chat(
      userId,
      transcript,
      { language, channel: 'voice' }
    );

    // Text to speech
    const audio = await this.textToSpeech(response, language);

    return { transcript, response, audio };
  }

  private async speechToText(audio: Buffer, language: string) {
    const { result } = await this.deepgram.listen.prerecorded.transcribeFile(
      audio,
      {
        model: language === 'ar' ? 'general' : 'nova-2',
        language: language === 'ar' ? 'ar' : 'fr',
      }
    );

    return result.results.channels[0].alternatives[0].transcript;
  }
}
```

### 3.2 Predictive Maintenance

**Use Cases**:
- Predict bus maintenance needs
- Optimize fleet health
- Reduce breakdowns
- Partner performance monitoring

**Implementation**:

```typescript
// Predictive Maintenance Service
export class PredictiveMaintenanceService {
  async analyzeBusHealth(busId: number) {
    const telemetry = await this.getBusTelemetry(busId);
    const maintenanceHistory = await this.getMaintenanceHistory(busId);

    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 2048,
      messages: [{
        role: 'user',
        content: `Analyze bus health and predict maintenance needs:

Bus ID: ${busId}
Telemetry: ${JSON.stringify(telemetry)}
Maintenance History: ${JSON.stringify(maintenanceHistory)}

Assess:
- Critical issues requiring immediate attention
- Wear patterns suggesting upcoming issues
- Optimal maintenance schedule
- Cost-benefit of repairs vs replacement

Return JSON with risk assessment and recommendations.`
      }],
    });

    return JSON.parse(
      message.content[0].type === 'text' 
        ? message.content[0].text 
        : '{}'
    );
  }
}
```

### 3.3 Fraud Detection & Security

**Use Cases**:
- Detect fraudulent bookings
- Identify suspicious patterns
- Payment fraud prevention
- Account takeover detection

**Implementation**:

```typescript
// Fraud Detection Service
export class FraudDetectionService {
  async analyzeBooking(booking: BookingAttempt, userContext: UserContext) {
    const signals = {
      booking,
      userHistory: await this.getUserHistory(booking.userId),
      deviceFingerprint: userContext.device,
      ipGeolocation: userContext.ipGeolocation,
      behaviorPatterns: await this.analyzeBehavior(booking.userId),
    };

    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      messages: [{
        role: 'user',
        content: `Analyze this booking for fraud signals:

Signals: ${JSON.stringify(signals)}

Red Flags:
- Multiple bookings in short time
- Inconsistent location/device
- Unusual payment method
- New account with high-value booking
- Suspicious patterns

Return JSON with:
- riskScore: 0-100
- flags: array of concerns
- recommendation: [approve, review, reject]
- reasoning: explanation`
      }],
    });

    const analysis = JSON.parse(
      message.content[0].type === 'text' 
        ? message.content[0].text 
        : '{}'
    );

    // Automatic actions based on risk score
    if (analysis.riskScore > 80) {
      await this.blockBooking(booking, analysis);
    } else if (analysis.riskScore > 50) {
      await this.flagForReview(booking, analysis);
    }

    return analysis;
  }
}
```

### 3.4 Intelligent Content Generation

**Use Cases**:
- Marketing copy generation
- Social media posts
- Email campaigns
- Route descriptions
- Blog articles

**Implementation**:

```typescript
// Content Generation Service
export class ContentGenerationService {
  async generateMarketingCampaign(
    campaign: CampaignBrief,
    audience: Audience
  ) {
    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 8192,
      messages: [{
        role: 'user',
        content: `Create marketing campaign:

Brief: ${JSON.stringify(campaign)}
Target Audience: ${JSON.stringify(audience)}

Generate:
1. Campaign theme and messaging
2. 3 email variations (A/B/C test)
3. 5 social media posts (Facebook/Instagram)
4. 3 WhatsApp message templates
5. SMS campaign (160 characters)
6. Landing page copy

All in Arabic and French. Culturally appropriate for Algeria.
Include CTAs and tracking parameters.`
      }],
    });

    return this.parseMarketingContent(message);
  }

  async generateRouteDescription(route: Route) {
    const attractions = await this.getDestinationAttractions(route.destination);
    const reviews = await this.getRouteReviews(route.id);

    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 2048,
      messages: [{
        role: 'user',
        content: `Write engaging route description:

Route: ${route.origin} → ${route.destination}
Distance: ${route.distance}km
Duration: ${route.duration}min
Attractions: ${JSON.stringify(attractions)}
Reviews: ${JSON.stringify(reviews)}

Create:
- SEO-optimized title (60 chars)
- Meta description (160 chars)
- Hero description (2-3 sentences)
- Full description (200-300 words)
- Highlights (5 bullet points)

In Arabic and French. Focus on experience, not just logistics.`
      }],
    });

    return this.parseRouteContent(message);
  }
}
```

## Phase 4: Enterprise AI (Months 19-24)

### 4.1 Business Intelligence & Analytics

**Features**:
- Natural language query interface for analytics
- Automated insights generation
- Competitive intelligence
- Market opportunity identification

**Implementation**:

```typescript
// Business Intelligence Service
export class BusinessIntelligenceService {
  async answerBusinessQuestion(question: string, context: BusinessContext) {
    // Get relevant data
    const data = await this.fetchRelevantData(question);

    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 4096,
      messages: [{
        role: 'user',
        content: `Answer this business question using data:

Question: ${question}
Data: ${JSON.stringify(data)}
Context: ${JSON.stringify(context)}

Provide:
- Direct answer
- Key insights
- Visualizations (describe charts to create)
- Actionable recommendations
- Follow-up questions to consider

Be specific with numbers and trends.`
      }],
    });

    return this.formatBIResponse(message);
  }

  async generateExecutiveReport(period: DateRange) {
    const metrics = await this.aggregateMetrics(period);

    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 8192,
      messages: [{
        role: 'user',
        content: `Create executive summary report:

Period: ${period.start} to ${period.end}
Metrics: ${JSON.stringify(metrics)}

Include:
1. Executive Summary (5 key points)
2. Financial Performance
3. Operational Metrics
4. Customer Insights
5. Market Position
6. Risks & Opportunities
7. Strategic Recommendations

Professional, data-driven, actionable.`
      }],
    });

    return this.formatExecutiveReport(message);
  }
}
```

### 4.2 Partner Performance Optimization

**Features**:
- Partner coaching
- Performance insights
- Route optimization suggestions
- Revenue optimization

**Implementation**:

```typescript
// Partner Intelligence Service
export class PartnerIntelligenceService {
  async generatePartnerInsights(partnerId: number) {
    const performance = await this.getPartnerPerformance(partnerId);
    const benchmarks = await this.getIndustryBenchmarks();

    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 4096,
      messages: [{
        role: 'user',
        content: `Analyze partner performance and provide coaching:

Partner: ${partnerId}
Performance: ${JSON.stringify(performance)}
Benchmarks: ${JSON.stringify(benchmarks)}

Provide:
1. Performance score (0-100)
2. Strengths (top 3)
3. Improvement areas (top 3)
4. Specific recommendations (actionable)
5. Revenue optimization opportunities
6. Customer satisfaction insights

Be constructive and specific.`
      }],
    });

    return JSON.parse(
      message.content[0].type === 'text' 
        ? message.content[0].text 
        : '{}'
    );
  }
}
```

### 4.3 Automated Operations

**Features**:
- Auto-scheduling
- Incident response
- Crisis management
- Resource allocation

### 4.4 Multimodal AI (Vision + Text)

**Use Cases**:
- Bus condition assessment from photos
- Document processing (IDs, licenses)
- Receipt scanning
- Social media monitoring

**Implementation**:

```typescript
// Vision AI Service
export class VisionAIService {
  async assessBusCondition(images: string[]) {
    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 2048,
      messages: [{
        role: 'user',
        content: [
          {
            type: 'text',
            text: `Assess bus condition from these images. Look for:
- Exterior damage, scratches, dents
- Interior cleanliness and wear
- Seat condition
- Safety equipment visibility
- Branding consistency

Provide condition score (0-100) and specific issues.`,
          },
          ...images.map(base64 => ({
            type: 'image',
            source: {
              type: 'base64',
              media_type: 'image/jpeg',
              data: base64,
            },
          })),
        ],
      }],
    });

    return JSON.parse(
      message.content[0].type === 'text' 
        ? message.content[0].text 
        : '{}'
    );
  }

  async verifyDriverLicense(imageBase64: string) {
    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1024,
      messages: [{
        role: 'user',
        content: [
          {
            type: 'text',
            text: `Extract information from this driver's license:
- Full name
- License number
- Issue date
- Expiry date
- Category/class
- Any restrictions

Validate if Category D (commercial passenger) is present.
Return structured JSON.`,
          },
          {
            type: 'image',
            source: {
              type: 'base64',
              media_type: 'image/jpeg',
              data: imageBase64,
            },
          },
        ],
      }],
    });

    return JSON.parse(
      message.content[0].type === 'text' 
        ? message.content[0].text 
        : '{}'
    );
  }
}
```

## AI Infrastructure Setup

### Model Selection

**Primary**: Claude Sonnet 4 (via Anthropic API)
- Best reasoning capabilities
- Multilingual (Arabic, French, English)
- Function calling for tool use
- Vision capabilities

**Secondary**: Local Models (for cost optimization)
- Use Ollama for simple tasks
- Classification, embeddings
- Sentiment analysis

### Cost Optimization

```typescript
// Intelligent Model Router
export class ModelRouterService {
  routeRequest(task: AITask): ModelConfig {
    const complexity = this.assessComplexity(task);

    if (complexity.score < 30 && task.type === 'classification') {
      // Use local model
      return { provider: 'ollama', model: 'mistral' };
    }

    if (complexity.score < 50 && !task.requiresLatestKnowledge) {
      // Use Claude Haiku (faster, cheaper)
      return { provider: 'anthropic', model: 'claude-haiku-4-5-20251001' };
    }

    // Use Claude Sonnet for complex tasks
    return { provider: 'anthropic', model: 'claude-sonnet-4-20250514' };
  }

  async executeCached(key: string, task: () => Promise<AIResponse>) {
    // Cache AI responses when appropriate
    const cached = await redis.get(`ai:${key}`);
    if (cached) return JSON.parse(cached);

    const result = await task();
    await redis.set(`ai:${key}`, JSON.stringify(result), 'EX', 3600);
    return result;
  }
}
```

### Monitoring & Observability

```typescript
// AI Monitoring
export class AIMonitoringService {
  async trackRequest(request: AIRequest, response: AIResponse) {
    await analytics.track({
      event: 'ai_request',
      properties: {
        model: request.model,
        task: request.task,
        latency: response.latency,
        tokens: response.usage.total_tokens,
        cost: this.calculateCost(response),
        success: response.success,
      },
    });

    // Alert on anomalies
    if (response.latency > 5000) {
      await alerts.warn('High AI latency', { request, response });
    }

    if (response.usage.total_tokens > 10000) {
      await alerts.info('High token usage', { request, response });
    }
  }

  async generateCostReport(period: DateRange) {
    // Aggregate costs by feature, model, user
    // Identify optimization opportunities
    // Forecast future costs
  }
}
```

## Key Metrics & KPIs

### AI Performance Metrics

**Accuracy**:
- Search intent accuracy: >90%
- Recommendation relevance: >75%
- Chatbot resolution rate: >60%

**Efficiency**:
- Response time: <2 seconds (95th percentile)
- Support ticket deflection: 40-60%
- Booking conversion from AI: +25%

**Cost**:
- AI cost per booking: <$0.10
- ROI on AI features: >300%
- Cost reduction vs human support: 70%

### Business Impact Metrics

**Customer Experience**:
- CSAT improvement: +15%
- Booking time reduction: -40%
- Support response time: -70%

**Revenue**:
- Conversion rate improvement: +20%
- Average booking value: +10%
- Repeat booking rate: +15%

**Operations**:
- Route optimization savings: 15%
- Partner performance improvement: 20%
- Fraud reduction: 80%

## Implementation Timeline

```
Phase 1: Foundation (Months 1-6)
├─ Month 1-2: AI Search & Chatbot MVP
├─ Month 3-4: WhatsApp Integration
├─ Month 5-6: Support Automation

Phase 2: Intelligence (Months 7-12)
├─ Month 7-8: Demand Forecasting
├─ Month 9-10: Personalization Engine
├─ Month 11-12: Route Optimization

Phase 3: Advanced (Months 13-18)
├─ Month 13-14: Voice Assistant
├─ Month 15-16: Predictive Maintenance
├─ Month 17-18: Fraud Detection

Phase 4: Enterprise (Months 19-24)
├─ Month 19-20: Business Intelligence
├─ Month 21-22: Partner Optimization
├─ Month 23-24: Vision AI + Automation
```

## Budget Allocation

**Year 1** (Phases 1-2):
- API costs: $15-20K
- Development: $80-100K
- Infrastructure: $10-15K
- Total: $105-135K

**Year 2** (Phases 3-4):
- API costs: $30-40K
- Development: $60-80K
- Infrastructure: $15-20K
- Total: $105-140K

**ROI Projection**:
- Cost savings (support): $100K+/year
- Revenue increase: $200K+/year
- Total ROI: 200-300%

---

*This AI roadmap transforms the platform from a booking system into an intelligent travel assistant, creating defensible competitive advantages and superior customer experiences.*