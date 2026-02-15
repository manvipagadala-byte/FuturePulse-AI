# FuturePulse Platform - Design Document

## Overview

FuturePulse is a ward-based youth civic tech platform built on a modern web architecture with AI-powered environmental projection capabilities. The system consists of five primary subsystems:

1. **Environmental Data Layer**: Real-time integration with AQI and weather APIs, providing ward-level environmental insights
2. **AI Projection Engine**: Computer vision and generative AI pipeline for creating future scenario visualizations
3. **Civic Action Coordination**: Ward-based chapter management and event organization system
4. **Reputation & Impact System**: Gamified scoring with AI-based contextual impact assessment
5. **Privacy-Preserving Infrastructure**: Optional geo-tagging with strong privacy controls

The platform prioritizes scalability, privacy, and responsible AI usage while delivering an engaging user experience that motivates youth-led climate action.

## Architecture

### System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                              │
│  ┌──────────────────┐              ┌──────────────────┐        │
│  │  Web Application │              │ Mobile App (PWA) │        │
│  │   (React/Next.js)│              │   (React Native) │        │
│  └──────────────────┘              └──────────────────┘        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ HTTPS/REST API
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API Gateway Layer                           │
│              (Authentication, Rate Limiting, Routing)            │
└─────────────────────────────────────────────────────────────────┘
                              │
                ┌─────────────┼─────────────┐
                │             │             │
                ▼             ▼             ▼
┌──────────────────┐ ┌──────────────┐ ┌──────────────────┐
│  Core Services   │ │ AI Services  │ │  Data Services   │
│                  │ │              │ │                  │
│ • Auth Service   │ │ • CV Engine  │ │ • Environmental  │
│ • User Service   │ │ • Gen Model  │ │   Data Fetcher   │
│ • Chapter Mgmt   │ │ • Content    │ │ • Cache Layer    │
│ • Event Service  │ │   Moderator  │ │ • Analytics      │
│ • Notification   │ │ • Impact AI  │ │                  │
└──────────────────┘ └──────────────┘ └──────────────────┘
        │                    │                   │
        └────────────────────┼───────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Data Layer                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  PostgreSQL  │  │    Redis     │  │  S3/Object   │         │
