# Higgsfield.ai Research Analysis
**Date:** 2026-02-13  
**Research Method:** Perplexity Pro (sonar-pro model)  
**Purpose:** Comprehensive analysis for building Higgsfield CLI client

---

## 1. What is Higgsfield.ai?

### Overview
Higgsfield.ai is a generative AI platform specializing in **cinematic-quality image and video creation**, designed for creators, marketers, enterprises, and professionals like filmmakers and social media storytellers. The platform generates approximately **4 million videos daily**.

### Core Capabilities
- **Video Generation**: Creates cinematic short-form videos using models like OpenAI's Sora 2 for motion, realism, and continuity
  - Supports up to **30-second clips**
  - **Kling Motion Control** for precise character actions and expressions
  - Includes audio in Kling 2.6 videos
  - Resolution: 720p or 1080p
  - Generation time: ~1 minute

- **Image Generation**: Produces realistic images with specialized tools:
  - **Skin Enhancer** for natural textures
  - **Higgsfield Soul** for consistent characters across images and videos

- **Advanced Controls**:
  - **WAN Camera Controls** (Angles 2.0): Precise motions like crash zooms, dolly moves, overheads, boltcam angles
  - **AI Influencer Studio** for viral character creation
  - Professional motion control outperforming competitors

- **Workflows**:
  - Text-to-video/image with realistic lighting and framing
  - **Product-to-Video** (Click-to-Ad): Analyzes product pages/links to auto-generate ads
  - Presets for trending social formats
  - Image-to-video transformations

### Unique Differentiators

1. **Cinematic Logic Layer**: Uses GPT models to plan narratives, pacing, camera logic, and virality patterns *before* generation
   - Reduces prompting to minimal inputs
   - Usable results in 1-2 tries (vs. 5-6 iterations elsewhere)

2. **Virality Optimization**: Analyzes social data to create daily-updated presets (e.g., Sora 2 Trends)
   - 150% boost in share velocity
   - 3x engagement over baselines

3. **Professional Motion Control**: First platform with true cinematic camera precision

4. **Integrated Ecosystem**: Combines planning, generation, enhancement, and enterprise tools in one workspace
   - Routes models dynamically by task (GPT-4.1 mini for precision, GPT-5 for inference)

5. **No-Code Accessibility**: Transforms static images into dynamic videos automatically

### Enterprise Features
- Team collaboration
- Secure sharing (SOC 2-aligned)
- Role assignments and approvals
- Unified access to leading models (GPT-4.1, GPT-5, Sora 2)

---

## 2. Tech Stack

### Backend/ML Infrastructure
- **Primary Framework**: **PyTorch** for machine learning models
  - ZeRO-3 DeepSpeed API support
  - Fully sharded data parallel (FSDP)
- **Open-Source Framework**: **Higgsfield** framework for fault-tolerant GPU orchestration
  - Large-scale training (e.g., Llama 70B models)
  - 3.5k stars, 589 forks on GitHub
  - Python-based, pip-installable: `pip install higgsfield==0.0.3`

### Frontend
- **Unknown Framework**: Specific frontend framework (React, Vue, etc.) not disclosed in available sources
- Likely uses standard web technologies
- Main interfaces:
  - Website: higgsfield.ai
  - Cloud API: cloud.higgsfield.ai

### API Infrastructure
- **Public Cloud API**: Available at cloud.higgsfield.ai for programmatic video generation
- **REST API** architecture with JSON payloads
- **Base URLs** (varies across integrations):
  - api.higgsfield.ai
  - higgsfieldapi.com/api
  - api.segmind.com (possibly hosted/partners)

### Known API Patterns
- **Asynchronous task-based generation**:
  1. Submit generation → returns task ID + status URL
  2. Poll status endpoint → check progress
  3. Retrieve completed outputs → array of URLs

- **Endpoint Structure** (inferred from SDK/docs):
  - POST `/v1/generations` or `/v1/generate` - Submit generation
  - POST `/v1/higgsfield-image2video` - Image-to-video conversion
  - GET `/v1/generations/{id}` - Check status
  - DELETE `/v1/generations/{id}` - Cancel task

- **Task Statuses**: processing, completed

### Official Repositories
1. **higgsfield-ai/higgsfield** - GPU orchestration framework (3.5k stars)
2. **higgsfield-ai/higgsfield-client** - Official Python SDK
3. **higgsfield-ai/invoker** - Cluster management (Go)
4. **higgsfield-ai/docker** - Internal Docker images
5. **higgsfield-ai/actions** - GitHub Actions

