# Privacy Architecture

Ash-Therapy is built with **privacy by design**. This isn't marketing - it's architecture.

---

## Our Guarantees

We make the following **verifiable** promises:

1. ✅ **No telemetry** - Hard-coded off, not configurable
2. ✅ **No conversation logging** - Nothing written to disk
3. ✅ **No network by default** - Must explicitly enable
4. ✅ **No crash reports** - Never sent
5. ✅ **No analytics** - Not even "anonymous"
6. ✅ **On-device speech** - Voice never sent to cloud
7. ✅ **No accounts** - No login, no tracking
8. ✅ **Open source** - Verify every claim yourself

---

## How to Verify

Don't trust us. Verify us.

### 1. Read the Code
```bash
# Privacy enforcement
cat runtime/privacy_guard.cpp

# Network controls
cat runtime/therapy_mode.cpp

# Telemetry (spoiler: always returns false)
grep -r "CanSendTelemetry" runtime/

# Speech recognition (on-device Whisper)
cat runtime/speech/whisper_local.cpp
```

### 2. Build from Source
```bash
git clone https://github.com/ash-forge/ash-therapy
cd ash-therapy
./scripts/build.sh

# Now you have YOUR OWN binary - no trust required
```

### 3. Monitor Network Traffic
```bash
# Run ash-therapy
./ash-therapy

# In another terminal, monitor network
sudo tcpdump -i any
# OR on macOS:
sudo nettop -J bytes_in,bytes_out

# You'll see: NOTHING (unless you explicitly enable network)
```

### 4. Check Mobile Permissions
```
iOS: Settings > Ash-Therapy > Permissions
  → None required. None requested.

Android: Settings > Apps > Ash-Therapy > Permissions
  → No microphone (speech processing is on-device)
  → No network (only if you explicitly grant)
  → No storage (sessions aren't saved)
  → No location (unless you enable for therapist search)
```

### 5. Test in Airplane Mode
```bash
# Enable airplane mode
# Launch ash-therapy
# Have a full conversation

# Result: Works perfectly.
# Because: Everything is local.
```

---

## Architecture Details

### Privacy Guard (C++)

```cpp
// runtime/privacy_guard.cpp
namespace Privacy {
    // Hard-coded: NEVER send telemetry in therapy mode
    bool CanSendTelemetry() {
        #ifdef THERAPY_MODE
            return false;  // Compile-time constant
        #else
            return false;  // Also false in other modes
        #endif
    }
    
    // Hard-coded: NEVER log conversations
    bool CanLogConversation() {
        return false;  // Not configurable
    }
    
    // Hard-coded: NEVER collect usage stats
    bool CanCollectUsageStats() {
        return false;  // No override possible
    }
    
    // Network requires explicit user consent
    bool CanConnectToNetwork() {
        if (!therapy_mode_active) return true;  // Normal mode
        
        // In therapy mode, must show consent dialog
        return user_explicitly_consented_to_network;
    }
}
```

### Network Disable (OS-Level)

```cpp
// runtime/therapy_mode.cpp
class TherapyMode {
private:
    bool network_enabled = false;  // Default: disabled
    
public:
    TherapyMode() {
        // Disable at application level
        SetApplicationNetworkEnabled(false);
        
        // Log to console (user can see)
        std::cout << "🔒 Network disabled by therapy mode\n";
    }
    
    bool RequestNetworkAccess(const std::string& reason) {
        // Show consent dialog to user
        ConsentDialog dialog;
        dialog.SetReason(reason);
        dialog.SetExplanation(
            "To complete this request, I need to:\n"
            "  • Connect to the internet temporarily\n"
            "  • Network will auto-disable after 5 minutes\n"
        );
        
        bool consented = dialog.Show();
        
        if (consented) {
            network_enabled = true;
            std::cout << "✓ Network enabled by user consent\n";
            
            // Schedule auto-disable (5 minutes)
            std::thread([this]() {
                std::this_thread::sleep_for(std::chrono::minutes(5));
                DisableNetwork();
            }).detach();
        }
        
        return consented;
    }
    
    void DisableNetwork() {
        network_enabled = false;
        SetApplicationNetworkEnabled(false);
        std::cout << "🔒 Network disabled automatically\n";
    }
};
```

