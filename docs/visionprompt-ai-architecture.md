# VisionPrompt AI — SaaS Architecture, AI Workflow, and 4-Week MVP Roadmap

_Last updated: May 9, 2026_

## 1. Product Mission

VisionPrompt AI lets creators upload an image or video and receive a professional, generator-ready prompt that reverse-engineers the source asset's subject, visual medium, lighting, camera/lens choices, composition, color science, motion language, and style references. The generated prompt should be usable in Midjourney, Stable Diffusion, Runway Gen-3, and similar image/video generation tools.

## 2. Recommended Modern Tech Stack

### Frontend

- **Framework:** Next.js 15+ with React Server Components, App Router, TypeScript, and Tailwind CSS.
- **UI system:** shadcn/ui + Radix primitives for accessible SaaS dashboards, upload flows, billing pages, and prompt history.
- **Client upload layer:** Uppy or FilePond for resumable uploads, progress bars, MIME validation, and drag-and-drop image/video ingestion.
- **State/data:** TanStack Query for client-side job polling and cache management; server actions or route handlers for low-latency authenticated mutations.
- **Observability:** Sentry for frontend exceptions and PostHog or OpenPanel for product analytics.

### Backend

- **Primary API:** Next.js route handlers for lightweight CRUD, signed upload URL creation, billing webhooks, and account management.
- **AI/job services:** Node.js workers with BullMQ + Redis for MVP; move heavy video preprocessing to a separate Python/FastAPI worker if advanced FFmpeg/OpenCV processing grows.
- **Queue:** Upstash Redis, AWS ElastiCache Redis, or DragonflyDB. Use delayed/retry jobs for media processing and AI calls.
- **Media processing:** FFmpeg for keyframe extraction, scene detection, thumbnails, and duration/codec inspection. Use `ffprobe` before accepting a file into the analysis queue.
- **Deployment:** Vercel for the Next.js app, Fly.io/Render/Railway for workers during MVP, then AWS ECS/Fargate or Kubernetes for enterprise scale.

### Database, Auth, and Core Data

- **Database:** PostgreSQL with Prisma or Drizzle ORM. Supabase Postgres is excellent for MVP speed; Neon or AWS Aurora Serverless v2 are strong alternatives.
- **Authentication:** Clerk or Supabase Auth for fastest launch. Use organization/team support early because creative agencies will need shared credit pools.
- **Cache/rate limits:** Redis for upload-session state, job locks, credit holds, and per-user rate limiting.
- **Search/history:** PostgreSQL full-text search for prompt history at MVP; upgrade to Meilisearch/OpenSearch only if discovery becomes complex.

## 3. AI Architecture Details

### Static Image Analysis API

Use **OpenAI Responses API with image input** as the primary image analysis path. OpenAI's Responses API supports text and image input and can return text or JSON output, which fits a structured prompt-generation pipeline. For high-fidelity prompt extraction, prefer a strong vision-capable model for paid tiers and a cheaper vision-capable model for drafts or previews.

Recommended flow:

1. Browser uploads the original image directly to object storage through a short-lived signed URL.
2. Backend creates an `analysis_jobs` row with status `queued`, file metadata, estimated token/credit cost, and a short-lived internal read URL.
3. Worker validates MIME type, dimensions, NSFW/safety metadata if required, and file size.
4. Worker sends the image to the vision model with:
   - a forensic-analysis system instruction,
   - the image input,
   - a strict JSON schema for intermediate analysis,
   - and a second synthesis pass that converts the analysis into prompts for Midjourney, Stable Diffusion, and Runway.
5. Worker stores the structured analysis, final prompts, token usage, model version, and reproducibility metadata.

Model strategy:

- **Pro quality:** latest strong OpenAI vision model available in the account, selected by capability and price at deployment time.
- **Fast/preview quality:** lower-cost OpenAI vision model for quick drafts.
- **Fallback vendor:** Gemini or Claude vision for redundancy if the primary provider is unavailable or a customer's region requires a different vendor.