│  │  (Primary DB)│  │   (Cache)    │  │   Storage    │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    External Services                             │
│  • AQI Data APIs (EPA, IQAir, etc.)                             │
│  • Weather APIs (OpenWeather, NOAA)                             │
│  • AI Model APIs (OpenAI, Stability AI, or self-hosted)        │
│  • Email Service (SendGrid, AWS SES)                            │
│  • Push Notification Service (FCM, APNs)                        │
└─────────────────────────────────────────────────────────────────┘
```

### Technology Stack

**Frontend:**
- React with Next.js for web application (SSR for performance)
- React Native for mobile PWA
- TailwindCSS for responsive design
- Chart.js for environmental data visualization
- Leaflet for ward-based mapping

**Backend:**
- Node.js with Express or Fastify for API services
- Python with FastAPI for AI services (better ML library support)
- PostgreSQL for relational data (users, chapters, events, scores)
- Redis for caching and session management
- AWS S3 or equivalent for image storage

**AI/ML:**
- PyTorch or TensorFlow for model serving
- OpenAI CLIP or custom CNN for computer vision
- Stable Diffusion or DALL-E API for generative modeling
- Hugging Face Transformers for text generation (explanations)
- Custom scoring models for impact assessment

**Infrastructure:**
- Docker containers for service isolation
- Kubernetes for orchestration and scaling
- NGINX for load balancing
- CloudFlare for CDN and DDoS protection
- Prometheus + Grafana for monitoring


## Components and Interfaces

### 1. Environmental Data Layer

**Purpose:** Fetch, cache, and serve real-time environmental data at ward granularity.

**Components:**

**EnvironmentalDataFetcher**
- Responsibilities: Poll external APIs, transform data to ward-level, handle failures
- Interface:
  ```typescript
  interface EnvironmentalDataFetcher {
    fetchAQI(wardId: string): Promise<AQIReading>
    fetchHeatIndex(wardId: string): Promise<HeatReading>
    getLastKnownReading(wardId: string, metric: MetricType): Promise<Reading | null>
  }
  
  type AQIReading = {
    wardId: string
    value: number  // 0-500
    category: 'Good' | 'Moderate' | 'Unhealthy for Sensitive' | 'Unhealthy' | 'Very Unhealthy' | 'Hazardous'
    timestamp: Date
    source: string
  }
  
  type HeatReading = {
    wardId: string
    temperature: number  // Celsius
    humidity: number     // Percentage
    heatIndex: number    // Calculated
    category: 'Safe' | 'Caution' | 'Extreme Caution' | 'Danger' | 'Extreme Danger'
    timestamp: Date
    source: string
  }
  ```

**DataCache**
- Responsibilities: Cache readings for 15 minutes, provide fallback data
- Interface:
  ```typescript
  interface DataCache {
    set(key: string, value: Reading, ttl: number): Promise<void>
    get(key: string): Promise<Reading | null>
    invalidate(key: string): Promise<void>
  }
  ```

**DataValidator**
- Responsibilities: Validate incoming data for reasonable ranges
- Interface:
  ```typescript
  interface DataValidator {
    validateAQI(value: number): boolean  // 0-500 range
    validateTemperature(value: number): boolean  // -50 to 60°C
    validateHumidity(value: number): boolean  // 0-100%
  }
  ```

**Integration Strategy:**
- Primary AQI source: Government EPA API or IQAir API
- Fallback AQI source: OpenAQ API
- Weather data: OpenWeatherMap API or NOAA API
- Polling interval: Every 10 minutes (cache for 15 minutes)
- Retry logic: 3 attempts with exponential backoff (1s, 2s, 4s)
- Ward mapping: Maintain a database table mapping ward IDs to lat/lon bounding boxes

### 2. AI Projection Engine

**Purpose:** Analyze uploaded images and generate two future scenarios (sustainable vs degradation) with explanations.

**Components:**

**ImageProcessor**
- Responsibilities: Validate uploads, extract metadata, strip geo-tags if requested
- Interface:
  ```typescript
  interface ImageProcessor {
    validateImage(file: File): ValidationResult
    extractMetadata(file: File): ImageMetadata
    stripGeoTags(file: File): Promise<File>
    resizeForProcessing(file: File, maxDimension: number): Promise<File>
  }
  
  type ImageMetadata = {
    format: string
    dimensions: { width: number, height: number }
    geoTag?: { lat: number, lon: number }
    timestamp?: Date
  }
  
  type ValidationResult = {
    valid: boolean
    errors: string[]
  }
  ```

**ComputerVisionAnalyzer**
- Responsibilities: Identify environmental elements in images
- Interface:
  ```typescript
  interface ComputerVisionAnalyzer {
    analyzeImage(imageBuffer: Buffer): Promise<EnvironmentalAnalysis>
  }
  
  type EnvironmentalAnalysis = {
    vegetation: {
      density: number  // 0-1
      types: string[]  // ['trees', 'grass', 'shrubs']
      health: number   // 0-1
    }
    infrastructure: {
      types: string[]  // ['buildings', 'roads', 'parking']
      density: number  // 0-1
    }
    waterBodies: {
      present: boolean
      types: string[]  // ['river', 'pond', 'canal']
      condition: number  // 0-1 (cleanliness estimate)
    }
    pollutionIndicators: {
      visible: boolean
      types: string[]  // ['litter', 'smog', 'industrial']
      severity: number  // 0-1
    }
    confidence: number  // 0-1 overall confidence
  }
  ```

**Implementation:** Use a pre-trained model like CLIP or a fine-tuned ResNet/EfficientNet for scene understanding. The model should be trained or fine-tuned on urban environmental imagery.

**ScenarioGenerator**
- Responsibilities: Generate sustainable and degradation future scenarios
- Interface:
  ```typescript
  interface ScenarioGenerator {
    generateScenarios(
      analysis: EnvironmentalAnalysis,
      wardContext: WardContext
    ): Promise<ScenarioPair>
  }
  
  type WardContext = {
    wardId: string
    currentAQI: number
    aqiTrend: 'improving' | 'stable' | 'degrading'
    temperatureTrend: 'cooling' | 'stable' | 'warming'
    recentCivicActions: number
  }
  
  type ScenarioPair = {
    sustainable: Scenario
    degradation: Scenario
    generationTime: number  // milliseconds
  }
  
  type Scenario = {
    imageUrl: string
    explanation: string
    confidence: number
    environmentalFactors: string[]
  }
  ```

**Implementation:** 
- Use Stable Diffusion or DALL-E API for image generation
- Construct prompts based on environmental analysis and ward context
- Sustainable prompt template: "A vibrant future of [location description] with increased greenery, clean air, thriving vegetation, solar panels, bike lanes, community gardens, clear skies, [positive elements from analysis]"
- Degradation prompt template: "A degraded future of [location description] with reduced vegetation, polluted air, cracked infrastructure, litter, smog, heat stress, [negative elements from analysis]"
- Use ControlNet or similar for maintaining structural consistency with original image
- Generate explanations using GPT-4 or similar LLM based on the analysis and generation parameters

**ExplanationGenerator**
- Responsibilities: Create human-readable explanations for scenarios
- Interface:
  ```typescript
  interface ExplanationGenerator {
    generateExplanation(
      scenario: 'sustainable' | 'degradation',
      analysis: EnvironmentalAnalysis,
      wardContext: WardContext
    ): Promise<string>
  }
  ```

**Implementation:** Use GPT-4 or similar to generate 2-3 paragraph explanations that:
- Reference specific environmental elements from the analysis
- Connect to ward-level trends (AQI, temperature)
- Explain the causal factors leading to the projected future
- Suggest actionable steps (for sustainable) or consequences (for degradation)

### 3. Civic Action Coordination System

**Purpose:** Manage ward-based youth chapters and coordinate environmental events.

**Components:**

**ChapterManager**
- Responsibilities: Handle chapter creation, verification, and management
- Interface:
  ```typescript
  interface ChapterManager {
    createChapterRequest(request: ChapterRequest): Promise<string>  // Returns requestId
    verifyChapter(requestId: string, adminId: string): Promise<Chapter>
    getChapterByWard(wardId: string): Promise<Chapter | null>
    updateChapter(chapterId: string, updates: Partial<Chapter>): Promise<Chapter>
  }
  
  type ChapterRequest = {
    wardId: string
    chapterName: string
    organizerUserId: string
    organizerContact: string
    description: string
  }
  
  type Chapter = {
    id: string
    wardId: string
    name: string
    organizerId: string
    status: 'pending' | 'active' | 'suspended'
    createdAt: Date
    verifiedAt?: Date
    verifiedBy?: string
  }
  ```

**EventManager**
- Responsibilities: Create, manage, and track civic action events
- Interface:
  ```typescript
  interface EventManager {
    createEvent(event: EventCreate): Promise<CivicEvent>
    registerUser(eventId: string, userId: string): Promise<Registration>
    markEventComplete(eventId: string, attendees: string[]): Promise<void>
    getUpcomingEvents(wardId: string): Promise<CivicEvent[]>
    getUserEvents(userId: string): Promise<CivicEvent[]>
  }
  
  type EventCreate = {
    chapterId: string
    type: 'cleanup' | 'tree_plantation'
    title: string
    description: string
    date: Date
    duration: number  // minutes
    location: string
    capacity: number
    wardId: string
  }
  
  type CivicEvent = EventCreate & {
    id: string
    registeredCount: number
    status: 'upcoming' | 'ongoing' | 'completed' | 'cancelled'
    createdAt: Date
  }
  
  type Registration = {
    eventId: string
    userId: string
    registeredAt: Date
    attended: boolean
  }
  ```

**RegistrationManager**
- Responsibilities: Handle event registrations and capacity management
- Interface:
  ```typescript
  interface RegistrationManager {
    checkCapacity(eventId: string): Promise<boolean>
    registerUser(eventId: string, userId: string): Promise<Registration>
    unregisterUser(eventId: string, userId: string): Promise<void>
    getRegistrations(eventId: string): Promise<Registration[]>
  }
  ```

**Implementation Notes:**
- Use database transactions for registration to prevent race conditions
- Implement optimistic locking for capacity checks
- Send notifications asynchronously after successful registration


### 4. Reputation & Impact System

**Purpose:** Track individual contributions and ward-level engagement with AI-based contextual scoring.

**Components:**

**ReputationTracker**
- Responsibilities: Award and track individual reputation points
- Interface:
  ```typescript
  interface ReputationTracker {
    awardPoints(userId: string, eventId: string, eventType: EventType): Promise<number>
    getUserReputation(userId: string): Promise<UserReputation>
    getWardLeaderboard(wardId: string, limit: number): Promise<LeaderboardEntry[]>
  }
  
  type EventType = 'cleanup' | 'tree_plantation'
  
  type UserReputation = {
    userId: string
    totalPoints: number
    wardRank: number
    eventsCompleted: number
    lastActivity: Date
  }
  
  type LeaderboardEntry = {
    userId: string
    displayName: string
    points: number
    rank: number
  }
  ```

**CivicScoreCalculator**
- Responsibilities: Calculate ward-level civic scores with AI-based contextual weighting
- Interface:
  ```typescript
  interface CivicScoreCalculator {
    calculateWardScore(wardId: string): Promise<WardScore>
    getAllWardScores(): Promise<WardScore[]>
    getWardRanking(wardId: string): Promise<number>
  }
  
  type WardScore = {
    wardId: string
    score: number
    rank: number
    eventsLast90Days: number
    participantsLast90Days: number
    environmentalImpact: ImpactMetrics
    lastUpdated: Date
  }
  
  type ImpactMetrics = {
    areaCleaned: number      // square meters
    treesPlanted: number
    participantHours: number
    weightedImpact: number   // AI-calculated contextual score
  }
  ```

**ImpactAI**
- Responsibilities: Apply contextual weighting to civic actions based on environmental factors
- Interface:
  ```typescript
  interface ImpactAI {
    calculateEventImpact(event: CompletedEvent): Promise<number>
    explainImpactScore(eventId: string): Promise<string>
  }
  
  type CompletedEvent = {
    eventId: string
    type: EventType
    wardId: string
    attendees: number
    duration: number
    metadata: {
      areaCleaned?: number
      treesPlanted?: number
      wasteCollected?: number  // kg
    }
    wardContext: {
      baselineAQI: number
      populationDensity: number
      existingGreenCover: number
    }
  }
  ```

**Implementation:**
- Base points: 10 for cleanup, 15 for tree plantation
- AI contextual multiplier: 0.5x to 2.0x based on:
  - Ward environmental baseline (higher impact in more degraded areas)
  - Event scale (participants, area, duration)
  - Seasonal factors (tree planting in monsoon = higher survival rate)
  - Historical engagement (bonus for consistent participation)
- Use a simple neural network or gradient boosting model trained on historical event data
- Model inputs: event features + ward features → impact multiplier
- Update ward scores daily at midnight using batch processing

### 5. Privacy-Preserving Infrastructure

**Purpose:** Provide optional geo-tagging with strong privacy controls.

**Components:**

**GeoTagManager**
- Responsibilities: Handle location data with privacy controls
- Interface:
  ```typescript
  interface GeoTagManager {
    extractGeoTag(image: File): Promise<GeoTag | null>
    stripGeoTag(image: File): Promise<File>
    wardFromCoordinates(lat: number, lon: number): Promise<string>
    aggregateToWardLevel(geoTag: GeoTag): WardLocation
  }
  
  type GeoTag = {
    lat: number
    lon: number
    accuracy: number  // meters
  }
  
  type WardLocation = {
    wardId: string
    wardName: string
    // No precise coordinates
  }
  ```

**PrivacyController**
- Responsibilities: Manage user privacy preferences and enforce controls
- Interface:
  ```typescript
  interface PrivacyController {
    getUserPreferences(userId: string): Promise<PrivacyPreferences>
    updatePreferences(userId: string, prefs: Partial<PrivacyPreferences>): Promise<void>
    shouldStripGeoTag(userId: string): Promise<boolean>
    canShareLocation(userId: string): Promise<boolean>
  }
  
  type PrivacyPreferences = {
    userId: string
    shareLocation: boolean
    autoStripGeoTags: boolean
    showInLeaderboards: boolean
    allowNotifications: NotificationPreferences
  }
  
  type NotificationPreferences = {
    environmental: boolean
    civicEvents: boolean
    achievements: boolean
  }
  ```

**LocationEncryption**
- Responsibilities: Encrypt location data at rest
- Interface:
  ```typescript
  interface LocationEncryption {
    encrypt(geoTag: GeoTag): Promise<string>
    decrypt(encrypted: string): Promise<GeoTag>
  }
  ```

**Implementation:**
- Store location data in separate table from user identity
- Use AES-256 encryption for any stored coordinates
- Never expose precise coordinates in API responses
- Implement data retention policy: delete location data after 90 days
- Provide user data export in JSON format
- Implement account deletion with cascading data removal

### 6. Authentication & User Management

**Purpose:** Secure user authentication and profile management.

**Components:**

**AuthService**
- Responsibilities: Handle registration, login, password management
- Interface:
  ```typescript
  interface AuthService {
    register(credentials: RegisterRequest): Promise<User>
    login(email: string, password: string): Promise<AuthToken>
    logout(token: string): Promise<void>
    refreshToken(refreshToken: string): Promise<AuthToken>
    requestPasswordReset(email: string): Promise<void>
    resetPassword(token: string, newPassword: string): Promise<void>
  }
  
  type RegisterRequest = {
    email: string
    password: string
    displayName: string
    wardId: string
  }
  
  type AuthToken = {
    accessToken: string
    refreshToken: string
    expiresIn: number  // seconds
  }
  
  type User = {
    id: string
    email: string
    displayName: string
    wardId: string
    verified: boolean
    createdAt: Date
  }
  ```

**SessionManager**
- Responsibilities: Manage user sessions and tokens
- Interface:
  ```typescript
  interface SessionManager {
    createSession(userId: string): Promise<string>
    validateSession(token: string): Promise<User | null>
    invalidateSession(token: string): Promise<void>
    cleanupExpiredSessions(): Promise<number>
  }
  ```

**PasswordManager**
- Responsibilities: Hash and verify passwords securely
- Interface:
  ```typescript
  interface PasswordManager {
    hash(password: string): Promise<string>
    verify(password: string, hash: string): Promise<boolean>
    validateStrength(password: string): ValidationResult
  }
  ```

**Implementation:**
- Use bcrypt with cost factor 12 for password hashing
- JWT tokens for session management (7-day expiry)
- Refresh tokens stored in Redis with 30-day expiry
- Rate limiting: 5 failed login attempts per 15 minutes per IP
- Email verification required before full access
- Password requirements: min 8 chars, mixed case, numbers

### 7. Notification System

**Purpose:** Send timely notifications for environmental alerts and civic opportunities.

**Components:**

**NotificationDispatcher**
- Responsibilities: Route notifications to appropriate channels
- Interface:
  ```typescript
  interface NotificationDispatcher {
    sendNotification(notification: Notification): Promise<void>
    sendBulkNotifications(notifications: Notification[]): Promise<void>
  }
  
  type Notification = {
    userId: string
    type: NotificationType
    channel: 'push' | 'email' | 'both'
    title: string
    body: string
    data?: Record<string, any>
    priority: 'low' | 'normal' | 'high'
  }
  
  type NotificationType = 
    | 'environmental_alert'
    | 'civic_event'
    | 'achievement'
    | 'reminder'
    | 'system'
  ```

**EnvironmentalAlertMonitor**
- Responsibilities: Monitor environmental data and trigger alerts
- Interface:
  ```typescript
  interface EnvironmentalAlertMonitor {
    checkThresholds(wardId: string): Promise<Alert[]>
    notifyWardUsers(wardId: string, alert: Alert): Promise<void>
  }
  
  type Alert = {
    wardId: string
    type: 'aqi' | 'heat'
    severity: 'moderate' | 'high' | 'extreme'
    value: number
    message: string
  }
  ```

**Implementation:**
- Use Firebase Cloud Messaging (FCM) for push notifications
- Use SendGrid or AWS SES for email notifications
- Background job checks environmental thresholds every 5 minutes
- Respect user notification preferences (stored in PrivacyPreferences)
- Implement notification deduplication (don't spam same alert)
- Queue notifications in Redis for reliable delivery

### 8. Content Moderation

**Purpose:** Ensure uploaded images are appropriate and safe.

**Components:**

**ContentModerator**
- Responsibilities: Screen uploaded images for inappropriate content
- Interface:
  ```typescript
  interface ContentModerator {
    moderateImage(imageBuffer: Buffer): Promise<ModerationResult>
    flagForReview(imageId: string, reason: string): Promise<void>
  }
  
  type ModerationResult = {
    approved: boolean
    confidence: number
    flags: string[]  // ['violence', 'explicit', 'hate_symbols']
    requiresHumanReview: boolean
  }
  ```

**Implementation:**
- Use AWS Rekognition, Google Cloud Vision, or similar for automated moderation
- Reject images with confidence > 0.8 for inappropriate content
- Flag images with confidence 0.5-0.8 for human review
- Maintain moderation queue for administrators
- Log all moderation decisions for audit


## Data Models

### Database Schema

**Users Table**
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  display_name VARCHAR(100) NOT NULL,
  ward_id VARCHAR(50) NOT NULL,
  verified BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  FOREIGN KEY (ward_id) REFERENCES wards(id)
);

CREATE INDEX idx_users_ward ON users(ward_id);
CREATE INDEX idx_users_email ON users(email);
```

