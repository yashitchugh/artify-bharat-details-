# Design Document: Artisan Identity Platform

## Overview

The Artisan Identity Platform is a web-based system that creates verifiable digital identities for Indian artisans and binds handmade products to their makers. The platform addresses the authenticity crisis in India's handmade products market by providing cryptographic proof of origin, voice-based storytelling, and multi-factor verification of artisan credentials.

The system consists of three primary user-facing components: an artisan portal for registration and product management, a verification subsystem for validating artisan credentials, and a public verification interface for buyers to authenticate products. The architecture emphasizes data integrity, privacy protection, and ease of use for artisans with varying levels of digital literacy.

## Architecture

The platform follows a three-tier architecture:

**Presentation Layer:**
- Artisan Portal (web application for artisan registration and product management)
- Buyer Verification Interface (public-facing product authentication)
- Admin Dashboard (verification workflow management)

**Application Layer:**
- Authentication Service (artisan login and session management)
- Registration Service (artisan profile creation and updates)
- Verification Service (location and craft validation logic)
- Product Service (product registration and digital binding)
- Voice Service (audio recording, storage, and playback)
- Search Service (artisan and product discovery)

**Data Layer:**
- Artisan Database (profile data, verification status, credentials)
- Product Database (product records, digital bindings, metadata)
- Media Storage (voice recordings, craft documentation, photos)
- Audit Log (verification history, profile changes, access records)

**External Integrations:**
- SMS/Email Gateway (notifications and verification codes)
- Geolocation Service (location validation)
- Cloud Storage (scalable media file storage)

The system uses a RESTful API architecture with JWT-based authentication for artisans and API key authentication for the public verification interface.

## Components and Interfaces

### 1. Registration Service

**Responsibilities:**
- Create new artisan profiles
- Validate registration data completeness
- Prevent duplicate registrations
- Initialize verification workflow

**Interface:**
```
POST /api/artisans/register
Request: {
  name: string,
  phone: string,
  email: string,
  location: {
    address: string,
    city: string,
    state: string,
    pincode: string
  },
  craftType: string,
  craftDescription: string
}
Response: {
  artisanId: string,
  status: "pending" | "verified" | "rejected",
  message: string
}
```

**Key Functions:**
- `validateRegistrationData(data)`: Checks completeness and format of registration fields
- `checkDuplicateArtisan(phone, email)`: Queries database for existing registrations
- `createArtisanProfile(data)`: Inserts new profile with unique ID and pending status
- `initiateVerification(artisanId)`: Triggers location and craft verification workflows

### 2. Verification Service

**Responsibilities:**
- Validate artisan location against craft regions
- Evaluate craft authenticity evidence
- Update verification status
- Maintain verification audit trail

**Interface:**
```
POST /api/verification/location
Request: {
  artisanId: string,
  gpsCoordinates: { latitude: number, longitude: number },
  addressProof: file
}
Response: {
  verified: boolean,
  method: "gps" | "document" | "manual",
  timestamp: datetime
}

POST /api/verification/craft
Request: {
  artisanId: string,
  craftType: string,
  evidence: file[],
  description: string
}
Response: {
  verified: boolean,
  reviewerId: string,
  feedback: string,
  timestamp: datetime
}
```

**Key Functions:**
- `validateLocation(coordinates, address)`: Checks location against known craft regions
- `evaluateCraftEvidence(craftType, evidence)`: Assesses authenticity of submitted craft work
- `updateVerificationStatus(artisanId, status, reason)`: Updates profile with verification outcome
- `logVerificationAttempt(artisanId, type, outcome)`: Records verification history

### 3. Product Service

**Responsibilities:**
- Register new products with unique identifiers
- Bind products to artisan profiles
- Generate non-transferable digital bindings
- Prevent post-registration tampering

**Interface:**
```
POST /api/products/register
Request: {
  artisanId: string,
  name: string,
  description: string,
  craftType: string,
  creationDate: date,
  voiceStoryId?: string
}
Response: {
  productId: string,
  digitalBinding: string,
  qrCode: string,
  registrationTimestamp: datetime
}

GET /api/products/verify/{digitalBinding}
Response: {
  product: {
    productId: string,
    name: string,
    description: string,
    craftType: string,
    creationDate: date,
    registrationTimestamp: datetime
  },
  artisan: {
    artisanId: string,
    name: string,
    location: string,
    craftType: string,
    verificationStatus: string,
    verifiedAt: datetime
  },
  voiceStory?: {
    url: string,
    duration: number
  }
}
```

