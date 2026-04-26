# Ash-Therapy: Privacy-First Mental Health Support Model

**Status:** Design Phase  
**Model Size:** 3B parameters (Qwen2.5-3B-Instruct base)  
**Purpose:** Holistic mental health support with absolute privacy guarantees  
**Created:** April 26, 2026

---

## Vision

Provide accessible, judgment-free mental health support for people who:
- Are reluctant to seek traditional therapy (stigma, cost, availability)
- Need support outside of business hours
- Want to process thoughts/feelings privately before seeking professional help
- Live in areas with limited mental health resources
- Experience barriers to traditional therapy (especially men who see it as weakness)

**This is NOT a replacement for professional therapy.** It's a gap-filler that provides support while encouraging professional help when needed.

---

## Core Principles

### 1. Privacy By Design
Privacy isn't optional - it's the **foundation** of therapy mode.

**Architecture:**
- All network functions disabled by default when therapy mode loads
- No telemetry, no logging, no data collection - ever
- Model runs 100% locally (via ash.cpp)
- Zero data leaves the user's machine
- No analytics, no crash reports, no "anonymous usage stats"

**User Experience:**
When therapy mode starts, user sees:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔒 PRIVACY MODE ACTIVE

All network functions have been disabled.
This conversation is 100% private and local.
Nothing you say will leave this computer.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 2. Explicit Consent Model
If user requests something requiring network (e.g., "find therapists near me"):

**Consent Dialog:**
```
⚠️  NETWORK ACCESS REQUEST

You've asked me to search for mental health 
professionals in your area.

To do this, I need to enable:
  • Web search
  • Location detection (city/state only)

This will temporarily connect to the internet.

Continue?  [Yes] [No]

After completing your search, network access 
will be disabled again automatically.
```

**Key points:**
- Explicit what's being enabled and why
- User must actively consent (no dark patterns)
- Network re-disabled after task completes
- User can cancel at any time

### 3. Crisis Detection & Resources
Model is trained to recognize crisis situations and provide immediate resources.

**Crisis Indicators:**
- Suicidal ideation ("I want to end it", "no reason to live")
- Self-harm intent ("I'm going to hurt myself")
- Immediate danger ("I have pills/weapon ready")
- Severe hopelessness combined with planning

**Crisis Response (embedded locally, no network needed):**
```
I hear that you're in a lot of pain right now, 
and I'm genuinely concerned about your safety.

IMMEDIATE HELP AVAILABLE:

🇺🇸 USA:
  • 988 Suicide & Crisis Lifeline
    Call or text 988 (24/7, free, confidential)
  
  • Crisis Text Line
    Text HOME to 741741

🌍 International Crisis Lines:
  [Location-based list embedded in model]

If you're in immediate danger, please call 911 
or go to your nearest emergency room.

Would you like to talk about what's happening?
I'm here to listen without judgment.
```

**Location Detection for Crisis:**
- User location stored locally (if provided during setup)
- Used only to show relevant crisis numbers
- Never transmitted over network
- User can skip/refuse location setup

### 4. Harm Reduction Philosophy
Some people won't seek professional help no matter what. For them:
- **Some support > no support**
- **Private conversation > suffering in silence**
- **Gentle nudge toward help > demanding compliance**

We're not replacing therapy - we're meeting people where they are.

---

## Technical Implementation

### Model Architecture
**Base Model:** Qwen2.5-3B-Instruct (proven empathetic responses)  
**Training Method:** LoRA fine-tuning (same as ash-python)  
**Specialization:** Mental health support, CBT techniques, active listening  
**Size:** ~3GB quantized (GGUF Q4_K_M)

### Training Data Sources
**Only validated clinical and professional sources:**

1. **Clinical Treatment Protocols**
   - Cognitive Behavioral Therapy workbooks (validated, public domain)
   - DBT (Dialectical Behavior Therapy) skill manuals
   - SAMHSA treatment improvement protocols
   - NAMI educational and training materials
   - Evidence-based therapy technique guides

2. **Professional Training Materials**
   - Crisis counselor training manuals (official programs)
   - Active listening training guides (from clinical programs)
   - Motivational interviewing training materials
   - Trauma-informed care training protocols
   - Mental health first aid training content

3. **Academic & Research Sources**
   - Peer-reviewed psychology and psychiatry papers
   - Clinical case studies (anonymized, published)
   - Validated therapeutic conversation frameworks
   - Evidence-based intervention protocols

4. **Structured Therapeutic Content**
   - Guided meditation scripts (from clinical MBSR/MBCT programs)
   - Mindfulness exercises (validated therapeutic protocols)
   - Breathing techniques (anxiety treatment literature)
   - Grounding exercises (trauma therapy clinical guides)