**Wards Table**
```sql
CREATE TABLE wards (
  id VARCHAR(50) PRIMARY KEY,
  name VARCHAR(200) NOT NULL,
  city VARCHAR(100) NOT NULL,
  state VARCHAR(100) NOT NULL,
  country VARCHAR(100) NOT NULL,
  bounding_box JSONB NOT NULL,  -- {north, south, east, west}
  population INTEGER,
  area_sq_km DECIMAL(10, 2),
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_wards_city ON wards(city);
```

**Environmental_Readings Table**
```sql
CREATE TABLE environmental_readings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ward_id VARCHAR(50) NOT NULL,
  metric_type VARCHAR(20) NOT NULL,  -- 'aqi' or 'heat'
  value DECIMAL(10, 2) NOT NULL,
  category VARCHAR(50) NOT NULL,
  source VARCHAR(100) NOT NULL,
  timestamp TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  FOREIGN KEY (ward_id) REFERENCES wards(id)
);

CREATE INDEX idx_readings_ward_time ON environmental_readings(ward_id, timestamp DESC);
CREATE INDEX idx_readings_type ON environmental_readings(metric_type);
```

**Chapters Table**
```sql
CREATE TABLE chapters (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ward_id VARCHAR(50) UNIQUE NOT NULL,
  name VARCHAR(200) NOT NULL,
  organizer_id UUID NOT NULL,
  organizer_contact VARCHAR(255) NOT NULL,
  description TEXT,
  status VARCHAR(20) DEFAULT 'pending',  -- 'pending', 'active', 'suspended'
  created_at TIMESTAMP DEFAULT NOW(),
  verified_at TIMESTAMP,
  verified_by UUID,
  FOREIGN KEY (ward_id) REFERENCES wards(id),
  FOREIGN KEY (organizer_id) REFERENCES users(id),
  FOREIGN KEY (verified_by) REFERENCES users(id)
);

CREATE INDEX idx_chapters_ward ON chapters(ward_id);
CREATE INDEX idx_chapters_status ON chapters(status);
```

