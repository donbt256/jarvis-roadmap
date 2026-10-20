# Jarvis Whole-Home Voice AI & Multiroom Audio — Roadmap

**Status:** Planning / architecture validation  
**Last updated:** 2026-10-20  
**Repository:** [donbt256/jarvis-roadmap](https://github.com/donbt256/jarvis-roadmap)

This is a staged engineering plan, not a final bill of materials. Validate the highest-risk parts—voice latency, synchronized audio, independent music streams, amplifier topology, and local AI performance—before scaling to all 12 zones.

## 1. Project goal

Build a locally operated, whole-home assistant called **Jarvis** that can:
- Respond to voice requests from six microphone-equipped zones.
- Control smart-home devices through a constrained integration layer.
- Play music in any combination of 12 audio zones, including synchronized groups and independent streams.
- Use a shared household music-taste profile while allowing independent playback sessions.
- Run speech recognition, language models, and speech synthesis locally wherever practical.
- Use a fast model for routine requests and a stronger model for complex reasoning.
- Start speaking before the language model has generated the entire response.
- Accept “think harder” to reconsider the original request with a stronger model.
- Centralize amplifiers, server, network gear, and storage in a rack; keep ceiling locations limited to passive speakers and appropriate endpoint wiring.

The goal is a robust home system, not a one-off demo. Build and test one zone first, then expand in controlled increments.

## 2. Requirements and design baseline

| Item | Baseline |
|---|---|
| Total audio zones | 12 |
| Voice-enabled zones | 6 |
| Speaker-only zones | 6 |
| Ceiling speakers per zone | 4 |
| Total ceiling speakers | 48 |
| Stereo arrangement per zone | 2 left speakers + 2 right speakers |
| Central amplifier channels | 24 channels, one left/right pair per zone |
| Initial budget | $5,000+; final budget requires quotes and a complete bill of materials |
| Local AI | Required design goal |
| UPS | Not planned for initial build; revisit if uptime requirements change |
| Installation approach | Mixed off-the-shelf electronics and custom integration |

### Electrical/audio constraint

The proposed zone wiring places two nominal 8-ohm speakers in parallel on each channel, giving a nominal 4-ohm load per channel. This is acceptable only if the selected amplifier is explicitly rated for that load and the speaker/amp combination is validated. Do not connect all four 8-ohm speakers in parallel to one ordinary amplifier channel: that would create a nominal 2-ohm load.

The final design must account for actual speaker impedance curves, amplifier minimum-load ratings, output topology, cable lengths, and required sound-pressure levels. Some Class-D amplifiers use bridged or otherwise non-common outputs; never assume speaker negative terminals can be tied together.

## 3. Proposed system architecture

### Logical flow

~~~text
[6 voice endpoints]
  mic array -> wake word -> audio capture / VAD
                    |
                    v
             [Central voice API]
                    |
        +-----------+-----------+
        |           |           |
      STT       dialogue      tool router
        |         manager        |
        |      fast or strong  Home Assistant /
        |          LLM          Music Assistant /
        |                         safe services
        +------> response text
                    |
              streaming TTS
                    |
             speech playback
                    |
          selected room / group

[Music sources] -> Music Assistant / playback services
                              |
                    zone/group routing
                              |
                   24-channel output path
                              |
                 24 amp channels (12 stereo zones)
                              |
                    48 ceiling speakers
~~~

This is a logical architecture; exact service boundaries and audio transports should be decided after the one-zone prototype. Keep conversational speech routing separable from music distribution so synchronization buffers do not force unnecessary latency on voice responses.

### Central rack

Plan for a rack approximately 48U if space permits, with roughly 1,000–1,200 mm usable depth as a starting point for a GPU server, rails, rear cabling, and airflow. Confirm dimensions against the actual chassis and rack before purchasing. Rack height is not itself a technical requirement; service access, ventilation, noise, weight, and cable management matter more.

Likely rack equipment:
- GPU-capable AI server.
- Network switch/router and patch panels as needed.
- Multichannel audio output hardware and amplifiers.
- Storage for service configuration, music metadata, backups, and logs.
- Power distribution, surge protection, cable management, and labeled termination panels.

Keep powered equipment and serviceable electronics in accessible locations. Do not put improvised mains-powered amplifier assemblies above ceilings.

### Compute baseline

- Prefer an NVIDIA RTX 4090-class GPU with 24 GB VRAM for the first serious local inference server if price and availability are reasonable. The 24 GB capacity is materially more flexible than the RTX 5080's 16 GB for local models.
- 64 GB system RAM is a practical minimum; 96–128 GB gives more headroom for concurrent services, CPU offload, and larger models.
- Select CPU, motherboard, PSU, chassis, and cooling as a complete system. Validate sustained GPU power, acoustic output, thermal behavior, and physical rack fit.
- Do not assume a 70B-class model will run fully in 24 GB VRAM at useful quantization and context. Target well-performing smaller models first and benchmark actual workloads.
- Treat 30–50 generated tokens/second as an initial conversational-performance target, not a guarantee. Measure time-to-first-token and end-to-end spoken latency, not only raw token throughput.

## 4. Software stack: candidates, not locked choices

| Function | Initial candidate | Decision criteria |
|---|---|---|
| Smart-home state and automations | Home Assistant | Integrations, local control, permissions |
| Wake word | openWakeWord or equivalent | False activations, per-room behavior, CPU use |
| Speech-to-text (STT) | faster-whisper / Whisper-family local STT | Accuracy in room noise, latency, language support |
| Dialogue / tool calling | Small project-owned service | Explicit state machine, tool allowlist, cancellation |
| Fast LLM | Quantized 3B–8B-class model | Low latency, reliable tool selection |
| Strong LLM | Quantized 14B–32B-class model where hardware allows | Quality, VRAM fit, response latency |
| LLM runtime | Ollama, llama.cpp, or another benchmarked local runtime | Streaming, model concurrency, VRAM management |
| Text-to-speech (TTS) | Piper and/or streaming-capable alternative such as Kokoro | Naturalness, chunk latency, voice licensing |
| Music management | Music Assistant | Providers, zones, queue semantics, player compatibility |
| Synchronized multiroom output | Snapcast or another verified transport | Sync accuracy, buffering, player integration |
| Endpoint compute | Raspberry Pi-class board or equivalent | Audio I/O, thermals, update support |
| Mic array | ReSpeaker XVF3800-class or equivalent | Echo cancellation, beamforming, driver support |

Do not commit to every item simply because it appears in this table. Verify current compatibility and licensing before procurement. Keep interfaces replaceable so the project is not tightly coupled to one model or TTS engine.

## 5. Voice interaction and dialogue behavior

### Target request pipeline

1. Endpoint detects the wake word and captures speech.
2. Endpoint or central service performs voice activity detection and sends audio to STT.
3. Dialogue manager identifies the current turn, conversation context, target room, and whether an interruption is in progress.
4. A routing policy selects a fast model for ordinary requests or a stronger model for complex reasoning.
5. Tool calls pass through a deterministic allowlist and permission checks.
6. Text is emitted incrementally.
7. A sentence/phrase buffer sends safe chunks to streaming TTS.
8. Speech is routed to the requesting room or requested group.
9. The turn manager tracks playback and generation so interruptions cancel stale output cleanly.

### Streaming speech

Do not wait for a complete multi-paragraph answer before starting TTS. Buffer text at sentence or clause boundaries; avoid sending incomplete words or unstable fragments to the synthesizer. Tune buffer size against the trade-off between low first-audio latency and natural prosody.

### “Think harder” behavior

Treat “think harder” as a control command, not ordinary text appended to the previous prompt.

- If Jarvis is speaking, stop playback, clear queued speech chunks, and cancel or supersede active generation.
- Preserve the original user request, relevant conversation context, and completed tool results.
- Re-run the reasoning step with the stronger model.
- Do not repeat external actions that have already completed. Separate reasoning from side-effecting tools and maintain action/turn IDs.
- Reject late audio chunks from a cancelled turn using cancellation tokens or monotonically increasing turn IDs.
- If the answer has already finished, interpret the command as a request to reconsider the last answer.
- Keep “go deeper” distinct if the intended behavior is to elaborate rather than reassess.

### Tool safety

The model must not receive unrestricted shell, filesystem, network, or device access. Implement explicit tools with typed arguments, validation, permission checks, timeouts, and audit logs. Require confirmation for high-impact or destructive actions. Treat speech recognition as fallible; confirm ambiguous requests that affect locks, alarms, access, purchases, or safety-critical equipment.

## 6. Music system and multi-stream behavior

### Separate four concepts

1. **Music identity:** the provider account or household profile that supplies preferences.
2. **Recommendation policy:** how Jarvis decides what to play next.
3. **Playback session:** one independent stream/queue.
4. **Output group:** the room or synchronized group receiving that session.

This separation prevents the recommendation system from being confused with speaker routing.

### Spotify constraints

- Do not assume one Spotify account supports multiple independent simultaneous streams. One stream distributed to several rooms as a synchronized group is different from multiple independent streams.
- Music Assistant's Spotify integration and Spotify Connect behavior have provider-specific limitations. Verify current Music Assistant documentation, Spotify account requirements, and relevant terms before choosing the final route.
- Separate Spotify accounts are the practical route for independent Spotify sessions. Spotify Premium Family has historically offered multiple individual Premium accounts in the US, but verify current plan terms and price before subscribing.
- Do not assume Music Assistant exposes Spotify's full personalized recommendation engine. Confirm which recommendations, radio, autoplay, queue, and account features are available in the chosen integration.

### Shared household taste profile

The desired behavior—one shared household taste profile but independent streams—should be implemented as a Jarvis-owned preference layer, not assumed to emerge automatically from multiple Spotify accounts.

Possible signals:
- Explicit likes/dislikes and “play more/less like this.”
- Skips and early stops, interpreted cautiously.
- Favorite artists, albums, tracks, genres, and eras.
- Context (room, time, activity) only when permitted.
- Household-wide exclusions and per-person or per-room overrides.

A recommendation service can use this shared profile to select candidates, then ask the playback layer to start a stream for a specific room/group. Maintain a transparent preference store with controls to inspect, correct, and delete recorded signals. Do not scrape private provider data or assume the Spotify Web API supplies raw audio.

### Initial music milestone

First prove one source can play reliably to one zone. Then prove one stream can be synchronized across two zones. Only after that test two independent streams on separate zone groups, including queue ownership, source account mapping, failure recovery, and volume isolation.

## 7. Audio hardware and wiring

### Channel topology

- 12 zones × 2 channels per zone = 24 amplifier channels.
- 4 speakers per zone = 48 speakers.
- Each zone has two left speakers and two right speakers.
- For a zone with four nominal 8-ohm speakers, wire the two left speakers in parallel on the left channel and the two right speakers in parallel on the right channel only if the chosen amplifier is stable at 4 ohms and the speakers are appropriate for this arrangement.
- Label every cable at both ends with zone and channel, e.g. Z03-L and Z03-R. Maintain a wiring schedule and as-built diagram.

### Amplifier implementation options

**Preferred first prototype:** use a reputable, protected multichannel amplifier or a small set of known-good amplifier modules in an enclosed, serviceable assembly.

**Custom rack implementation:** discrete Class-D boards can be mounted in a rack shelf, but 1U is likely too tight for convenient cooling, safe power distribution, connectors, and service access. A 2U or larger enclosure is a more realistic starting point. Evaluate either 12 stereo amplifier boards or six four-channel boards, provided the chosen modules meet output-power, load, thermal, protection, and input requirements.

A custom amplifier assembly needs:
- A properly rated, enclosed power supply.
- Correct mains wiring, earthing, branch protection, and strain relief.
- Per-module or per-branch protection as required by the design.
- Thermal design based on measured dissipation, not fan assumptions.
- Input connectors and output terminals that cannot be accidentally shorted.
- Consideration of EMI, grounding, hum, and bridged output topologies.
- A safe power-on sequence and fault isolation.
- Documentation and service access.

Use a qualified electrician for building wiring, circuits, and mains-connected rack equipment. Validate electrical safety and local code before installation.

### Audio interfaces

The system needs 12 stereo program feeds (24 analog channels) if every zone is independently addressable through an analog input path. Confirm the actual output architecture before buying amplifiers: a device with many amplifier channels is not automatically equipped with the required independent inputs or software routing. Digital transport may reduce analog-channel requirements but adds compatibility and clocking constraints.

### Acoustic validation

Before installing all speakers:
- Confirm speaker cutout, depth, back-can/fire-rating requirements, ceiling type, and service access.
- Check whether ceiling speakers need enclosures for predictable bass and sound isolation.
- Test intelligibility and sound coverage at normal listening levels.
- Verify that voice microphones do not hear excessive playback echo.
- Test echo cancellation with music and TTS at representative volume.
- Use consistent gain structure across zones and provide per-zone maximum-volume limits.

## 8. Rack, networking, and power

### Network

- Prefer wired Ethernet for the central server, fixed endpoints where practical, and networked audio transports.
- Put IoT/endpoint devices on a segmented network or VLAN if the router supports it; restrict their access to only the services they need.
- Reserve stable DHCP leases or use a documented address plan.
- Keep the server and endpoints on a local path so core voice functions do not depend on cloud availability.
- Document ports, hostnames, certificates, firewall rules, and discovery requirements.
- Test multicast/discovery behavior before relying on Snapcast or Music Assistant player discovery across VLANs.

### Power planning

Earlier rough planning ranges—not final electrical loads—were:
- GPU server: approximately 700–1,500 W under heavy load depending on hardware and workload.
- Multichannel amplification: highly dependent on output level, efficiency, and duty cycle; use actual equipment specifications and measured use.
- Network, receivers, storage, and miscellaneous rack equipment: budget from selected devices rather than generic estimates.

Two dedicated 20 A, 120 V circuits may be a useful planning concept, but do not treat that as an approved final design. A 20 A / 120 V circuit is 2,400 VA nominal; continuous-load planning often uses 80% (1,920 VA) where applicable. Actual sizing depends on equipment nameplates, power factor, continuous-load rules, local code, and load distribution. Consider 240 V if the server PSU/PDU and local electrical design support it. Have a licensed electrician calculate and install the final circuits.

A UPS is intentionally excluded from the initial build because outages have not been a practical issue. Mitigate ordinary power loss by:
- Enabling BIOS/UEFI restore-on-AC-power-loss.
- Configuring services to start automatically after reboot.
- Using clean shutdown behavior where possible.
- Keeping backups of configuration and critical data.
- Using appropriate surge protection and grounding.

Surge protection is not a UPS and does not replace proper grounding or electrical protection.

## 9. Security, privacy, and reliability

- Prefer local processing for wake-word, STT, LLM, and TTS. Document every cloud dependency and make it opt-in where feasible.
- Keep secrets in environment/configuration secret stores, not in the Git repository.
- Do not commit Wi-Fi passwords, API tokens, provider credentials, private audio, transcripts, or personal listening logs.
- Define audio retention: default to no persistent raw microphone recordings. Store transcripts or preference signals only when needed and under an explicit retention policy.
- Provide a physical or software microphone mute state with a clear indicator.
- Use least-privilege accounts and network rules.
- Separate read-only smart-home queries from state-changing actions.
- Keep a local manual control path for audio and essential smart-home functions.
- Monitor service health, disk space, GPU temperature, model failures, endpoint disconnects, and amplifier faults where hardware permits.
- Pin known-good software versions for the stable system; test updates on a pilot endpoint before deploying to all zones.
- Maintain a rollback procedure and configuration backup.

## 10. Phased implementation plan

Each phase ends with a go/no-go decision. Do not buy all 48 speakers or build all amplifier channels before the critical tests pass.

### Phase 0 — Requirements and design freeze

**Deliverables**
- Confirm zone names, room layout, cable routes, and microphone placement.
- Define target loudness, acceptable latency, privacy rules, budget ceiling, and expected independent streams.
- Draft a full bill of materials with exact SKUs, current prices, dimensions, warranty, and availability.
- Confirm rack location, cooling/noise constraints, electrical capacity, and network topology.
- Select initial software candidates and record known licensing/provider constraints.

**Exit criteria**
- Every zone has a unique ID, speaker pair mapping, endpoint type, and cable plan.
- Amp load and output routing are documented.
- Electrical requirements have been reviewed by a qualified professional before construction.

### Phase 1 — Desktop software proof of concept

**Build**
- Run the voice stack on a development machine/server.
- Connect Home Assistant in a test environment.
- Implement a minimal conversation manager and a small allowlisted tool set.
- Benchmark candidate fast and strong models on real requests.
- Add local STT and TTS; measure complete request-to-audio latency.

**Acceptance tests**
- Wake-word/manual request path can produce a spoken answer repeatedly.
- Basic commands are handled by the fast model.
- A complex request can be routed to the stronger model.
- “Think harder” cancels old speech and regenerates from the original request without duplicating completed actions.
- Restarting services does not corrupt conversation state or leave audio stuck.

### Phase 2 — One-zone audio and voice prototype

**Build**
- One mic-equipped endpoint.
- One stereo zone using four speakers, with two speakers per channel as described above.
- One tested amplifier path and the chosen audio transport.
- Microphone array, endpoint compute, echo cancellation, wake word, STT, LLM, TTS, and speaker output.

**Acceptance tests**
- Voice activation works from intended listening positions.
- TTS remains intelligible at normal music volume.
- No persistent feedback loop or runaway self-activation occurs.
- Interrupting speech stops audible output promptly.
- Playback resumes or cancels according to documented rules.
- Temperatures, noise, and electrical behavior are acceptable after a sustained test.

**Gate:** Do not scale to more endpoints until microphone echo, wake-word behavior, and spoken-response latency are acceptable.

### Phase 3 — Fast/strong model routing and streaming TTS

**Build**
- Add explicit model routing policy.
- Implement streaming text generation and chunked/streaming TTS.
- Add turn IDs, cancellation tokens, bounded queues, and error handling.
- Add timeouts and fallback responses when the strong model is unavailable.

**Acceptance tests**
- Speech begins before the full LLM response has been generated.
- A cancelled turn never speaks stale queued text.
- “Think harder” reconsiders the original request with the stronger model.
- Side-effecting tools are not repeated just because the reasoning pass is repeated.
- Measure median and p95 first-audio latency under idle and concurrent load.

### Phase 4 — Multiroom music, synchronization, and independent streams

**Build**
- Integrate Music Assistant and the selected music provider.
- Establish a known-good output path to one zone, then two zones.
- Configure zone groups, independent volume, and source/queue ownership.
- Add separate provider accounts if required for concurrent streams.
- Prototype a Jarvis-owned household taste profile and explicit feedback controls.

**Acceptance tests**
- One stream plays reliably in one zone.
- The same stream plays in sync in two zones.
- Two independent streams play concurrently to disjoint zone groups.
- Each stream's pause/skip/queue actions affect only its intended session.
- Restarting a player or provider has a documented recovery behavior.
- Verify current provider account limits, plan terms, and Music Assistant integration behavior.

**Gate:** Do not purchase the complete amplifier system until independent streams and room assignment work end-to-end on the chosen playback architecture.

### Phase 5 — Central rack and amplifier prototype

**Build**
- Finalize rack dimensions, ventilation, power distribution, patching, and cable labeling.
- Build or acquire the smallest safe multichannel amplifier assembly that validates the intended topology.
- Validate the 24-channel output architecture with real speakers and audio routing.
- Measure idle draw, operating draw, heat, hum, and clipping under representative use.

**Acceptance tests**
- Every output maps to the intended zone and channel.
- The amplifier remains stable at its specified load during sustained tests.
- No overheating, unsafe exposed mains, unexpected shared grounds, or audible hum.
- Loss of a single amp module does not create an unsafe condition for the rest of the system.
- All serviceable parts can be reached without dismantling unrelated equipment.

### Phase 6 — Expand to four zones

**Build**
- Expand to four zones, including at least two voice endpoints and at least one speaker-only endpoint.
- Verify network discovery, audio grouping, voice-room attribution, and simultaneous speech/music behavior.
- Create installation and endpoint provisioning procedures.

**Acceptance tests**
- Zone mapping is repeatable and documented.
- Multiroom synchronization remains acceptable.
- Voice from one room does not accidentally trigger a response in another room.
- All zones can be muted and volume-limited independently.
- Reboot and network-loss recovery are predictable.

### Phase 7 — Full 12-zone installation

**Build**
- Install all 48 speakers, all 24 amplifier channels, six voice endpoints, six speaker-only endpoints, central server, and network/audio infrastructure.
- Label both ends of every cable.
- Record final rack layout, wiring, IPs, device identifiers, software versions, and configuration backup locations.

**Acceptance tests**
- All 12 zones pass left/right channel and polarity tests.
- All six voice endpoints pass wake-word, STT, TTS, and interruption tests.
- All 12 zones can be addressed independently and grouped correctly.
- At least two independent music streams work concurrently in intended room groups.
- The system survives a planned server reboot and an endpoint disconnect without manual database repair.
- A full-house listening test identifies no major coverage, echo, noise, or synchronization defects.

### Phase 8 — Household recommendation and daily-use polish

**Build**
- Refine household taste profile, feedback semantics, and per-room preferences.
- Add natural-language room selection and playback control.
- Add safe defaults for “play something,” “continue,” “more like this,” “less like this,” and “stop everywhere.”
- Add observability dashboards and maintenance documentation.

**Acceptance tests**
- The profile is inspectable and correctable.
- Skips and likes update the profile predictably.
- Recommendations can be overridden and reset.
- Music sessions remain isolated while using shared preferences.
- Common tasks work without cloud services where the design promises local operation.

### Phase 9 — Hardening and maintenance

**Deliverables**
- Configuration backups and restore drill.
- Version pinning and update procedure.
- Security review of exposed services, secrets, and tool permissions.
- Power-loss recovery test.
- Maintenance checklist and spare-parts list.
- As-built documentation and troubleshooting guide.

**Acceptance tests**
- A clean restore recovers the system from documented backups.
- Server restart automatically returns essential services to operation.
- Credentials can be rotated without rebuilding every endpoint.
- A failed service reports a clear status and does not silently execute unsafe actions.

## 11. Budget and procurement strategy

The project budget is **$5,000+**, but the complete installed cost is not yet known. Avoid treating early estimates as a quote. The final budget must include tax, shipping, connectors, speaker wire, rack hardware, cooling, electrical work, spares, and replacement risk.

| Category | Costing method |
|---|---|
| 48 ceiling speakers | Compare identical pairs/units; verify cutout, depth, impedance, and fire-rating requirements |
| 24 amplifier channels | Compare protected multichannel amps against custom Class-D assemblies, including enclosures and power supplies |
| 24-channel audio output path | Confirm independent feeds, driver support, and zone routing before purchase |
| 6 mic arrays and endpoint boards | Include mounting, power, network, cases, and replaceable cabling |
| GPU server | Price the full system, not only the GPU; include PSU, cooling, RAM, storage, and rack fit |
| Rack and network | Include rails, patching, switches, cable management, airflow, and service clearance |
| Wiring and installation | Include speaker cable, termination, labeling, access, and professional electrical work |
| Storage and backup | Include configuration backup and recovery storage |
| Contingency | Reserve approximately 15–20% until design and installation costs are firm |

Procurement rules:
1. Buy prototype quantities first.
2. Prefer components with published specifications, driver support, warranties, and replaceable parts.
3. Compare total installed cost, not board-only price.
4. Confirm return policies and current availability before buying components that depend on one another.
5. Do not purchase all speakers or custom amp boards until the one-zone and independent-stream gates pass.
6. Record actual cost, SKU, purchase date, warranty, and configuration for every component.

## 12. Key risks and mitigations

| Risk | Mitigation |
|---|---|
| Voice latency feels slow | Measure first-audio latency; stream LLM output to chunked TTS; keep a fast model resident |
| Strong model does not fit VRAM | Benchmark quantized models and context sizes; keep the strong model optional/on-demand |
| TTS hears itself and reactivates wake word | Mic array with echo cancellation, acoustic testing, mute/debounce logic, and playback-aware wake-word handling |
| Spotify integration does not support intended behavior | Prove provider/account behavior in a two-stream prototype before committing |
| Shared taste profile is not equivalent to Spotify recommendations | Build a separate Jarvis-owned preference model and make its behavior transparent |
| Amplifier load or topology is wrong | Confirm minimum impedance, output topology, wiring, and thermal limits with actual hardware |
| DIY amplifier has electrical or thermal faults | Use an enclosed, protected design; professional review; no exposed mains or inaccessible ceiling electronics |
| Zone sync conflicts with low-latency speech | Separate or tune speech and music transport paths |
| Network discovery fails across VLANs | Prototype multicast and discovery rules before segmentation is finalized |
| Software update breaks the house | Pin versions, stage updates on pilot equipment, keep backups and rollback steps |
| Power loss leaves services down | BIOS restore-on-AC, automatic service startup, configuration backups, surge protection |
| Scope and cost expand | Use phase gates and do not scale until acceptance tests pass |

## 13. Operational metrics

Record these during every major test:
- Wake-word false-accept and false-reject rate in each intended environment.
- STT accuracy for common commands, names, and music titles.
- Median and p95 time from end of user speech to first audible response.
- Time from “stop” or “think harder” to silence.
- Model time-to-first-token and tokens/second under realistic concurrency.
- TTS chunk underruns, playback glitches, and stale-audio events.
- Inter-zone synchronization error.
- Independent-stream success rate and recovery time after a provider/player failure.
- CPU/GPU/RAM utilization, GPU temperature, server power draw, and amplifier temperatures.
- Service uptime, endpoint disconnects, and backup restore success.

Do not optimize a metric in isolation. A slightly slower but reliable, interruptible voice pipeline is preferable to a fast pipeline that repeats actions or speaks stale output.

## 14. Initial task backlog

Start with these tasks in order:

- [ ] Draw the 12-zone floor plan and assign permanent zone IDs.
- [ ] Define room-level volume limits, mic mute behavior, and voice response routing.
- [ ] Confirm the intended maximum number of independent music streams.
- [ ] Build the complete BOM spreadsheet and rack dimension drawing.
- [ ] Benchmark at least two fast models and one stronger model on the target GPU class.
- [ ] Test local STT and streaming TTS using representative room noise.
- [ ] Implement a minimal turn manager with cancellation and “think harder.”
- [ ] Prove one zone with four speakers and the intended 4-ohm-per-channel topology.
- [ ] Prove synchronized two-zone playback.
- [ ] Prove two independent streams with separate queues.
- [ ] Decide whether the final amplifier path is commercial multichannel gear or a custom enclosed assembly.
- [ ] Review power, wiring, and installation with qualified professionals.
- [ ] Freeze the architecture only after the prototype acceptance gates pass.

## 15. Reference documentation

Use these as starting points and re-check current documentation during implementation:
- [Home Assistant](https://www.home-assistant.io/)
- [Music Assistant](https://www.music-assistant.io/)
- [Music Assistant — Spotify provider](https://www.music-assistant.io/music-providers/spotify/)
- [Music Assistant — Spotify Connect plugin](https://www.music-assistant.io/plugins/spotify-connect/)
- [Home Assistant — Spotify integration](https://www.home-assistant.io/integrations/spotify/)
- [openWakeWord](https://github.com/dscripka/openWakeWord)
- [faster-whisper](https://github.com/SYSTRAN/faster-whisper)
- [Piper](https://github.com/rhasspy/piper)
- [Snapcast](https://github.com/badaix/snapcast)
- [Ollama](https://ollama.com/)
- [llama.cpp](https://github.com/ggml-org/llama.cpp)

Provider capabilities, pricing, account rules, software compatibility, and licensing can change. Verify them before purchase or deployment.

## Definition of done

The project is ready for normal household use when all 12 zones are documented and tested; all 48 speakers have correct stereo mapping; six voice endpoints support reliable, interruptible local voice interaction; music can be synchronized or played independently as required; “think harder” safely escalates reasoning; the household preference profile is controllable; recovery and backup procedures have been tested; and the as-built wiring, software, and maintenance documentation is complete.