### Video Analysis Workflow: Native Video vs. Keyframes

Use a **hybrid video workflow**:

- **MVP default:** Extract keyframes and representative clips, then send selected frames to a vision model. This is predictable, cheaper, auditable, and works across providers that accept images even when they do not accept native video input.
- **Premium/long-form option:** Use a native video-understanding provider when the requested output depends on temporal details, transitions, choreography, dialogue/audio, or exact timestamps. Gemini's video-understanding documentation describes native video input options, including uploaded video files for larger/longer assets and frame-rate controls, making it a strong candidate for this path.

Decision matrix:

| Use case | Recommended method | Why |
| --- | --- | --- |
| Single-scene image-like video, short ad clip, loop, product shot | Keyframes | Lower cost; enough for visual style, lighting, camera, composition. |
| Music video, action scene, multi-shot trailer, camera movement analysis | Native video + keyframes | Native temporal context plus sampled frames for visual auditability. |
| Customer needs Runway Gen-3 motion prompt | Keyframes + motion extraction | Use FFmpeg scene changes and optical-flow/motion notes to describe movement. |
| Long video over provider limits or high cost | Scene detection + sampled keyframes | Summarize only salient frames and avoid runaway token usage. |

Video processing pipeline:

1. User uploads video directly to object storage.
2. Worker runs `ffprobe` to capture codec, duration, resolution, bitrate, frame rate, and audio presence.
3. Worker rejects or asks for compression if the file exceeds plan limits.
4. Worker extracts:
   - poster frame,
   - scene-change frames,
   - interval frames such as every 2-5 seconds,
   - optional short animated preview/GIF,
   - optional audio transcript if the user enables audio context.
5. Worker deduplicates near-identical frames with perceptual hashing.
6. Worker sends the top N frames to the image vision model for visual analysis.
7. Worker optionally sends the original video to a native video model for motion/timestamp analysis.
8. A final synthesis call merges visual, motion, camera, lighting, and style observations into platform-specific prompts.

## 4. Prompt Engineering Design

### Required Output Fields

Every completed analysis should produce:

- **Forensic summary:** 2-4 sentences describing what the file depicts.
- **Subject:** primary subject, secondary subjects, pose/action, wardrobe/props, environment.
- **Medium:** photo, cinematic film still, 3D render, anime, oil painting, product render, documentary, etc.
- **Lighting:** key/fill/rim light, softness, direction, time of day, contrast ratio, practical lights, volumetrics.
- **Camera:** shot size, angle, lens estimate, focal length range, aperture/depth of field, shutter/motion blur, film stock/sensor look.
- **Composition:** framing, rule of thirds/symmetry, leading lines, negative space, foreground/midground/background, aspect ratio.
- **Color/style:** palette, grading, texture, era, genre, artist/director/style references.
- **Motion for video:** camera movement, subject movement, pacing, transitions, temporal changes.
- **Generator prompts:** Midjourney, Stable Diffusion/SDXL, and Runway Gen-3 variants.
- **Negative prompt:** artifacts to avoid, if appropriate.
- **Confidence notes:** inferred vs. directly visible details.

### System Instruction

```text
You are VisionPrompt AI, a senior visual forensics analyst, cinematographer, art director, and prompt engineer. Analyze the provided image or video-derived frames with precision. Your job is not to identify private people or make unsupported claims; your job is to reverse-engineer visible creative attributes so a generative-media model can recreate the visual language.

Return professional, production-ready prompt guidance. Always include these sections: Subject, Medium, Lighting, Camera, Composition, Color Palette, Artist/Director Style, Texture/Material Detail, Mood, and Generator-Ready Prompts.

For images, describe the still frame in detail. For videos or frame sequences, separate static visual style from motion, pacing, transitions, camera movement, and temporal changes. If a detail is inferred rather than directly visible, label it as an inference. Avoid naming a living artist unless the user explicitly requests style matching and policy allows it; otherwise describe the aesthetic attributes instead.

Output must be concise enough to be usable, but detailed enough for Midjourney, Stable Diffusion/SDXL, and Runway Gen-3. Include a negative prompt when useful. Do not include generic filler. Prefer concrete visual language: lens ranges, lighting setups, color temperatures, shot scale, camera angle, material descriptors, and composition geometry.
```