**Events Table**
```sql
CREATE TABLE events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  chapter_id UUID NOT NULL,
  ward_id VARCHAR(50) NOT NULL,
  type VARCHAR(20) NOT NULL,  -- 'cleanup', 'tree_plantation'
  title VARCHAR(200) NOT NULL,
  description TEXT,
  event_date TIMESTAMP NOT NULL,
  duration INTEGER NOT NULL,  -- minutes
  location VARCHAR(500) NOT NULL,
  capacity INTEGER NOT NULL,
  registered_count INTEGER DEFAULT 0,
  status VARCHAR(20) DEFAULT 'upcoming',  -- 'upcoming', 'ongoing', 'completed', 'cancelled'
  created_at TIMESTAMP DEFAULT NOW(),
  completed_at TIMESTAMP,
  FOREIGN KEY (chapter_id) REFERENCES chapters(id),
  FOREIGN KEY (ward_id) REFERENCES wards(id),
  CHECK (registered_count <= capacity)
);

CREATE INDEX idx_events_ward_date ON events(ward_id, event_date);
CREATE INDEX idx_events_status ON events(status);
```

**Registrations Table**
```sql
CREATE TABLE registrations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_id UUID NOT NULL,
  user_id UUID NOT NULL,
  registered_at TIMESTAMP DEFAULT NOW(),
  attended BOOLEAN DEFAULT FALSE,
  FOREIGN KEY (event_id) REFERENCES events(id) ON DELETE CASCADE,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  UNIQUE(event_id, user_id)
);

CREATE INDEX idx_registrations_event ON registrations(event_id);
CREATE INDEX idx_registrations_user ON registrations(user_id);
```

**Reputation Table**
```sql
CREATE TABLE reputation (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL,
  event_id UUID NOT NULL,
  points INTEGER NOT NULL,
  awarded_at TIMESTAMP DEFAULT NOW(),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  FOREIGN KEY (event_id) REFERENCES events(id)
);

CREATE INDEX idx_reputation_user ON reputation(user_id);
```

**Ward_Scores Table**
```sql
CREATE TABLE ward_scores (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ward_id VARCHAR(50) NOT NULL,
  score DECIMAL(10, 2) NOT NULL,
  rank INTEGER,
  events_count INTEGER DEFAULT 0,
  participants_count INTEGER DEFAULT 0,
  area_cleaned DECIMAL(10, 2) DEFAULT 0,  -- square meters
  trees_planted INTEGER DEFAULT 0,
  participant_hours INTEGER DEFAULT 0,
  weighted_impact DECIMAL(10, 2) DEFAULT 0,
  calculation_date DATE NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  FOREIGN KEY (ward_id) REFERENCES wards(id),
  UNIQUE(ward_id, calculation_date)
);

CREATE INDEX idx_ward_scores_date ON ward_scores(calculation_date DESC);
CREATE INDEX idx_ward_scores_rank ON ward_scores(rank);
```

**Projections Table**
```sql
CREATE TABLE projections (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL,
  ward_id VARCHAR(50),
  original_image_url VARCHAR(500),
  sustainable_image_url VARCHAR(500) NOT NULL,
  sustainable_explanation TEXT NOT NULL,
  degradation_image_url VARCHAR(500) NOT NULL,
  degradation_explanation TEXT NOT NULL,
  environmental_analysis JSONB NOT NULL,
  confidence DECIMAL(3, 2),
  generation_time INTEGER,  -- milliseconds
  created_at TIMESTAMP DEFAULT NOW(),
  FOREIGN KEY (user_id) REFERENCES users(id),
  FOREIGN KEY (ward_id) REFERENCES wards(id)
);

CREATE INDEX idx_projections_user ON projections(user_id);
CREATE INDEX idx_projections_ward ON projections(ward_id);
```

**Privacy_Preferences Table**
```sql
CREATE TABLE privacy_preferences (
  user_id UUID PRIMARY KEY,
  share_location BOOLEAN DEFAULT FALSE,
  auto_strip_geotags BOOLEAN DEFAULT TRUE,
  show_in_leaderboards BOOLEAN DEFAULT TRUE,
  notify_environmental BOOLEAN DEFAULT TRUE,
  notify_civic_events BOOLEAN DEFAULT TRUE,
  notify_achievements BOOLEAN DEFAULT TRUE,
  updated_at TIMESTAMP DEFAULT NOW(),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

**Moderation_Queue Table**
```sql
CREATE TABLE moderation_queue (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  image_url VARCHAR(500) NOT NULL,
  user_id UUID NOT NULL,
  moderation_result JSONB NOT NULL,
  status VARCHAR(20) DEFAULT 'pending',  -- 'pending', 'approved', 'rejected'
  reviewed_by UUID,
  reviewed_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  FOREIGN KEY (user_id) REFERENCES users(id),
  FOREIGN KEY (reviewed_by) REFERENCES users(id)
);