### Third-Party Integrations
- No native integrations with Premiere, HubSpot, etc.
- Automated workflows available (details limited)

---

## 3. Authentication

### User Authentication Methods

1. **Google OAuth**:
   - Direct signup/login with Google account
   - Shares profile data (username, profile picture) based on Google account settings
   - Adheres to Google's API Services User Data Policy

2. **Email/Password**:
   - Traditional email address and password authentication
   - Users set username and password during registration

3. **No Other Social Logins**: Only Google OAuth mentioned; no Facebook, Twitter, etc.

### API Authentication

- **Bearer Token**: `Authorization: Bearer YOUR_API_KEY`
- **API Key Source**: Obtained from user dashboard after account subscription and login
- **SDK Support**: Official JavaScript SDK (v2) uses configurable credentials field

### Implementation Details

- **Authentication Provider**: Unknown (not Auth0, Firebase, Clerk, or Supabase based on available info)
- Appears to be **custom implementation** with API key support
- Third-party data sharing with linked services (e.g., Google) governed by their policies
- Users manage profiles and data via dashboard
- No disclosed details on backend auth flow (JWT, session management, etc.)

### Integration Examples
- API integrations via:
  - Make.com (requires API key)
  - n8n (requires API key)

---

## 4. Known API Endpoints & Documentation

### Official SDK
- **Repository**: higgsfield-ai/higgsfield-client
- **Language**: Python
- **Features**: Synchronous and asynchronous support
- **Installation**: Via GitHub (not on PyPI at time of research)

### API Endpoints (Inferred from Third-Party Docs)

| Endpoint | Method | Purpose | Parameters |
|----------|--------|---------|------------|
| `/v1/generations` | POST | Submit generation task | task, model, prompt, width, height, steps, seed, enhance_prompt, check_nsfw |
| `/v1/generate` | POST | Generate image/video | prompt, image_url, duration, aspect_ratio |
| `/v1/higgsfield-image2video` | POST | Image-to-video conversion | prompt, image_url, duration |
| `/v1/generations/{id}` | GET | Check task status | - |
| `/v1/generations/{id}` | DELETE | Cancel task | - |

### Common Parameters

**Text-to-Image/Video**:
- `model`: Model selection (e.g., "flux", "dop-preview", "kling-3.0")
- `prompt`: Text description
- `width`, `height`: Dimensions (e.g., 1024x1024)
- `aspect_ratio`: Alternative to explicit dimensions
- `steps`: Generation steps/quality
- `seed`: Reproducibility seed
- `enhance_prompt`: Boolean for auto-enhancement
- `check_nsfw`: Boolean for content filtering

**Image-to-Video**:
- `image_url`: Source image URL
- `reference_image_urls`: For Soul Mode (character consistency)
- `duration`: Video length (3-15 seconds)

**Soul Mode** (Character Consistency):
- `reference_image_urls`: Array of reference images for consistent character generation

### Response Structure
```json
{
  "id": "task-id-uuid",
  "status": "processing|completed",
  "outputs": ["https://url-to-generated-media"],
  "nsfw": false,
  "related_endpoints": {
    "status": "/v1/generations/{id}",
    "cancel": "/v1/generations/{id}"
  }
}
```

### Documentation Sources
- **No official comprehensive API docs** found at higgsfield.ai
- **Third-party documentation** from:
  - Segmind (partner/hosting)
  - Apidog
  - Scribd
  - WaveSpeed
  - YouTube tutorials

- **Official How-to Guides**: Available at Higgsfield.ai site for workflows, but no API specifics

### GitHub Reverse-Engineering Status
- **No unofficial clients found**
- **No reverse-engineered documentation**
- **No Python wrappers** (beyond official SDK)
- **No JavaScript libraries** (beyond official SDK v2)
- Only related repos:
  - Anil-matcha/Open-Higgsfield-AI: Open-source local clone (not API client)
  - Anil-matcha/higgsfield-ai-scam: Critical commentary (not technical)

---

## 5. Rate Limits and Pricing

### Pricing Tiers