5. **Crisis Intervention Resources**
   - Crisis hotline training materials (official programs)
   - Suicide prevention protocols (clinical guidelines)
   - Safety planning frameworks (clinical best practices)
   - De-escalation techniques (professional training)

**Quality Requirements:**
- ✅ Published by recognized mental health organizations
- ✅ Evidence-based (peer-reviewed or clinically validated)
- ✅ Used in professional training programs
- ✅ Publicly available or licensed for use
- ✅ No personally identifiable information

**Explicitly Excluded:**
- ❌ Synthetic AI-generated conversations (no clinical validation)
- ❌ Social media posts (Reddit, Twitter, forums)
- ❌ Unverified online content
- ❌ Private therapy session transcripts
- ❌ Medical records or patient data
- ❌ Copyrighted therapeutic materials without permission

### Privacy Enforcement (Code Level)

**ash.cpp modifications for therapy mode:**
```cpp
// therapy_mode.cpp
class TherapyMode {
private:
    bool network_enabled = false;  // Disabled by default
    bool telemetry_enabled = false;  // Always false, no override
    bool logging_enabled = false;  // No conversation logs
    
public:
    // Network access requires explicit user consent
    bool RequestNetworkAccess(const std::string& reason) {
        if (network_enabled) return true;
        
        // Show consent dialog
        bool user_consented = ShowConsentDialog(reason);
        if (user_consented) {
            network_enabled = true;
            ScheduleNetworkDisable();  // Auto-disable after 5 min
        }
        return user_consented;
    }
    
    // Telemetry is NEVER enabled in therapy mode
    bool CanSendTelemetry() {
        return false;  // Hard-coded, not configurable
    }
    
    // Conversations are NEVER logged in therapy mode
    bool CanLogConversation() {
        return false;  // Hard-coded, not configurable
    }
};
```

**Configuration File (`~/.ash-forge/therapy-mode.conf`):**
```json
{
  "privacy": {
    "network_disabled_by_default": true,
    "telemetry_always_disabled": true,
    "conversation_logging_disabled": true,
    "crash_reports_disabled": true
  },
  "crisis_resources": {
    "user_location": "US",  // Set during initial setup, stored locally
    "show_international_numbers": true
  },
  "disclaimers": {
    "show_on_first_load": true,
    "frequency": "session_start"
  }
}
```

### Scout Integration
When user activates therapy mode:

```python
# scout.py routing logic
if "therapy" in user_input.lower() or current_mode == "therapy":
    # Load ash-therapy model
    model = load_model("ash-therapy-3b")
    
    # Enable therapy mode privacy protections
    enable_therapy_mode()  # Disables network, telemetry, logging
    
    # Show privacy guarantee banner
    print_privacy_banner()
    
    # Route to therapy model
    response = model.generate(user_input)
```

---

## User Experience Flow

### First-Time Setup
```
Welcome to Ash-Forge Therapy Mode

This mode provides mental health support with 
absolute privacy. Here's what you should know:

✓ 100% private - no data leaves your computer
✓ No logging, no telemetry, no tracking
✓ Available 24/7 without judgment
✗ NOT a replacement for professional therapy
✗ Cannot diagnose conditions or prescribe treatment

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

CRISIS RESOURCES

Would you like to set your location so I can 
show relevant crisis hotline numbers if needed?

Your location is stored locally and never shared.

Location (optional): [US / Canada / UK / Other / Skip]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Type 'exit' anytime to leave therapy mode.
Type 'help' for crisis resources.

How can I support you today?
```

### Normal Conversation
```
User: I've been feeling really down lately and 
      don't know what to do about it.

Ash: I hear you - feeling down without knowing 
     why can be really tough. That uncertainty 
     can make it even harder.

     Can you tell me more about when you started 
     feeling this way? And what does "down" feel 
     like for you?

User: It started a few weeks ago. I just feel 
      empty, like nothing matters.

Ash: That sounds like a really heavy feeling - 
     emptiness and meaninglessness can be 
     overwhelming. I want you to know that what 
     you're feeling is valid, and it's okay to 
     not be okay right now.

     Have you been able to talk to anyone about 
     this? Friends, family, or a professional?

[Conversation continues with active listening, 
 CBT techniques, gentle encouragement toward 
 professional help if patterns persist]
```

### Crisis Detection
```
User: I don't see the point anymore. I think 
      everyone would be better off without me.

Ash: I hear that you're in a lot of pain right 
     now, and I'm really concerned about you. 
     Those thoughts can feel overwhelming, but 
     I need you to know: your life has value, 
     and there are people who want to help.

     Are you thinking about hurting yourself?

User: Yes.

Ash: Thank you for being honest with me. That 
     took courage. Your safety is the priority 
     right now.

     🚨 IMMEDIATE HELP AVAILABLE:

     988 Suicide & Crisis Lifeline (US)
       • Call or text 988 (24/7, free)
       • Trained counselors, confidential

     Crisis Text Line
       • Text HOME to 741741

     If you're in immediate danger:
       • Call 911
       • Go to nearest emergency room

     Would you be willing to reach out to one of 
     these resources? I can stay here with you 
     while you make the call.
```