CREATE INDEX idx_moderation_status ON moderation_queue(status);
```

### API Endpoints

**Authentication**
- `POST /api/auth/register` - Register new user
- `POST /api/auth/login` - Login user
- `POST /api/auth/logout` - Logout user
- `POST /api/auth/refresh` - Refresh access token
- `POST /api/auth/verify-email` - Verify email address
- `POST /api/auth/request-reset` - Request password reset
- `POST /api/auth/reset-password` - Reset password

**Environmental Data**
- `GET /api/environmental/ward/:wardId` - Get current readings for ward
- `GET /api/environmental/ward/:wardId/history` - Get historical readings
- `GET /api/environmental/alerts/:wardId` - Get active alerts for ward

**Projections**
- `POST /api/projections/upload` - Upload image and generate scenarios
- `GET /api/projections/:id` - Get projection by ID
- `GET /api/projections/user/:userId` - Get user's projections
- `DELETE /api/projections/:id` - Delete projection

**Chapters**
- `POST /api/chapters/request` - Request chapter creation
- `GET /api/chapters/ward/:wardId` - Get chapter for ward
- `PUT /api/chapters/:id` - Update chapter (organizer only)
- `POST /api/chapters/:id/verify` - Verify chapter (admin only)

**Events**
- `POST /api/events` - Create event (chapter organizer only)
- `GET /api/events/ward/:wardId` - Get upcoming events for ward
- `GET /api/events/:id` - Get event details
- `POST /api/events/:id/register` - Register for event
- `DELETE /api/events/:id/register` - Unregister from event
- `PUT /api/events/:id/complete` - Mark event complete (organizer only)

**Reputation & Scores**
- `GET /api/reputation/user/:userId` - Get user reputation
- `GET /api/reputation/leaderboard/ward/:wardId` - Get ward leaderboard
- `GET /api/scores/ward/:wardId` - Get ward score
- `GET /api/scores/leaderboard` - Get city-wide ward leaderboard

**User Profile**
- `GET /api/users/:id` - Get user profile
- `PUT /api/users/:id` - Update user profile
- `GET /api/users/:id/events` - Get user's registered events
- `DELETE /api/users/:id` - Delete user account

**Privacy**
- `GET /api/privacy/preferences` - Get privacy preferences
- `PUT /api/privacy/preferences` - Update privacy preferences
- `GET /api/privacy/export` - Export user data
- `POST /api/privacy/delete-data` - Request data deletion


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Environmental Data Properties

**Property 1: Timely Environmental Data Display**
*For any* ward and any environmental metric type (AQI or Heat Index), when current data is available, the dashboard should display that data within 5 minutes of its availability.
**Validates: Requirements 1.1, 1.2**

**Property 2: Automatic Data Refresh**
*For any* dashboard session, environmental readings should be automatically updated every 15 minutes without requiring user action.
**Validates: Requirements 1.3**

**Property 3: Fallback to Last Known Reading**
*For any* ward where current environmental data is unavailable, the dashboard should display the most recent cached reading along with a timestamp indicating when that reading was captured.
**Validates: Requirements 1.4**

**Property 4: Correct Environmental Categorization**
*For any* environmental reading (AQI or Heat Index), the system should assign the correct category and color code according to EPA (for AQI) or NOAA (for Heat Index) standards.
**Validates: Requirements 1.5, 1.6**

**Property 5: Explanatory Tooltips**
*For any* environmental metric displayed on the dashboard, hovering over it should reveal a tooltip containing health implication information.
**Validates: Requirements 1.7**

### AI Projection Engine Properties

**Property 6: Image Upload Validation**
*For any* uploaded file, the system should accept it if and only if it is in a supported format (JPEG, PNG, HEIC) and is 10MB or smaller.
**Validates: Requirements 2.1**

**Property 7: Geo-Tag Extraction and Ward Mapping**
*For any* geo-tagged image, the system should correctly extract the coordinates and map them to the appropriate ward based on the ward's bounding box.
**Validates: Requirements 2.2**

**Property 8: Manual Ward Selection for Non-Geo-Tagged Images**
*For any* image without geo-tag metadata, the system should allow the user to manually select a ward before processing.
**Validates: Requirements 2.3**

**Property 9: Dual Scenario Generation**
*For any* valid uploaded image, the projection engine should generate exactly two scenarios within 60 seconds: one sustainable scenario and one degradation scenario.
**Validates: Requirements 2.4, 2.5, 2.6**

**Property 10: Scenario Explanations**
*For any* generated scenario (sustainable or degradation), the system should provide a non-empty text explanation describing the environmental factors depicted in the projection.
**Validates: Requirements 2.7**

**Property 11: Computer Vision Analysis**
*For any* uploaded image, the computer vision analyzer should return a structured EnvironmentalAnalysis object containing vegetation, infrastructure, water bodies, and pollution indicator assessments.
**Validates: Requirements 2.8**

**Property 12: Generative Model Output**
*For any* EnvironmentalAnalysis, the generative model should produce valid image URLs for both sustainable and degradation scenarios.
**Validates: Requirements 2.9**

**Property 13: Ward Context Integration**
*For any* scenario generation request, the projection engine should incorporate ward-specific environmental data (AQI trends, temperature trends) into the generation logic.
**Validates: Requirements 2.10**

### Chapter Management Properties

**Property 14: Chapter Creation Validation**
*For any* chapter creation request, the system should reject it if any required field (ward ID, chapter name, organizer contact) is missing.
**Validates: Requirements 3.1**

**Property 15: Chapter Verification Queue**
*For any* submitted chapter creation request, the system should add it to the verification queue with status "pending" until an administrator reviews it.
**Validates: Requirements 3.2**

**Property 16: Chapter Activation**
*For any* chapter in "pending" status, when an administrator approves it, the system should change its status to "active" and queue a notification to the organizer.
**Validates: Requirements 3.3**

**Property 17: Event Creation Permissions**
*For any* user, they should be able to create civic action events if and only if they are the organizer of an active chapter.
**Validates: Requirements 3.4**

**Property 18: Event Creation Validation**
*For any* event creation request, the system should reject it if any required field (type, date, time, location, capacity) is missing.
**Validates: Requirements 3.5**

**Property 19: Ward-Based Event Visibility**
*For any* civic action event, it should be visible to all users in the event's ward and not visible to users in other wards.
**Validates: Requirements 3.6**

**Property 20: Registration Capacity Decrement**
*For any* event with available capacity, when a user successfully registers, the available capacity should decrease by exactly 1.
**Validates: Requirements 3.7**

**Property 21: Capacity Enforcement**
*For any* event at full capacity (registered_count == capacity), additional registration attempts should be rejected.
**Validates: Requirements 3.8**

**Property 22: Event Completion**
*For any* event marked as complete by its organizer, the system should update the event status to "completed" and record attendance for confirmed participants.
**Validates: Requirements 3.9**

**Property 23: One Chapter Per Ward**
*For any* ward that already has an active chapter, attempts to create or activate a second chapter for that ward should be rejected.
**Validates: Requirements 3.10**

### Reputation and Impact Properties

**Property 24: Reputation Points Award**
*For any* user who completes a civic action event, the system should award reputation points based on the event type (10 for cleanup, 15 for tree plantation).
**Validates: Requirements 4.1**

**Property 25: Immediate Profile Update**
*For any* reputation points award, the user's total reputation points should be updated immediately and reflected in their profile.
**Validates: Requirements 4.4**

**Property 26: 90-Day Civic Score Window**
*For any* ward civic score calculation, only events completed within the past 90 days should be included in the score.
**Validates: Requirements 4.5**

**Property 27: Impact Dashboard Content**
*For any* user viewing their impact dashboard, the response should include their total reputation points, ward ranking, ward civic score, and city-wide ward ranking.
**Validates: Requirements 4.7, 4.8**

**Property 28: Leaderboard Ordering and Limiting**
*For any* leaderboard request (ward-level or city-wide), the system should return at most 10 entries ordered by score/points in descending order.
**Validates: Requirements 4.9, 4.10**

**Property 29: Contextual Impact Scoring**
*For any* civic score calculation, the system should apply AI-based contextual weighting that considers environmental impact indicators (area cleaned, trees planted, participant count, ward baseline conditions).
**Validates: Requirements 4.11**

### Privacy Properties

**Property 30: Functionality Without Location Permissions**
*For any* user who denies location permissions, all platform features should remain accessible through manual ward selection.
**Validates: Requirements 5.2**

**Property 31: Geo-Tag Stripping Option**
*For any* image upload, the system should provide an option to strip geo-tags, and when selected, should permanently remove all location metadata from the image.
**Validates: Requirements 5.3, 5.4**

**Property 32: Ward-Level Location Aggregation**
*For any* public API response containing location information, the system should display only ward-level location (ward name) and never precise coordinates.
**Validates: Requirements 5.5, 5.6**

**Property 33: Privacy Preference Management**
*For any* user, they should be able to view and update their privacy preferences including location sharing, geo-tag stripping, leaderboard visibility, and notification settings.
**Validates: Requirements 5.7**

### Authentication Properties

**Property 34: Registration Field Validation**
*For any* registration request, the system should reject it if any required field (email, password, display name, ward) is missing or if the email format is invalid or the password is weak (< 8 characters, no mixed case, no numbers).
**Validates: Requirements 6.1, 6.2**

**Property 35: Email Verification Flow**
*For any* new registration, the system should queue a verification email, and when the user clicks the verification link, should activate the account and allow login.
**Validates: Requirements 6.3, 6.4**

**Property 36: Session Token Creation**
*For any* valid login attempt (correct email and password), the system should create and return a secure session token.
**Validates: Requirements 6.5**

**Property 37: Session Token Expiry**
*For any* session token older than 7 days, authentication attempts using that token should be rejected.
**Validates: Requirements 6.6**

**Property 38: Password Reset Flow**
*For any* password reset request, the system should generate a reset link valid for 1 hour, and after 1 hour, attempts to use that link should be rejected.
**Validates: Requirements 6.7**

**Property 39: Profile Update**
*For any* authenticated user, they should be able to update their display name, ward, and notification preferences, and those changes should be persisted.
**Validates: Requirements 6.8**

**Property 40: Email Uniqueness**
*For any* registration attempt using an email address that already exists in the system, the registration should be rejected.
**Validates: Requirements 6.9**

### Transparency Properties

**Property 41: AI Transparency Notice**
*For any* AI-generated scenario displayed to a user, the response should include a transparency notice explaining that the content was generated by AI.
**Validates: Requirements 7.1**

### Content Moderation Properties

**Property 42: Automatic Content Moderation**
*For any* uploaded image, the system should pass it through AI-based content moderation before processing.
**Validates: Requirements 8.1**

**Property 43: Inappropriate Content Rejection**
*For any* image flagged as inappropriate by the content moderator (confidence > 0.8), the system should reject the upload and provide a reason to the user.
**Validates: Requirements 8.2**

**Property 44: Content Report Queue**
*For any* user-reported content, the system should add it to the moderation queue for administrator review.
**Validates: Requirements 8.3**

**Property 45: Moderation Action Options**
*For any* content in the moderation queue, administrators should have the ability to remove, warn, or dismiss the report.
**Validates: Requirements 8.4**

**Property 46: Moderation Audit Log**
*For any* moderation action (approve, reject, warn, ban), the system should record it in an audit log with timestamp, administrator ID, and action details.
**Validates: Requirements 8.5**

**Property 47: Automatic Suspension**
*For any* user who receives their third content warning, the system should automatically suspend their account for 7 days.
**Validates: Requirements 8.6**

**Property 48: Administrator Ban**
*For any* user banned by an administrator, all authentication attempts should be rejected.
**Validates: Requirements 8.7**

### Notification Properties

**Property 49: Environmental Threshold Alerts**
*For any* ward where AQI exceeds 150 or Heat Index exceeds 103°F, the system should queue push notifications for all users in that ward who have environmental alerts enabled.
**Validates: Requirements 9.1, 9.2**

**Property 50: New Event Notifications**
*For any* newly created civic action event, the system should queue notifications for all users in the event's ward who have civic event notifications enabled.
**Validates: Requirements 9.3**

**Property 51: Event Reminder Notifications**
*For any* event occurring within 24 hours, the system should queue reminder notifications for all registered users who have civic event notifications enabled.
**Validates: Requirements 9.4**

**Property 52: Achievement Notifications**
*For any* reputation points award, the system should queue an achievement notification for the user if they have achievement notifications enabled.
**Validates: Requirements 9.5**

**Property 53: Notification Preference Management**
*For any* user, they should be able to enable or disable notifications by category (environmental, civic events, achievements).
**Validates: Requirements 9.6**

**Property 54: Notification Preference Enforcement**
*For any* notification, it should only be sent if the user has that notification category enabled in their preferences.
**Validates: Requirements 9.7**

**Property 55: Multi-Channel Notification Support**
*For any* notification, the system should support sending it via both push notification (mobile) and email channels based on user preferences.
**Validates: Requirements 9.8**

### Data Integration Properties

**Property 56: External API Integration**
*For any* ward, the system should successfully fetch environmental data (AQI and weather) from external APIs.
**Validates: Requirements 10.1, 10.2**

**Property 57: API Retry Logic**
*For any* failed external API call, the system should retry up to 3 times with exponential backoff (1s, 2s, 4s) before giving up.
**Validates: Requirements 10.3**

**Property 58: Fallback to Cached Data**
*For any* external API call that fails after all retries, the system should log the failure and serve cached data if available.
**Validates: Requirements 10.4**

**Property 59: Data Caching**
*For any* environmental data fetched from external APIs, the system should cache it for 15 minutes and serve subsequent requests from cache within that window.
**Validates: Requirements 10.5**

**Property 60: Data Range Validation**
*For any* incoming environmental data, the system should validate that values are within reasonable ranges (AQI: 0-500, temperature: -50°C to 60°C, humidity: 0-100%) and reject out-of-range values.
**Validates: Requirements 10.6, 10.7**

### Security Properties

**Property 61: Data Encryption at Rest**
*For any* sensitive data (location coordinates, personal information), the system should encrypt it using AES-256 before storing it in the database.
**Validates: Requirements 13.2**

**Property 62: Password Hashing**
*For any* user password, the system should hash it using bcrypt with a cost factor of at least 12 before storing it.
**Validates: Requirements 13.3**

**Property 63: Rate Limiting**
*For any* IP address, after 5 failed login attempts within 15 minutes, subsequent login attempts should be blocked for the remainder of the 15-minute window.
**Validates: Requirements 13.4**

**Property 64: Input Sanitization**
*For any* user input, the system should sanitize it to prevent SQL injection and XSS attacks before processing or storing it.
**Validates: Requirements 13.5**

**Property 65: Authentication Audit Logging**
*For any* authentication event (login, logout, failed login, password reset), the system should record it in an audit log with timestamp, user ID, IP address, and event type.
**Validates: Requirements 13.7**

**Property 66: Data Export**
*For any* authenticated user, they should be able to request a data export and receive all their personal data in JSON format.
**Validates: Requirements 13.9**

**Property 67: Account Deletion**
*For any* user who requests account deletion, the system should remove all their personal data including profile, registrations, reputation, and uploaded content.
**Validates: Requirements 13.10**


## Error Handling

### Error Categories

**1. Client Errors (4xx)**
- **400 Bad Request**: Invalid input, missing required fields, validation failures
- **401 Unauthorized**: Missing or invalid authentication token
- **403 Forbidden**: Insufficient permissions (e.g., non-organizer trying to create events)
- **404 Not Found**: Resource doesn't exist (user, event, chapter, ward)
- **409 Conflict**: Resource conflict (duplicate email, ward already has chapter, event at capacity)
- **429 Too Many Requests**: Rate limit exceeded

**2. Server Errors (5xx)**
- **500 Internal Server Error**: Unexpected server errors
- **502 Bad Gateway**: External API failures (AQI, weather, AI services)
- **503 Service Unavailable**: Service temporarily down (maintenance, overload)
- **504 Gateway Timeout**: External API timeout

### Error Response Format

All errors should follow a consistent JSON structure:

```typescript
type ErrorResponse = {
  error: {
    code: string           // Machine-readable error code
    message: string        // Human-readable error message
    details?: any          // Additional context (validation errors, etc.)
    timestamp: string      // ISO 8601 timestamp
    requestId: string      // Unique request ID for debugging
  }
}
```

### Error Handling Strategies

**Input Validation Errors**
- Validate all inputs at the API boundary
- Return 400 with specific field-level errors
- Example: `{ code: 'VALIDATION_ERROR', message: 'Invalid email format', details: { field: 'email' } }`

**Authentication Errors**
- Return 401 for missing/invalid tokens
- Return 403 for insufficient permissions
- Never reveal whether an email exists during login failures (security)

**Resource Not Found**
- Return 404 with resource type
- Example: `{ code: 'RESOURCE_NOT_FOUND', message: 'Ward not found', details: { wardId: 'W123' } }`

**External API Failures**
- Implement retry logic with exponential backoff
- Fall back to cached data when available
- Return 502 if external service is down and no cache available
- Log all external API failures for monitoring

**AI Service Failures**
- Retry transient failures (network issues, timeouts)
- Return 503 if AI service is unavailable
- Provide fallback behavior where possible (e.g., skip content moderation in emergency)
- Queue failed projections for retry

**Database Errors**
- Wrap database operations in try-catch blocks
- Use transactions for multi-step operations (registration, event completion)
- Implement connection pooling and retry logic
- Return 500 for unexpected database errors
- Log all database errors with stack traces

**Rate Limiting**
- Implement rate limiting at API gateway level
- Return 429 with Retry-After header
- Different limits for different endpoints (stricter for auth, looser for reads)

**Graceful Degradation**
- If environmental data is unavailable, show last known readings
- If AI projection fails, allow users to retry
- If notifications fail, queue for retry
- Never block core functionality due to auxiliary service failures

### Logging and Monitoring

**Log Levels**
- **ERROR**: All errors that affect user experience
- **WARN**: Degraded functionality, fallback behavior activated
- **INFO**: Important state changes (user registration, event creation)
- **DEBUG**: Detailed execution flow (development only)

**Structured Logging**
```typescript
{
  level: 'ERROR',
  timestamp: '2024-01-15T10:30:00Z',
  requestId: 'req-123',
  userId: 'user-456',
  service: 'projection-engine',
  error: {
    code: 'AI_SERVICE_TIMEOUT',
    message: 'Scenario generation timed out',
    stack: '...'
  },
  context: {
    imageId: 'img-789',
    wardId: 'W123'
  }
}
```

**Monitoring Metrics**
- Error rate by endpoint and error code
- External API success/failure rates
- AI service latency and success rates
- Database query performance
- Cache hit/miss rates
- User-facing error rates (4xx vs 5xx)

**Alerting**
- Alert on error rate > 5% for any endpoint
- Alert on external API failure rate > 20%
- Alert on AI service latency > 90 seconds (p95)
- Alert on database connection pool exhaustion
- Alert on disk space < 20%


## Testing Strategy

### Dual Testing Approach

The FuturePulse platform requires both unit testing and property-based testing for comprehensive coverage. These approaches are complementary:

- **Unit tests** verify specific examples, edge cases, and error conditions
- **Property tests** verify universal properties across all inputs through randomization

Together, they provide comprehensive coverage where unit tests catch concrete bugs and property tests verify general correctness.

### Property-Based Testing

**Framework Selection:**
- **JavaScript/TypeScript**: fast-check
- **Python**: Hypothesis

**Configuration:**
- Minimum 100 iterations per property test (due to randomization)
- Each property test must reference its design document property
- Tag format: `Feature: futurepulse-platform, Property {number}: {property_text}`

**Property Test Examples:**

```typescript
// Property 9: Dual Scenario Generation
// Feature: futurepulse-platform, Property 9: For any valid uploaded image, 
// the projection engine should generate exactly two scenarios within 60 seconds
import fc from 'fast-check';