**Key Functions:**
- `generateDigitalBinding()`: Creates unique, cryptographically secure product identifier
- `createProductRecord(artisanId, productData)`: Inserts product with binding to artisan
- `linkVoiceStory(productId, voiceStoryId)`: Associates audio recording with product
- `verifyProductAuthenticity(digitalBinding)`: Retrieves product and artisan data for verification
- `preventBindingModification(productId)`: Enforces immutability of digital binding

### 4. Voice Service

**Responsibilities:**
- Record and upload audio from artisans
- Store voice recordings securely
- Serve audio for playback to buyers
- Validate audio format and duration

**Interface:**
```
POST /api/voice/upload
Request: {
  artisanId: string,
  audioFile: file,
  productId?: string
}
Response: {
  voiceStoryId: string,
  duration: number,
  url: string
}

GET /api/voice/{voiceStoryId}
Response: audio/mpeg stream
```

**Key Functions:**
- `validateAudioFormat(file)`: Checks file type and codec compatibility
- `validateDuration(audioFile)`: Ensures recording is between 30 seconds and 5 minutes
- `storeVoiceRecording(artisanId, audioFile)`: Uploads to cloud storage with access controls
- `generatePlaybackUrl(voiceStoryId)`: Creates time-limited signed URL for audio access
- `associateWithProduct(voiceStoryId, productId)`: Links voice story to product record

### 5. Search Service

**Responsibilities:**
- Index artisan profiles for discovery
- Filter by craft type and location
- Rank results by verification status
- Provide search suggestions

**Interface:**
```
GET /api/search/artisans?craftType={type}&location={location}&verified={boolean}
Response: {
  results: [
    {
      artisanId: string,
      name: string,
      craftType: string,
      location: string,
      verificationStatus: string,
      productCount: number
    }
  ],
  total: number,
  page: number
}
```

**Key Functions:**
- `searchByCraftType(craftType, verified)`: Queries artisans by craft with verification filter
- `searchByLocation(location, verified)`: Queries artisans by geographic area
- `rankResults(results)`: Sorts with verified artisans first
- `generateSuggestions(query)`: Provides alternative search terms for empty results

## Data Models

### Artisan Profile
```
{
  artisanId: string (UUID, primary key),
  name: string,
  phone: string (encrypted),
  email: string (encrypted),
  location: {
    address: string,
    city: string,
    state: string,
    pincode: string,
    gpsCoordinates: { latitude: number, longitude: number },
    verified: boolean,
    verificationMethod: "gps" | "document" | "manual",
    verifiedAt: datetime
  },
  craftType: string,
  craftDescription: string,
  craftVerified: boolean,
  craftVerifiedAt: datetime,
  craftVerificationEvidence: string[],
  validationStatus: "pending" | "verified" | "rejected",
  rejectionReason: string,
  createdAt: datetime,
  updatedAt: datetime,
  publicProfile: {
    displayName: string,
    craftType: string,
    location: string (city/state only),
    verificationBadges: string[]
  }
}
```

### Product Record
```
{
  productId: string (UUID, primary key),
  artisanId: string (foreign key),
  digitalBinding: string (unique, immutable),
  name: string,
  description: string,
  craftType: string,
  creationDate: date,
  registrationTimestamp: datetime,
  voiceStoryId: string (optional),
  qrCodeUrl: string,
  metadata: {
    dimensions: string,
    materials: string[],
    techniques: string[]
  }
}
```

### Voice Story
```
{
  voiceStoryId: string (UUID, primary key),
  artisanId: string (foreign key),
  productId: string (optional, foreign key),
  audioUrl: string,
  duration: number (seconds),
  format: string,
  uploadedAt: datetime,
  transcription: string (optional)
}
```

### Verification Log
```
{
  logId: string (UUID, primary key),
  artisanId: string (foreign key),
  verificationType: "location" | "craft",
  outcome: "success" | "failure" | "pending",
  method: string,
  reviewerId: string (optional),
  evidence: string[],
  feedback: string,
  timestamp: datetime
}
```


## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Property 1: Unique Profile Creation
*For any* valid registration data, submitting it to the registration service should create exactly one Artisan_Profile with a unique artisanId that differs from all existing profiles.
**Validates: Requirements 1.1, 1.5**

### Property 2: Complete Field Capture
*For any* registration or product creation request, all required fields specified in the request should be present and non-empty in the created record.
**Validates: Requirements 1.2, 5.3**

### Property 3: 