| Plan | Monthly Price | Annual Equivalent | Credits/Month | Key Features |
|------|---------------|-------------------|---------------|--------------|
| **Free** | $0 | $0 | 5-10/day or 10 total | Basic access, limited models/jobs, not ideal for serious use |
| **Basic** | $9 | ~$7.20 | 150 | Entry-level, ~75 Nano Banana Pro generations |
| **Pro** | $29 | ~$17.40-24 | 500-600 | Advanced models, priority queue, ~300 Nano Banana Pro gens (most popular) |
| **Ultimate** | $49 | ~$24.50-39 | 1,000-1,200 | Unlimited on select models, higher concurrency |
| **Creator** | $149 | ~$49.80-119 | 5,000-6,000 | API access, teams, extended unlimited (2-year Nano Banana Pro) |
| **Custom** | Variable | Variable | Custom | Enterprise with dedicated support |

**Note**: Pricing shows variance across sources due to promotions, annual discounts (~20%), and updates. Always verify at higgsfield.ai/pricing.

### Credit Consumption

**Video Generation**:
- Basic text-to-video: **15-20 credits**
- Advanced videos (Veo 3.1, Sora 2): **50-80+ credits**
- 150 credits = ~2-3 quality videos

**Image Generation**: **0.25-5 credits** per image

**Factors Affecting Credit Cost**:
- Model complexity
- Video length
- Resolution (720p vs 1080p)
- Special features (audio sync, VFX, upscale)

### Rate Limits

**Concurrent Job Limits** (varies by plan):
- Basic: 2 videos / 2 images simultaneously
- Ultimate: 4 videos / 8 images simultaneously
- Higher tiers: Priority queue access

**Throttling**:
- Unlimited generation plans may have peak-time throttling
- Personal-use limits apply even on unlimited tiers

### Billing Details

- **Monthly or Annual**: Annual saves ~20%
- **Overage Credits**: Available via packs (e.g., 80-100 credits for $5)
- **No True Free Trial**: Free tier is limited; Basic acts as paid testing tier
- **Promotions**: Past examples include Cyber Week discounted plans ($35 with $0.058-$0.32 per generation)

### Usage System
- Credit-based consumption
- Credits deducted per generation
- No rollover mentioned
- API access restricted to Creator+ plans

---

## 6. Image/Video Generation Workflow

### User Flow (Step-by-Step)

#### 1. **Open Video Workspace**
- Access Higgsfield platform
- Navigate to video generation area
- Canvas-based editing studio interface

#### 2. **Choose Model**
Available models based on use case:
- **Kling 3.0**: Scene-based multi-shot (2-6 scenes), elements/subject consistency
- **Kling 2.6**: Lip-sync, voice alignment for performance/dialogue
- **Sora 2**: Cinematic motion and realism
- Other specialized models for speed, fidelity, continuity

#### 3. **Upload Input & Set Controls**

**Image Requirements** (for image-to-video):
- Sharp, well-lit
- Clear subject
- Good composition

**Prompt Structure** (recommended):
```
[Shot type], [camera angle]. [Subject action]. [Camera movement]. [Lighting/mood].

Example:
"Medium close-up, eye level. Subject turns slightly. Slow dolly-in. Soft cinematic lighting."
```

**Optional Elements**:
- Start/end frames for motion guidance
- Camera movement presets (pan, tilt, dolly, zoom)
- Style presets or templates

#### 4. **Define Structure** (Kling 3.0 specific)
- Set **2-6 scenes**
- Describe each scene
- Assign durations per scene
- Add **elements** (characters/objects) for cross-scene consistency
- Apply frame constraints

#### 5. **Generate**
- Click generate button
- Output specifications:
  - Duration: **3-15 seconds**
  - Resolution: **720p or 1080p**
  - Audio: Optional (on/off)
  - Generation time: **~1 minute**

#### 6. **Review & Edit**
- Place clip on **canvas** as base layer
- Refine:
  - Extend scenes
  - Adjust motion
  - Add typography/transitions
- **Iterate** by tweaking one variable at a time (prompt, image, or model)

#### 7. **Export**
- Generate publish-ready video from canvas

### Controllable Parameters

| Parameter | Options | Notes |
|-----------|---------|-------|
| **Prompts** | Freeform text | Overall concept, visual style, camera behavior, motion, scene descriptions |
| **Style** | Presets or prompt-based | Cinematic looks, trending styles |
| **Aspect Ratio** | Not explicitly listed | Inferred via canvas/export, standard video resolutions |
| **Duration** | 3-15 seconds | Per-scene assignment in Kling 3.0, slow-motion option |
| **Models** | User-selectable | Task-specific selection |
| **Audio** | On/Off | Available in certain models (e.g., Kling 2.6) |
| **Resolution** | 720p, 1080p | Quality tier selection |
| **Camera Movements** | Presets | Flying cam, dolly-in, pan, tilt, zoom, crash zoom, overhead, boltcam |
| **Start/End Frames** | Upload frames | Motion guidance |
| **Elements/Subjects** | Character/object definitions | Cross-scene consistency (Kling 3.0) |