test('Property 9: Dual Scenario Generation', async () => {
  await fc.assert(
    fc.asyncProperty(
      fc.record({
        format: fc.constantFrom('jpeg', 'png', 'heic'),
        size: fc.integer({ min: 1000, max: 10_000_000 }),
        width: fc.integer({ min: 100, max: 4000 }),
        height: fc.integer({ min: 100, max: 4000 })
      }),
      async (imageSpec) => {
        const mockImage = generateMockImage(imageSpec);
        const startTime = Date.now();
        
        const result = await projectionEngine.generateScenarios(mockImage);
        const duration = Date.now() - startTime;
        
        expect(result.scenarios).toHaveLength(2);
        expect(result.scenarios[0].type).toBe('sustainable');
        expect(result.scenarios[1].type).toBe('degradation');
        expect(duration).toBeLessThan(60000);
      }
    ),
    { numRuns: 100 }
  );
});
```

```typescript
// Property 20: Registration Capacity Decrement
// Feature: futurepulse-platform, Property 20: For any event with available capacity,
// when a user successfully registers, the available capacity should decrease by exactly 1
test('Property 20: Registration Capacity Decrement', async () => {
  await fc.assert(
    fc.asyncProperty(
      fc.record({
        capacity: fc.integer({ min: 1, max: 100 }),
        initialRegistrations: fc.integer({ min: 0, max: 50 })
      }),
      async ({ capacity, initialRegistrations }) => {
        fc.pre(initialRegistrations < capacity); // Ensure capacity available
        
        const event = await createTestEvent({ capacity });
        await registerUsers(event.id, initialRegistrations);
        
        const beforeCapacity = await getAvailableCapacity(event.id);
        await registerUser(event.id, generateTestUser());
        const afterCapacity = await getAvailableCapacity(event.id);
        
        expect(beforeCapacity - afterCapacity).toBe(1);
      }
    ),
    { numRuns: 100 }
  );
});
```

```python
# Property 60: Data Range Validation
# Feature: futurepulse-platform, Property 60: For any incoming environmental data,
# the system should validate that values are within reasonable ranges
from hypothesis import given, strategies as st