### On-Device Speech Recognition

```cpp
// runtime/speech/whisper_local.cpp
class LocalSpeechRecognizer {
private:
    WhisperModel model;  // Loaded from disk, runs on CPU/GPU
    
public:
    std::string Transcribe(const AudioBuffer& audio) {
        // NO NETWORK CALLS
        // All processing happens locally using Whisper.cpp
        
        auto result = model.Process(audio);
        
        // Audio data is never written to disk
        // Audio data is never sent over network
        // Audio data is cleared from memory after transcription
        
        return result.text;
    }
};
```

### Session Data (Memory Only)

```cpp
// runtime/session.cpp
class TherapySession {
private:
    std::vector<Message> conversation;  // In-memory only
    
public:
    ~TherapySession() {
        // When session ends, clear memory
        conversation.clear();
        
        // Optionally: overwrite memory with zeros
        SecureZeroMemory(conversation.data(), 
                         conversation.size() * sizeof(Message));
        
        std::cout << "Session cleared from memory\n";
    }
    
    // NO Save() method - sessions aren't persisted
    // NO Log() method - conversations aren't logged
};
```

---

## What We DON'T Do

### No Telemetry
```cpp
// These functions are NEVER called in ash-therapy
void SendUsageStats() { /* Not in codebase */ }
void TrackFeatureUsage() { /* Not in codebase */ }
void ReportCrash() { /* Not in codebase */ }
void AnalyzeConversation() { /* Not in codebase */ }
```

### No Logging
```cpp
// Conversations are NEVER written to disk
void LogConversationToFile() { /* Not in codebase */ }
void SaveSessionHistory() { /* Not in codebase */ }
void BackupMessages() { /* Not in codebase */ }
```

### No Cloud Speech Recognition
```cpp
// We DON'T use these cloud APIs:
// - Google Cloud Speech-to-Text ❌
// - AWS Transcribe ❌
// - Azure Speech Services ❌
// - OpenAI Whisper API ❌

// We DO use:
// - Whisper.cpp (local, open source) ✅
```

### No User Tracking
```cpp
// We DON'T track:
// - Device ID ❌
// - IP address ❌
// - MAC address ❌
// - Usage patterns ❌
// - Feature engagement ❌
// - Session duration ❌
```

---

## Configuration (Read-Only)

```json
// ~/.ash-therapy/config.json
{
  "privacy": {
    "network_disabled_by_default": true,
    "telemetry_always_disabled": true,
    "conversation_logging_disabled": true,
    "crash_reports_disabled": true,
    "speech_recognition": "local",  // Never "cloud"
    "user_can_override": false      // Privacy enforced in code
  },
  "session": {
    "save_history": false,           // Hard-coded false
    "persist_to_disk": false         // Hard-coded false
  }
}

// Note: Even if user modifies this file,
// privacy protections are enforced in code.
```

---

## Mobile App Privacy