### Available Models

1. **Kling 3.0**:
   - Scene-based multi-shot (2-6 scenes)
   - Elements/subject consistency
   - Post-generation editing
   - Precise pacing/motion control

2. **Kling 2.6**:
   - Lip-sync capabilities
   - Voice alignment
   - Performance/dialogue focus

3. **Sora 2** (OpenAI):
   - Cinematic motion
   - Realism and continuity
   - Up to 30-second clips

4. **Other Models**:
   - Nano Banana Pro (unlimited on higher tiers)
   - Seedream 4.0 (unlimited on higher tiers)
   - Flux
   - DOP Preview
   - Veo 3.1

### Workflow Patterns

**Text-to-Image/Video**:
1. Write prompt
2. Select model
3. Configure parameters
4. Generate
5. Refine

**Image-to-Video**:
1. Upload image
2. Add motion prompt
3. Select camera movements
4. Configure duration/audio
5. Generate
6. Iterate

**Product-to-Video** (Click-to-Ad):
1. Provide product page URL
2. AI analyzes product
3. Auto-generates ad video
4. Review and customize
5. Export

**Soul Mode** (Character Consistency):
1. Upload reference images
2. Generate consistent character across multiple outputs
3. Use in images and videos

### Best Practices (from research)
- Prioritize speed via presets and layers over single-prompt generation
- Iterate by changing one variable at a time
- Use sharp, well-lit images for image-to-video
- Structure prompts with shot type, camera angle, action, movement, lighting
- Leverage canvas for layered editing vs. re-generation
- Workflow mimics production studio approach

---

## Key Findings Summary

### ✅ What We Know
1. **Platform Purpose**: Cinematic AI video/image generation with virality optimization
2. **Tech Stack**: PyTorch backend, REST API, official Python SDK exists
3. **Auth**: Google OAuth + Email/Password, Bearer token API auth
4. **Pricing**: Credit-based, $9-$149/month, annual discounts available
5. **Workflow**: Canvas-based, multi-model, preset-driven with 1-minute generation
6. **Models**: Kling 3.0/2.6, Sora 2, and others for different use cases

### ⚠️ What We Don't Know
1. **Frontend Framework**: React/Vue/other not disclosed
2. **Auth Provider**: Unknown (custom implementation suspected)
3. **Complete API Docs**: No official comprehensive documentation found
4. **Exact Endpoints**: Inferred from third-party sources, not officially documented
5. **SDK Methods**: Python SDK exists but internal methods not detailed
6. **Rate Limit Specifics**: Concurrent job limits known, but no requests-per-minute data

### 🚧 Challenges for CLI Development
1. **No Official API Docs**: Will need to reverse-engineer from SDK or network inspection
2. **No Unofficial Clients**: No community examples to reference
3. **Async Task Model**: Will need polling mechanism for status checks
4. **Credit System**: Need to track and display credit consumption
5. **Model Selection**: Need to expose model options to CLI users
6. **File Uploads**: Need to handle local image uploads for image-to-video

### 🎯 Recommended Next Steps
1. **Inspect Official Python SDK**: Clone higgsfield-ai/higgsfield-client, analyze code
2. **Network Analysis**: Sign up for account, use browser DevTools to capture API calls
3. **Test API Endpoints**: Validate inferred endpoints from third-party docs
4. **Authentication Flow**: Test API key generation and Bearer token auth
5. **Build Prototype**: Create minimal CLI with text-to-video generation
6. **Expand Features**: Add image-to-video, model selection, status polling

---

## Research Metadata
- **Date**: 2026-02-13
- **Researcher**: Clawdbot Subagent
- **Method**: Perplexity Pro (sonar-pro model)
- **Queries**: 7 research queries
- **Cost**: $0.042 (7 queries × ~$0.006 each)
- **Sources**: Mixed (official repos, third-party docs, platform descriptions)
- **Confidence**: High for platform capabilities, Medium for API specifics, Low for exact implementation details

---

**End of Analysis**