@given(
    metric_type=st.sampled_from(['aqi', 'temperature', 'humidity']),
    value=st.floats(allow_nan=False, allow_infinity=False)
)
def test_property_60_data_range_validation(metric_type, value):
    """Property 60: Data Range Validation"""
    result = validator.validate_environmental_data(metric_type, value)
    
    if metric_type == 'aqi':
        assert result.valid == (0 <= value <= 500)
    elif metric_type == 'temperature':
        assert result.valid == (-50 <= value <= 60)
    elif metric_type == 'humidity':
        assert result.valid == (0 <= value <= 100)
```

### Unit Testing

**Framework Selection:**
- **JavaScript/TypeScript**: Jest or Vitest
- **Python**: pytest

**Unit Test Focus Areas:**

**1. Specific Examples**
```typescript
// Example: Cleanup event awards 10 points
test('Cleanup event awards 10 reputation points', async () => {
  const user = await createTestUser();
  const event = await createTestEvent({ type: 'cleanup' });
  
  await completeEvent(event.id, [user.id]);
  
  const reputation = await getReputation(user.id);
  expect(reputation.totalPoints).toBe(10);
});

// Example: Tree plantation event awards 15 points
test('Tree plantation event awards 15 reputation points', async () => {
  const user = await createTestUser();
  const event = await createTestEvent({ type: 'tree_plantation' });
  
  await completeEvent(event.id, [user.id]);
  
  const reputation = await getReputation(user.id);
  expect(reputation.totalPoints).toBe(15);
});
```

**2. Edge Cases**
```typescript
// Edge case: Empty image upload
test('Rejects empty image upload', async () => {
  const emptyFile = new File([], 'empty.jpg', { type: 'image/jpeg' });
  
  await expect(projectionEngine.uploadImage(emptyFile))
    .rejects.toThrow('Image file is empty');
});