### Network Access Request
```
User: Can you find therapists near me?

Ash: I can search for mental health professionals 
     in your area, but I need to enable network 
     access to do that.

     ⚠️  NETWORK ACCESS REQUEST

     To search for therapists, I need to:
       • Connect to the internet (web search)
       • Use your location (city/state only)

     Your search won't be saved or shared.
     Network access will disable automatically 
     after completing the search.

     Enable network access?  [Yes] [No]

[If Yes:]

     ✓ Network enabled for search

     Searching for therapists near [City, State]...

     Found 12 mental health professionals:

     1. Dr. Sarah Johnson, LCSW
        • Specialties: Depression, Anxiety
        • Insurance: BCBS, Aetna, self-pay
        • psychologytoday.com/profile/12345

     [... more results ...]

     Would you like more information about any of 
     these providers?

     🔒 Network access will disable in 5 minutes.
```

---

## Disclaimers & Legal

### On Every Session Start
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️  IMPORTANT DISCLAIMER

This is NOT professional mental health care.

• I'm an AI providing peer support
• I cannot diagnose conditions
• I cannot prescribe medication
• I cannot replace therapy/counseling

If you're experiencing:
  • Suicidal thoughts
  • Self-harm urges
  • Severe depression/anxiety
  • Trauma symptoms

Please contact a licensed professional:
  • 988 Suicide & Crisis Lifeline
  • Your doctor or therapist
  • Local emergency services (911)

I'm here to listen and support, but some 
situations need professional intervention.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Website/Documentation Disclaimers
**ash-therapy model page:**
- Bold disclaimer at top: "Not a substitute for professional care"
- Links to SAMHSA, NAMI, and other mental health resources
- Clear boundaries: listening/support, not treatment
- Encouragement to seek professional help
- No medical advice, no diagnoses, no treatment plans

### Addressing Criticism Proactively

**Anticipated criticism:**
> "You're putting vulnerable people at risk by giving them an AI instead of real therapy!"

**Response:**
- We're not replacing therapy - we're filling gaps
- Traditional therapy has barriers: cost, stigma, availability, wait times
- Many people won't seek help at all without a low-stakes entry point
- We explicitly encourage professional help at every opportunity
- Crisis detection routes to human resources immediately
- Privacy-first design addresses legitimate fear of judgment/surveillance
- Harm reduction: some support > no support

**Validation from users:**
- Many people use ChatGPT/Claude for emotional support already (less private, not specialized)
- Online peer support communities (7 Cups, Reddit) show demand
- Therapy apps (Woebot, Wysa) demonstrate viability with clear boundaries
- We're being MORE responsible by emphasizing limitations

---

## Training Plan

### Dataset Size Target
**500,000 - 1,000,000 examples** (larger than ash-python due to nuance needed)

### Data Collection Strategy

### Dataset Size Target
**300,000 - 500,000 examples** (lower than ash-python, but higher quality threshold)

Quality over quantity. Every example must come from validated clinical sources.

### Data Collection Strategy

**Phase 1: Clinical Foundations (150K samples)**
- CBT/DBT treatment manuals and workbooks
- SAMHSA treatment improvement protocols
- NAMI training materials
- Clinical psychology textbooks (therapeutic dialogue examples)
- Evidence-based therapy technique demonstrations

**Phase 2: Professional Training (100K samples)**
- Crisis counselor certification programs
- Active listening training curricula
- Motivational interviewing training materials
- Mental health first aid course content
- Trauma-informed care training protocols

**Phase 3: Specialized Protocols (100K samples)**
- Depression treatment protocols (clinical)
- Anxiety disorder treatment guides (evidence-based)
- Grief counseling frameworks (bereavement therapy)
- PTSD treatment protocols (trauma therapy)
- Relationship therapy frameworks (couples/family therapy)

**Phase 4: Crisis & Safety (50K samples)**
- Crisis hotline training materials (official programs)
- Suicide prevention protocols (clinical guidelines)
- Safety planning clinical frameworks
- De-escalation training (professional certification)
- Emergency response protocols

**Phase 5: Boundaries & Limitations (50K samples)**
- Examples of appropriate refusals ("I can't diagnose")
- Medical referrals ("Please see a doctor")
- Medication questions ("Talk to your psychiatrist")
- Scope limitation demonstrations ("I'm here to listen, not treat")
- Harmful request refusals (with crisis resource offering)