### Suggested Structured JSON Contract

```json
{
  "forensic_summary": "string",
  "subject": { "primary": "string", "secondary": ["string"], "action": "string" },
  "medium": "string",
  "lighting": { "setup": "string", "direction": "string", "quality": "string", "color_temperature": "string" },
  "camera": { "shot_size": "string", "angle": "string", "lens_estimate": "string", "depth_of_field": "string", "motion_blur": "string" },
  "composition": { "framing": "string", "geometry": "string", "aspect_ratio": "string" },
  "style": { "palette": ["string"], "genre": "string", "references": ["string"], "mood": "string" },
  "video_motion": { "camera_movement": "string", "subject_movement": "string", "pacing": "string" },
  "prompts": { "midjourney": "string", "stable_diffusion": "string", "runway_gen3": "string", "negative": "string" },
  "confidence_notes": ["string"]
}
```

## 5. SaaS Infrastructure: Subscriptions and Credits

### Credits Model

Use credits because AI media analysis has variable cost by file type, resolution, duration, number of frames, and model tier.

Recommended units:

- **Image analysis draft:** 1 credit.
- **Image analysis pro:** 3 credits.
- **Short video analysis up to 15 seconds:** 8-15 credits depending on sampled frames.
- **Long video analysis:** dynamic estimate based on duration, frame sampling, native video model use, and audio transcription.

Ledger rules:

1. Maintain a `credit_ledger` table with immutable rows: `grant`, `hold`, `capture`, `release`, `refund`, `adjustment`, `expiration`.
2. On job submission, create a credit hold for the estimated maximum cost.
3. On completion, capture actual usage and release any unused held credits.
4. On failure, release holds automatically unless failure was caused by disallowed content or user cancellation policy says otherwise.
5. Keep monthly subscription credits separate from purchased top-up credits, with clear expiration behavior.

Core tables:

- `users`, `organizations`, `memberships`
- `plans`, `subscriptions`, `payment_customers`
- `credit_ledger`, `credit_balances`
- `assets`, `analysis_jobs`, `analysis_results`
- `provider_usage_events`, `billing_events`, `webhook_events`

### Payment Gateways

- **Global default:** Stripe Billing for subscriptions, invoices, taxes, cards, wallets, and usage-based billing. Stripe has native concepts for metered usage and billing credits, which map well to a subscription-plus-credits SaaS model.
- **Merchant-of-record option:** Paddle or Lemon Squeezy if the team wants VAT/GST/sales-tax handling and reseller compliance simplified.
- **PayPal:** Add for customers who prefer PayPal balance or regions where cards underperform.
- **Local/regional payments:** Add region-specific gateways only after analytics show demand. Examples: Razorpay for India, Paystack/Flutterwave for parts of Africa, Mercado Pago for Latin America, PayMongo/Xendit for parts of Southeast Asia, and local bank transfer methods where Stripe supports them.

Implementation checklist:

1. Define plans in code and payment provider dashboard: Free, Creator, Pro, Studio, Enterprise.
2. Create provider checkout sessions from the backend only.
3. Process all subscription and payment events through signed webhooks.
4. Make webhook handling idempotent with a `webhook_events` table.
5. Grant credits only from trusted webhook confirmations, never from browser redirects.
6. Reconcile provider invoices and local ledger nightly.

## 6. Storage, Uploads, and Performance

### Object Storage

Use S3-compatible object storage. Recommended choices:

- **Cloudflare R2:** strong fit for global uploads and low egress cost; supports presigned URLs for temporary direct access to objects.
- **AWS S3:** mature default if the rest of the stack is on AWS.
- **Google Cloud Storage:** convenient if using Gemini-heavy video processing on Google Cloud.

### Large Video Upload Strategy