// Edge case: Event at exact capacity
test('Prevents registration when event is at exact capacity', async () => {
  const event = await createTestEvent({ capacity: 5 });
  await registerUsers(event.id, 5); // Fill to capacity
  
  const newUser = await createTestUser();
  await expect(registerUser(event.id, newUser.id))
    .rejects.toThrow('Event is at capacity');
});

// Edge case: Exactly 90 days old event
test('Includes event from exactly 90 days ago in civic score', async () => {
  const ward = await createTestWard();
  const event = await createTestEvent({
    wardId: ward.id,
    completedAt: new Date(Date.now() - 90 * 24 * 60 * 60 * 1000)
  });
  
  const score = await calculateWardScore(ward.id);
  expect(score.eventsCount).toBe(1);
});
```

**3. Error Conditions**
```typescript
// Error: Invalid email format
test('Rejects registration with invalid email', async () => {
  await expect(authService.register({
    email: 'not-an-email',
    password: 'SecurePass123',
    displayName: 'Test User',
    wardId: 'W123'
  })).rejects.toThrow('Invalid email format');
});

// Error: Weak password
test('Rejects registration with weak password', async () => {
  await expect(authService.register({
    email: 'test@example.com',
    password: 'weak',
    displayName: 'Test User',
    wardId: 'W123'
  })).rejects.toThrow('Password does not meet strength requirements');
});

// Error: External API timeout
test('Falls back to cached data on API timeout', async () => {
  mockExternalAPI.mockImplementation(() => {
    throw new Error('TIMEOUT');
  });
  
  const cachedData = { aqi: 50, timestamp: new Date() };
  await cache.set('ward-W123-aqi', cachedData);
  
  const result = await dataFetcher.fetchAQI('W123');
  expect(result).toEqual(cachedData);
});
```

**4. Integration Points**
```typescript
// Integration: Event creation triggers notifications
test('Creating event queues notifications for ward users', async () => {
  const ward = await createTestWard();
  const users = await createTestUsers(5, { wardId: ward.id });
  const chapter = await createTestChapter({ wardId: ward.id });
  
  const event = await eventManager.createEvent({
    chapterId: chapter.id,
    type: 'cleanup',
    title: 'Beach Cleanup',
    date: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    capacity: 20,
    wardId: ward.id
  });
  
  const notifications = await getQueuedNotifications();
  expect(notifications).toHaveLength(5);
  expect(notifications.every(n => n.type === 'civic_event')).toBe(true);
});

// Integration: Account deletion removes all user data
test('Account deletion cascades to all related data', async () => {
  const user = await createTestUser();
  await createTestReputation(user.id);
  await createTestRegistration(user.id);
  await createTestProjection(user.id);
  
  await userService.deleteAccount(user.id);
  
  expect(await getUser(user.id)).toBeNull();
  expect(await getReputation(user.id)).toHaveLength(0);
  expect(await getRegistrations(user.id)).toHaveLength(0);
  expect(await getProjections(user.id)).toHaveLength(0);
});
```

### Test Coverage Goals

- **Unit test coverage**: Minimum 80% line coverage
- **Property test coverage**: All 67 correctness properties implemented
- **Integration test coverage**: All critical user flows (registration → event participation → reputation)
- **E2E test coverage**: Key user journeys (new user onboarding, creating projection, joining event)

### Test Data Management

**Generators for Property Tests:**
```typescript
// Generate random valid images
const imageGenerator = fc.record({
  format: fc.constantFrom('jpeg', 'png', 'heic'),
  size: fc.integer({ min: 1000, max: 10_000_000 }),
  dimensions: fc.record({
    width: fc.integer({ min: 100, max: 4000 }),
    height: fc.integer({ min: 100, max: 4000 })
  }),
  hasGeoTag: fc.boolean(),
  geoTag: fc.option(fc.record({
    lat: fc.float({ min: -90, max: 90 }),
    lon: fc.float({ min: -180, max: 180 })
  }))
});

// Generate random events
const eventGenerator = fc.record({
  type: fc.constantFrom('cleanup', 'tree_plantation'),
  capacity: fc.integer({ min: 5, max: 100 }),
  date: fc.date({ min: new Date(), max: new Date(Date.now() + 365 * 24 * 60 * 60 * 1000) }),
  duration: fc.integer({ min: 30, max: 480 })
});

// Generate random environmental readings
const readingGenerator = fc.record({
  aqi: fc.integer({ min: 0, max: 500 }),
  temperature: fc.float({ min: -50, max: 60 }),
  humidity: fc.float({ min: 0, max: 100 })
});
```

**Test Database:**
- Use separate test database with same schema as production
- Reset database between test suites
- Use transactions for test isolation where possible
- Seed with minimal required data (wards, test users)

### Continuous Integration

**CI Pipeline:**
1. Lint and format check
2. Unit tests (parallel execution)
3. Property tests (parallel execution)
4. Integration tests
5. E2E tests (critical paths only)
6. Security scans (SAST, dependency vulnerabilities)
7. Build and containerize
8. Deploy to staging

**Test Execution Time:**
- Unit tests: < 5 minutes
- Property tests: < 10 minutes (100 iterations each)
- Integration tests: < 5 minutes
- E2E tests: < 10 minutes
- Total CI time: < 30 minutes

### Performance Testing

**Load Testing:**
- Simulate 10,000 concurrent users
- Test projection engine under load (100 projections/minute)
- Test event registration race conditions (100 concurrent registrations)
- Test database query performance under load

**Stress Testing:**
- Gradually increase load until system degrades
- Identify bottlenecks (database, AI services, API gateway)
- Test auto-scaling behavior

**Tools:**
- k6 or Artillery for load testing
- Locust for user behavior simulation
- Database query profiling tools

---

**Document Version:** 1.0  
**Last Updated:** 2024  
**Status:** Draft - Pending Review