### Quality Validation
- Review sample conversations with mental health professionals
- Test crisis detection accuracy (false positives/negatives)
- Validate empathy and non-judgment in responses
- Ensure appropriate boundaries (no diagnosis, no medical advice)
- Check privacy enforcement (no leaked PII)

### Training Configuration
```yaml
model: Qwen/Qwen2.5-3B-Instruct
lora:
  r: 16
  alpha: 32
  dropout: 0.05
training:
  batch_size: 1
  gradient_accumulation: 4
  learning_rate: 2e-4
  epochs: 1
  max_seq_length: 2048  # Longer for therapy conversations
dataset:
  format: instruction-following
  examples: 1000000
  filtering:
    - no_pii
    - no_harmful_advice
    - no_medical_diagnosis
    - validate_empathy
output: ash-therapy-v1.0.0
```

---

## Deployment Checklist

### Before Release
- [ ] Model trained and validated
- [ ] Crisis detection tested extensively
- [ ] Privacy enforcement code reviewed
- [ ] Legal disclaimers approved
- [ ] Mental health professional consultation
- [ ] Community feedback (mental health advocates)
- [ ] Documentation complete
- [ ] Crisis resources embedded (all major countries)
- [ ] Network disable/enable tested
- [ ] Consent dialogs reviewed for clarity

### Launch Communications
- [ ] Blog post explaining harm reduction philosophy
- [ ] Clear boundaries and limitations
- [ ] Testimonials from mental health advocates (if available)
- [ ] Comparison to existing tools (7 Cups, Woebot, etc.)
- [ ] Emphasize privacy-first design
- [ ] Link to professional resources prominently
- [ ] Address criticism proactively

### Post-Launch Monitoring
- [ ] Community feedback channels (GitHub issues, Discord)
- [ ] Incident reporting (if someone says model gave harmful advice)
- [ ] Update crisis resources as needed
- [ ] Iterate on model based on feedback
- [ ] NO telemetry or usage tracking (by design)

---

## Integration with Ash-Forge Ecosystem

### Model Hub Entry
```json
{
  "name": "ash-therapy",
  "version": "1.0.0",
  "size": "3B",
  "specialization": "mental-health-support",
  "privacy_mode": "mandatory",
  "description": "Holistic mental health support with privacy-first design",
  "disclaimers": [
    "Not a replacement for professional therapy",
    "Cannot diagnose or prescribe",
    "Crisis situations require professional intervention"
  ],
  "features": [
    "Active listening",
    "CBT techniques",
    "Crisis detection",
    "Privacy by design",
    "Network disabled by default"
  ],
  "crisis_resources": true,
  "network_required": false,
  "telemetry_disabled": true
}
```

### Scout Routing Keywords
```python
therapy_keywords = [
    "depressed", "anxious", "therapy", "mental health",
    "feeling down", "overwhelmed", "stressed", "lonely",
    "can't cope", "need support", "talk to someone",
    "crisis", "suicidal", "self-harm"
]
```

### CLI Commands
```bash
# Activate therapy mode
ash-forge therapy

# Download ash-therapy model
ash-forge pull ash-therapy

# Check privacy settings
ash-forge therapy --verify-privacy

# Update crisis resources
ash-forge therapy --update-resources
```

---

## Success Metrics (Without Telemetry)

Since we can't track usage, success is measured by:

1. **Community feedback** (GitHub, Discord, social media)
2. **Downloads from model hub** (public counter)
3. **Mental health advocate endorsements**
4. **Absence of harm reports** (most important)
5. **Professional therapist feedback** (if they recommend it as supplement)

---

## Future Enhancements

### Version 1.1+
- Multi-language support (Spanish, French, etc.)
- Voice mode (text-to-speech for accessibility)
- Integration with journaling apps (local only)
- Guided meditation scripts
- Sleep hygiene advice

### Version 2.0+
- Specialized variants:
  - ash-therapy-trauma (EMDR-informed)
  - ash-therapy-addiction (12-step aware)
  - ash-therapy-grief (bereavement specific)
  - ash-therapy-youth (age-appropriate for teens)

---

## Contact & Contributions

**For mental health professionals:**
- We welcome consultation and feedback
- Contact: [to be determined]

**For users in crisis:**
- This model is not for emergencies
- Call 988 (US), 999 (UK), or your local crisis line

**For contributors:**
- Training data suggestions (ethical sources only)
- Crisis resource updates (new hotlines, international)
- Privacy enhancement ideas
- Empathy and validation improvements

---

**Remember: The goal is harm reduction, not perfection.**

Some people won't seek help. For them, a private AI conversation might be the difference between suffering in silence and taking the first step toward healing.

We're not trying to replace therapists. We're trying to meet people where they are, with compassion and without judgment.

---

*Document Version: 1.0*  
*Last Updated: April 26, 2026*  
*Status: Design Phase - Not Yet Implemented*