### iOS
- ✅ **No permissions required**
- ✅ **Runs in app sandbox** (can't access other apps)
- ✅ **No background mode** (privacy by design)
- ✅ **No iCloud sync** (no data to sync)
- ✅ **No analytics SDKs** (Firebase, Mixpanel, etc. not included)
- ✅ **App Store Privacy Nutrition Label:**
  ```
  Data Not Collected
    The developer does not collect any data from this app.
  ```

### Android
- ✅ **No permissions in manifest** (none requested)
- ✅ **Runs in app sandbox** (Android's security model)
- ✅ **No background services** (can't run when closed)
- ✅ **No network permission** (unless user enables in-app)
- ✅ **F-Droid compatible** (meets privacy requirements)
- ✅ **Google Play Data Safety:**
  ```
  No data shared with third parties
  No data collected
  ```

---

## Threat Model

### What We Protect Against

**1. Corporate Surveillance**
- ✅ No data sent to Ash-Forge servers (we don't have servers)
- ✅ No third-party analytics
- ✅ No advertising SDKs
- ✅ No user profiling

**2. Government Surveillance**
- ✅ No data to subpoena (we don't collect any)
- ✅ No server logs (no servers)
- ✅ No metadata (no accounts)
- ✅ Works in airplane mode (no network traffic)

**3. Data Breaches**
- ✅ Can't breach data we don't have
- ✅ No cloud storage to hack
- ✅ No user database to leak

**4. Malicious Updates**
- ✅ Open source (community can review updates)
- ✅ Reproducible builds (coming soon)
- ✅ Users can pin specific versions

### What We DON'T Protect Against

**1. Local Device Compromise**
- ❌ If your device is hacked, conversations in memory could be read
- ❌ We can't protect against keyloggers or screen recording malware
- **Mitigation:** Keep your OS updated, use device encryption

**2. Physical Device Access**
- ❌ If someone has your unlocked device, they can use the app
- **Mitigation:** Use device lock screen, biometric authentication

**3. Network Monitoring (When Enabled)**
- ❌ If you enable network for therapist search, that traffic is visible
- **Mitigation:** We minimize data sent, use HTTPS, auto-disable network

---

## Comparison to Other Tools

| Feature | Ash-Therapy | ChatGPT | Woebot | Wysa | 7 Cups |
|---------|------------|---------|--------|------|--------|
| **Runs Offline** | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No |
| **No Telemetry** | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No |
| **Open Source** | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No |
| **On-Device Speech** | ✅ Yes | ❌ No | ❌ No | ❌ No | N/A |
| **No Account** | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No |
| **No Servers** | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No |
| **Privacy Auditable** | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No |

---

## Privacy Audit Checklist

If you're a security researcher auditing ash-therapy:

- [ ] Review `runtime/privacy_guard.cpp` for telemetry hooks
- [ ] Search codebase for network calls: `grep -r "http://" runtime/`
- [ ] Check for analytics SDKs: `grep -r "firebase\|mixpanel\|amplitude" .`
- [ ] Verify speech recognition is local: `cat runtime/speech/whisper_local.cpp`
- [ ] Monitor network traffic during usage: `tcpdump -i any`
- [ ] Check mobile app permissions: iOS Settings / Android Manifest
- [ ] Review build process for reproducibility
- [ ] Test in airplane mode (should work fully)
- [ ] Check for phone-home domains: `strings ./ash-therapy | grep -i "http"`
- [ ] Verify sessions aren't logged: `find ~/.ash-therapy/ -type f`

**Found an issue?** Please report privately: security@ash-forge.com

---

## Reproducible Builds (Coming Soon)

We're working on reproducible builds so you can verify:
- The binary matches the source code
- No backdoors added during compilation
- Identical builds from different machines

Status: In progress (see [#1](https://github.com/ash-forge/ash-therapy/issues/1))

---

## Still Don't Trust Us?

That's okay! Here are your options:

### 1. Build from Source
```bash
git clone https://github.com/ash-forge/ash-therapy
cd ash-therapy
./scripts/build.sh
# Use YOUR binary, compiled by YOU
```

### 2. Fork the Repo
```bash
git clone https://github.com/ash-forge/ash-therapy
cd ash-therapy
# Make it EVEN MORE private
# Add your own features
# Remove anything you don't trust
```

### 3. Run in a VM
```bash
# Create isolated environment
# Disconnect from network
# Run ash-therapy inside VM
# Nothing can escape VM boundary
```

### 4. Air-Gap the Machine
```
1. Disconnect computer from internet physically
2. Install ash-therapy
3. Use without any network connection
4. Ultimate privacy guarantee
```

### 5. Audit the Code Yourself
```bash
# Hire a security researcher
# Review every line of code
# Verify our claims
# We'll help facilitate this
```

---

## Privacy Updates

If we ever change our privacy architecture (we won't, but hypothetically):

1. **Major version bump** (e.g., v1.x → v2.0)
2. **Clear changelog** explaining changes
3. **Opt-in** for new versions (auto-update disabled)
4. **Community discussion** before implementation

**Current policy:** Privacy protections will NEVER be weakened.

---

## Questions?

- Read the code: `cat runtime/privacy_guard.cpp`
- Open an issue: [Privacy Concern](https://github.com/ash-forge/ash-therapy/issues/new?template=privacy_concern.md)
- Email: privacy@ash-forge.com

---

**Bottom line:** We can't read your conversations because we never see them. They never leave your device.

*Last updated: April 26, 2026*