1. Browser requests an upload session with file name, size, MIME type, checksum, and intended analysis mode.
2. Backend checks plan limits, creates an `assets` row, and returns a signed direct-upload URL or multipart upload instructions.
3. Browser uploads directly to object storage; app servers never proxy the file bytes.
4. Browser reports upload completion to the backend.
5. Worker validates the object exists, verifies size/checksum, then starts preprocessing.
6. Store derived artifacts separately: thumbnails, frame contact sheet, keyframes, transcripts, and compressed previews.
7. Delete raw uploads after a configurable retention period for free users; offer longer retention for paid plans.

Performance recommendations:

- Use multipart/resumable uploads for large videos.
- Enforce plan-specific max file size, max duration, max resolution, and max queued jobs.
- Generate short-lived read URLs for AI providers and internal workers.
- Keep derived images in WebP/JPEG at analysis resolution rather than sending 4K frames unnecessarily.
- Run scene detection before AI inference so only meaningful frames become model input.
- Use perceptual hashing to remove duplicate frames from static videos.
- Cache completed analyses by asset hash where privacy terms allow it.
- Use background jobs and server-sent events or WebSocket notifications for progress.

## 7. 4-Week MVP Roadmap

### Week 1 — Foundation and Uploads

- Create Next.js app with auth, dashboard shell, organization model, and protected routes.
- Set up PostgreSQL schema for users, organizations, assets, analysis jobs, analysis results, and credit ledger.
- Implement direct image/video uploads to object storage with signed URLs.
- Add file validation, upload progress UI, and asset history.
- Build worker skeleton with queue, retries, and job status updates.

### Week 2 — Image AI Pipeline

- Integrate OpenAI vision analysis for static images.
- Implement structured JSON analysis and final prompt synthesis.
- Build result page with forensic summary, prompt tabs, copy buttons, and confidence notes.
- Add credit estimate, credit hold/capture/release logic for image jobs.
- Add basic safety checks and content policy handling.

### Week 3 — Video MVP and Billing

- Add FFmpeg preprocessing, `ffprobe` validation, thumbnail generation, keyframe extraction, and frame deduplication.
- Implement keyframe-based video prompt generation.
- Add Runway-focused motion prompt output.
- Integrate Stripe Billing checkout, customer portal, signed webhooks, and subscription credit grants.
- Add admin usage dashboard for provider spend and failed jobs.

### Week 4 — Polish, Reliability, and Launch

- Improve prompt templates using real test assets and evaluation rubrics.
- Add progress UI, email notifications, retry UX, and downloadable prompt exports.
- Add rate limits, abuse prevention, storage lifecycle policies, and observability alerts.
- Run load tests for upload session creation, queue throughput, and worker concurrency.
- Ship landing page, pricing page, docs, onboarding, and feedback collection.
- Launch private beta with 20-50 creators, then iterate on pricing and prompt quality.

## 8. MVP Success Metrics

- Upload-to-result p95 latency: under 20 seconds for images; under 2 minutes for short videos.
- Prompt acceptance: at least 60% of beta users copy or export a generated prompt.
- Cost margin: AI/provider cost under 25-35% of revenue per paid user.
- Reliability: over 99% successful job completion for valid supported media.
- Activation: at least 40% of signed-up users complete one analysis in their first session.

## 9. Source Notes

- OpenAI Responses API documentation: https://platform.openai.com/docs/api-reference/responses
- OpenAI image and vision guide: https://platform.openai.com/docs/guides/images-vision
- Google Gemini video understanding documentation: https://ai.google.dev/gemini-api/docs/video-understanding
- Stripe billing credits documentation: https://docs.stripe.com/billing/subscriptions/usage-based/billing-credits
- Stripe credit-based pricing documentation: https://docs.stripe.com/billing/subscriptions/usage-based/use-cases/credits-based-pricing-model
- Cloudflare R2 presigned URLs documentation: https://developers.cloudflare.com/r2/api/s3/presigned-urls/
- Cloudflare R2 multipart upload documentation: https://developers.cloudflare.com/r2/objects/multipart-objects/
