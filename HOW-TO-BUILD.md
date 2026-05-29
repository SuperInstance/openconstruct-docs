# HOW-TO-BUILD.md — The Engineer's Guide to Building OpenConstruct Modules

> "If you can compile it, it's a module. If you can test it, it's *good*."

This guide takes you from zero to publishing. We'll build real modules, write real tests, and wire into the fleet — step by step, with code that compiles.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Your First Module](#2-your-first-module)
3. [Building a Custom Sense Module](#3-building-a-custom-sense-module)
4. [Building a Fleet Node](#4-building-a-fleet-node)
5. [Building a Binding](#5-building-a-binding)
6. [Building a Plato Room](#6-building-a-plato-room)
7. [Testing Philosophy](#7-testing-philosophy)
8. [Publishing](#8-publishing)

---

## 1. Prerequisites

Before you write a single line, make sure your environment is ready.

### Rust Toolchain

OpenConstruct is Rust-first. You need a recent stable toolchain plus some extras:

```bash
# Install rustup if you don't have it
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Ensure stable is up to date
rustup update stable

# Add wasm32 target (needed for some bindings)
rustup target add wasm32-unknown-unknown

# Install cargo tools you'll use constantly
cargo install cargo-edit cargo-nextest cargo-audit
```

Minimum Rust version: **1.75.0** (we use `impl Trait` in `trait` bounds and async closures in traits).

### Python (for bindings and tooling)

```bash
# Python 3.11+ recommended
python3 --version

# Install maturin (for building Python wheels from Rust)
pip install maturin pytest
```

### Node.js (for JS/TS bindings)

```bash
# Node 20+ recommended
node --version

# Install napi-cli for building native Node addons from Rust
npm install -g @napi-rs/cli
```

### Other Tools

| Tool | Why | Install |
|------|-----|---------|
| `git` | Version control, duh | System package manager |
| `pkg-config` | Build scripts for C ABI | `apt install pkg-config` / `brew install pkg-config` |
| `protobuf-compiler` | If your module uses proto definitions | `apt install protobuf-compiler` |
| `just` | Task runner we use for development | `cargo install just` |

### Verify Everything Works

```bash
# Clone the monorepo
git clone https://github.com/openconstruct/openconstruct.git
cd openconstruct

# Run the full test suite — this should pass on your machine
just test

# If it doesn't pass, open an issue. Seriously. We care about clean main.
```

---

## 2. Your First Module

Let's build a hello-world sense module from scratch. This module will observe the system clock and emit text shadows describing the current time.

### Create the Rust Crate

We use a cargo workspace. Modules live in `crates/`:

```bash
# From the repo root
cargo init --lib crates/sense-hello
cd crates/sense-hello
```

Edit `Cargo.toml`:

```toml
[package]
name = "openconstruct-sense-hello"
version = "0.1.0"
edition = "2021"
description = "Hello world sense module — emits time as text shadows"
license = "MIT OR Apache-2.0"

[dependencies]
openconstruct-core = { path = "../openconstruct-core", version = "0.1.0" }
openconstruct-sense = { path = "../openconstruct-sense", version = "0.1.0" }
tokio = { version = "1", features = ["time", "macros", "rt"] }
chrono = "0.4"
tracing = "0.1"

[dev-dependencies]
openconstruct-test = { path = "../openconstruct-test", version = "0.1.0" }
```

### Implement the SenseModule Trait

The `SenseModule` trait is the heart of every sense module:

```rust
// crates/sense-hello/src/lib.rs

use openconstruct_core::shadow::{ShadowRef, TextShadow};
use openconstruct_sense::{SenseModule, SenseContext, SenseResult};
use chrono::Local;
use std::time::Duration;

/// A sense module that observes the system clock and emits text shadows.
pub struct HelloSense {
    /// How often to poll (defaults to 60 seconds).
    interval: Duration,
    /// The last time we emitted a shadow.
    last_emission: Option<String>,
}

impl HelloSense {
    pub fn new() -> Self {
        Self {
            interval: Duration::from_secs(60),
            last_emission: None,
        }
    }

    /// Configure a custom poll interval.
    pub fn with_interval(mut self, interval: Duration) -> Self {
        self.interval = interval;
        self
    }

    /// Generate a human-readable time string.
    fn current_time_description(&self) -> String {
        let now = Local::now();
        format!(
            "The time is {} on {}. It's a {} day.",
            now.format("%I:%M %p"),
            now.format("%B %d, %Y"),
            now.weekday().to_string().to_lowercase()
        )
    }
}

impl Default for HelloSense {
    fn default() -> Self {
        Self::new()
    }
}

#[async_trait::async_trait]
impl SenseModule for HelloSense {
    /// Unique identifier for this module in the registry.
    fn module_id(&self) -> &str {
        "sense.hello.time"
    }

    /// Human-readable description.
    fn description(&self) -> &str {
        "Emits the current time as a text shadow every 60 seconds"
    }

    /// How often the sense loop should invoke `observe()`.
    fn poll_interval(&self) -> Duration {
        self.interval
    }

    /// The main sense observation. Returns zero or more shadows.
    async fn observe(&mut self, _ctx: &SenseContext) -> SenseResult<Vec<ShadowRef>> {
        let description = self.current_time_description();

        // Only emit if the description has changed
        if self.last_emission.as_ref() == Some(&description) {
            return Ok(vec![]);
        }

        self.last_emission = Some(description.clone());

        let shadow = ShadowRef::text(
            "sense.hello.time",
            TextShadow::new(description)
                .with_confidence(1.0)  // We're pretty sure about the clock
                .with_source("system_clock"),
        );

        Ok(vec![shadow])
    }
}
```

### Add Text Shadow Output

You already saw `TextShadow` above. Let's look at what's happening under the hood. A `ShadowRef` is a lightweight reference to a shadow — the actual data lives in the shadow store. `TextShadow::new()` creates a simple text payload:

```rust
// This is what TextShadow looks like internally (simplified)
pub struct TextShadow {
    pub text: String,
    pub confidence: f64,      // 0.0 to 1.0 — how certain is this observation
    pub source: Option<String>, // Where the data came from
    pub timestamp: i64,        // Unix epoch milliseconds
}
```

The `ShadowRef::text()` constructor handles the plumbing — it assigns a unique ID, sets the timestamp, and registers the shadow with the store.

### Write Tests

Every module needs tests. We're not kidding — see [Section 7](#7-testing-philosophy). Here are the minimum tests for `HelloSense`:

```rust
// crates/sense-hello/src/lib.rs (continued, at the bottom)

#[cfg(test)]
mod tests {
    use super::*;
    use openconstruct_test::mock_context;
    use std::time::Duration;

    fn new_module() -> HelloSense {
        HelloSense::new().with_interval(Duration::from_millis(100))
    }

    #[tokio::test]
    async fn test_module_id() {
        let m = new_module();
        assert_eq!(m.module_id(), "sense.hello.time");
    }

    #[tokio::test]
    async fn test_description_exists() {
        let m = new_module();
        assert!(!m.description().is_empty());
    }

    #[tokio::test]
    async fn test_observe_emits_shadow_on_first_call() {
        let mut m = new_module();
        let ctx = mock_context();
        let shadows = m.observe(&ctx).await.unwrap();
        assert_eq!(shadows.len(), 1);
        assert!(shadows[0].is_text());
    }

    #[tokio::test]
    async fn test_observe_no_duplicate_within_same_minute() {
        let mut m = new_module();
        let ctx = mock_context();

        // First call emits
        let first = m.observe(&ctx).await.unwrap();
        assert_eq!(first.len(), 1);

        // Second call within the same time string — no duplicate
        let second = m.observe(&ctx).await.unwrap();
        assert_eq!(second.len(), 0);
    }

    #[tokio::test]
    async fn test_poll_interval_default() {
        let m = HelloSense::new();
        assert_eq!(m.poll_interval(), Duration::from_secs(60));
    }

    #[tokio::test]
    async fn test_poll_interval_custom() {
        let m = HelloSense::new().with_interval(Duration::from_secs(10));
        assert_eq!(m.poll_interval(), Duration::from_secs(10));
    }

    #[tokio::test]
    async fn test_shadow_has_confidence() {
        let mut m = new_module();
        let ctx = mock_context();
        let shadows = m.observe(&ctx).await.unwrap();
        let text_shadow = shadows[0].as_text().unwrap();
        assert_eq!(text_shadow.confidence, 1.0);
    }

    #[tokio::test]
    async fn test_shadow_source_is_set() {
        let mut m = new_module();
        let ctx = mock_context();
        let shadows = m.observe(&ctx).await.unwrap();
        let text_shadow = shadows[0].as_text().unwrap();
        assert_eq!(text_shadow.source.as_deref(), Some("system_clock"));
    }

    #[tokio::test]
    async fn test_default_impl() {
        let m = HelloSense::default();
        assert_eq!(m.poll_interval(), Duration::from_secs(60));
    }

    #[tokio::test]
    async fn test_time_description_format() {
        let m = new_module();
        let desc = m.current_time_description();
        // Should contain "The time is"
        assert!(desc.starts_with("The time is"));
        // Should mention a day of the week
        assert!(["monday", "tuesday", "wednesday", "thursday",
                 "friday", "saturday", "sunday"]
            .iter().any(|d| desc.to_lowercase().contains(d)));
    }
}
```

That's 10 tests. Yes, we're counting. Yes, we care.

### Register with the Module Registry

Modules self-register using the inventory pattern (zero-cost, compile-time registration):

```rust
// crates/sense-hello/src/lib.rs — add at module root level

use openconstruct_sense::register_sense_module;

register_sense_module!(HelloSense);
```

Then add your crate to the workspace `Cargo.toml`:

```toml
# /Cargo.toml (workspace root)
[workspace]
members = [
    # ... existing members ...
    "crates/sense-hello",
]
```

And add it to the module aggregator in `crates/openconstruct-modules/src/lib.rs`:

```rust
pub use openconstruct_sense_hello;
```

That's it. At compile time, the inventory collector picks up your module and makes it available to the runtime.

### Push to GitHub

```bash
# From repo root
git add crates/sense-hello/
git commit -m "feat(sense): add hello-world time sense module"
git push origin your-branch
```

Open a PR. CI will run your tests. If they pass, you're golden.

---

## 3. Building a Custom Sense Module

Now let's build something real: a temperature sensor module that reads from a hardware sensor and emits shadows with temporal tracking.

### Define the ShadowRef Format for Temperature Readings

Temperature is numeric data, so we use `NumericShadow` instead of `TextShadow`:

```rust
// crates/sense-temperature/src/lib.rs

use openconstruct_core::shadow::{ShadowRef, NumericShadow, ShadowMetadata};
use openconstruct_sense::{SenseModule, SenseContext, SenseResult};
use std::collections::VecDeque;
use std::time::Duration;

/// Configuration for a temperature sensor.
pub struct TemperatureConfig {
    /// Poll interval in seconds (default: 30).
    pub poll_interval_secs: u64,
    /// Number of historical readings to retain (default: 10).
    pub history_size: usize,
    /// Sensor device path (e.g., "/sys/class/thermal/thermal_zone0/temp").
    pub device_path: String,
}

impl Default for TemperatureConfig {
    fn default() -> Self {
        Self {
            poll_interval_secs: 30,
            history_size: 10,
            device_path: "/sys/class/thermal/thermal_zone0/temp".into(),
        }
    }
}

/// Trend detected from temporal analysis of readings.
#[derive(Debug, Clone, PartialEq)]
pub enum TemperatureTrend {
    Rising,
    Falling,
    Stable,
    InsufficientData,
}

/// A sense module that reads temperature from a hardware sensor.
pub struct TemperatureSense {
    config: TemperatureConfig,
    /// Circular buffer of recent readings: (timestamp_ms, temp_celsius).
    history: VecDeque<(i64, f64)>,
    /// The last reading we emitted.
    last_reading: Option<f64>,
}

impl TemperatureSense {
    pub fn new(config: TemperatureConfig) -> Self {
        let history = VecDeque::with_capacity(config.history_size);
        Self {
            config,
            history,
            last_reading: None,
        }
    }

    /// Read temperature from the sensor device.
    /// Returns temperature in Celsius.
    pub async fn read_sensor(&self) -> Result<f64, std::io::Error> {
        let content = tokio::fs::read_to_string(&self.config.device_path).await?;
        // Linux thermal zone format: integer millidegrees
        let millidegrees: f64 = content.trim().parse()
            .map_err(|_| std::io::Error::new(
                std::io::ErrorKind::InvalidData,
                "Failed to parse temperature value",
            ))?;
        Ok(millidegrees / 1000.0)
    }

    /// Record a reading in the history buffer.
    fn record_reading(&mut self, timestamp_ms: i64, temp_celsius: f64) {
        if self.history.len() == self.config.history_size {
            self.history.pop_front();
        }
        self.history.push_back((timestamp_ms, temp_celsius));
    }

    /// Analyze the trend over recent readings.
    pub fn detect_trend(&self) -> TemperatureTrend {
        if self.history.len() < 3 {
            return TemperatureTrend::InsufficientData;
        }

        let readings: Vec<f64> = self.history.iter().map(|(_, t)| *t).collect();
        let len = readings.len();

        // Simple linear regression slope
        let x_mean = (len - 1) as f64 / 2.0;
        let y_mean = readings.iter().sum::<f64>() / len as f64;

        let mut numerator = 0.0;
        let mut denominator = 0.0;
        for (i, &y) in readings.iter().enumerate() {
            let x = i as f64;
            numerator += (x - x_mean) * (y - y_mean);
            denominator += (x - x_mean).powi(2);
        }

        if denominator == 0.0 {
            return TemperatureTrend::Stable;
        }

        let slope = numerator / denominator;

        if slope > 0.05 {
            TemperatureTrend::Rising
        } else if slope < -0.05 {
            TemperatureTrend::Falling
        } else {
            TemperatureTrend::Stable
        }
    }

    /// Compute the average of recent readings.
    pub fn average(&self) -> Option<f64> {
        if self.history.is_empty() {
            return None;
        }
        let sum: f64 = self.history.iter().map(|(_, t)| t).sum();
        Some(sum / self.history.len() as f64)
    }
}

#[async_trait::async_trait]
impl SenseModule for TemperatureSense {
    fn module_id(&self) -> &str {
        "sense.temperature.thermal_zone"
    }

    fn description(&self) -> &str {
        "Reads CPU/zone temperature and emits numeric shadows with trend analysis"
    }

    fn poll_interval(&self) -> Duration {
        Duration::from_secs(self.config.poll_interval_secs)
    }

    async fn observe(&mut self, _ctx: &SenseContext) -> SenseResult<Vec<ShadowRef>> {
        let temp = match self.read_sensor().await {
            Ok(t) => t,
            Err(e) => {
                tracing::warn!("Failed to read temperature sensor: {}", e);
                return Ok(vec![]);
            }
        };

        let timestamp = chrono::Utc::now().timestamp_millis();
        self.record_reading(timestamp, temp);

        // Skip emission if temperature hasn't changed meaningfully
        if let Some(last) = self.last_reading {
            if (temp - last).abs() < 0.1 {
                return Ok(vec![]);
            }
        }
        self.last_reading = Some(temp);

        let trend = self.detect_trend();
        let avg = self.average();

        let shadow = ShadowRef::numeric(
            "sense.temperature.thermal_zone",
            NumericShadow::new(temp, "°C")
                .with_confidence(0.95)
                .with_source(&self.config.device_path)
                .with_metadata(ShadowMetadata::new()
                    .set("trend", format!("{:?}", trend))
                    .set("average", format!("{:.2}", avg.unwrap_or(0.0)))
                    .set("history_count", self.history.len().to_string())
                ),
        );

        Ok(vec![shadow])
    }
}
```

### Tests for the Temperature Module

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn test_config() -> TemperatureConfig {
        TemperatureConfig {
            poll_interval_secs: 1,
            history_size: 10,
            device_path: "/tmp/test_temp_sensor".into(),
        }
    }

    #[test]
    fn test_config_default() {
        let c = TemperatureConfig::default();
        assert_eq!(c.poll_interval_secs, 30);
        assert_eq!(c.history_size, 10);
    }

    #[test]
    fn test_record_reading_fills_history() {
        let mut s = TemperatureSense::new(test_config());
        for i in 0..5 {
            s.record_reading(i * 1000, 20.0 + i as f64);
        }
        assert_eq!(s.history.len(), 5);
    }

    #[test]
    fn test_history_is_circular() {
        let mut s = TemperatureSense::new(TemperatureConfig {
            history_size: 3,
            ..test_config()
        });
        for i in 0..5 {
            s.record_reading(i * 1000, 20.0 + i as f64);
        }
        assert_eq!(s.history.len(), 3);
        // Should have kept the last 3
        assert_eq!(s.history[0].1, 22.0);
        assert_eq!(s.history[2].1, 24.0);
    }

    #[test]
    fn test_trend_rising() {
        let mut s = TemperatureSense::new(test_config());
        for i in 0..5 {
            s.record_reading(i * 1000, 20.0 + (i as f64) * 0.5);
        }
        assert_eq!(s.detect_trend(), TemperatureTrend::Rising);
    }

    #[test]
    fn test_trend_falling() {
        let mut s = TemperatureSense::new(test_config());
        for i in 0..5 {
            s.record_reading(i * 1000, 25.0 - (i as f64) * 0.5);
        }
        assert_eq!(s.detect_trend(), TemperatureTrend::Falling);
    }

    #[test]
    fn test_trend_stable() {
        let mut s = TemperatureSense::new(test_config());
        for i in 0..5 {
            s.record_reading(i * 1000, 22.0);
        }
        assert_eq!(s.detect_trend(), TemperatureTrend::Stable);
    }

    #[test]
    fn test_trend_insufficient_data() {
        let mut s = TemperatureSense::new(test_config());
        s.record_reading(0, 22.0);
        assert_eq!(s.detect_trend(), TemperatureTrend::InsufficientData);
    }

    #[test]
    fn test_average_computation() {
        let mut s = TemperatureSense::new(test_config());
        for &t in &[20.0, 22.0, 24.0] {
            s.record_reading(0, t);
        }
        assert_eq!(s.average(), Some(22.0));
    }

    #[test]
    fn test_average_empty() {
        let s = TemperatureSense::new(test_config());
        assert_eq!(s.average(), None);
    }

    #[tokio::test]
    async fn test_module_id() {
        let s = TemperatureSense::new(test_config());
        assert_eq!(s.module_id(), "sense.temperature.thermal_zone");
    }
}
```

### Connect to Plato-Correlator for Fusion

To feed temperature data into the correlation/fusion engine, your module's shadows automatically flow through the shadow bus. If you want to explicitly participate in fusion with other modules (e.g., combining temperature + humidity for heat index), implement the `Correlatable` trait:

```rust
use openconstruct_correlator::Correlatable;

impl Correlatable for TemperatureSense {
    fn correlation_keys(&self) -> Vec<String> {
        vec![
            "temperature".into(),
            "thermal_zone".into(),
            "environment".into(),
        ]
    }

    fn fusion_preferences(&self) -> Vec<(String, f64)> {
        // (module_id, weight) — how much to trust each source
        vec![
            ("sense.temperature.thermal_zone".into(), 0.8),
            ("sense.temperature.dht22".into(), 0.95),
        ]
    }
}
```

The correlator will automatically fuse readings from modules sharing correlation keys, weighted by the preferences you declare.

---

## 4. Building a Fleet Node

A fleet node is hardware running the OpenConstruct runtime. Nodes range from tiny microcontrollers to cloud orchestrators. Here's how to make your hardware join.

### ESP32: Arduino Sketch That Registers as a Room

The ESP32 runs a lightweight sense loop and communicates over MQTT:

```cpp
// firmware/esp32-room/esp32-room.ino

#include <WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>

const char* WIFI_SSID = "your-network";
const char* WIFI_PASS = "your-password";
const char* MQTT_BROKER = "192.168.1.100";
const int MQTT_PORT = 1883;

WiFiClient espClient;
PubSubClient mqtt(espClient);

// Room identity
const char* ROOM_ID = "room.living_room";
const char* NODE_TYPE = "esp32";

void setup() {
    Serial.begin(115200);

    WiFi.begin(WIFI_SSID, WIFI_PASS);
    while (WiFi.status() != WL_CONNECTED) {
        delay(500);
        Serial.print(".");
    }
    Serial.println(" WiFi connected");

    mqtt.setServer(MQTT_BROKER, MQTT_PORT);
    mqtt.setCallback(on_message);

    connect_mqtt();
    register_node();
}

void connect_mqtt() {
    while (!mqtt.connected()) {
        String clientId = String(ROOM_ID) + "-" + String(random(0xffff), HEX);
        if (mqtt.connect(clientId.c_str())) {
            Serial.println("MQTT connected");
            mqtt.subscribe("fleet/commands/#");
            mqtt.subscribe("fleet/rooms/living_room/in");
        } else {
            delay(5000);
        }
    }
}

void register_node() {
    StaticJsonDocument<256> doc;
    doc["node_id"] = ROOM_ID;
    doc["node_type"] = NODE_TYPE;
    doc["capabilities"] = serialized("[\"sense\",\"room\"]");
    doc["firmware"] = "0.1.0";

    char buffer[256];
    serializeJson(doc, buffer);
    mqtt.publish("fleet/register", buffer);
}

void on_message(char* topic, byte* payload, unsigned int length) {
    // Handle fleet commands
    Serial.printf("Message arrived [%s]: ", topic);
    for (int i = 0; i < length; i++) {
        Serial.print((char)payload[i]);
    }
    Serial.println();
}

void publish_shadow(const char* sense_id, float value, const char* unit) {
    StaticJsonDocument<256> doc;
    doc["module_id"] = sense_id;
    doc["type"] = "numeric";
    doc["value"] = value;
    doc["unit"] = unit;
    doc["confidence"] = 0.9;
    doc["timestamp"] = millis();

    char buffer[256];
    serializeJson(doc, buffer);

    String topic = "fleet/shadows/" + String(ROOM_ID);
    mqtt.publish(topic.c_str(), buffer);
}

void loop() {
    if (!mqtt.connected()) {
        connect_mqtt();
    }
    mqtt.loop();

    // Read internal temperature sensor (approximate)
    float temp = temperatureRead();  // ESP32 built-in, in Fahrenheit
    float temp_c = (temp - 32.0) * 5.0 / 9.0;

    publish_shadow("sense.temperature.esp32_internal", temp_c, "°C");

    // Publish heartbeat every 30 seconds
    static unsigned long last_heartbeat = 0;
    if (millis() - last_heartbeat > 30000) {
        StaticJsonDocument<128> doc;
        doc["node_id"] = ROOM_ID;
        doc["uptime_ms"] = millis();
        doc["free_heap"] = ESP.getFreeHeap();

        char buffer[128];
        serializeJson(doc, buffer);
        mqtt.publish("fleet/heartbeat", buffer);

        last_heartbeat = millis();
    }

    delay(5000);  // Sense every 5 seconds
}
```

### Jetson: CUDA Sense Processing + Local Inference

For NVIDIA Jetson devices, you get CUDA acceleration for sense processing:

```rust
// crates/fleet-jetson/src/lib.rs

use openconstruct_core::shadow::{ShadowRef, ImageShadow};
use openconstruct_sense::{SenseModule, SenseContext, SenseResult};
use std::time::Duration;

/// A Jetson-accelerated vision sense module using CUDA.
pub struct JetsonVisionSense {
    /// Path to the CUDA inference engine.
    engine_path: String,
    /// Confidence threshold for detections.
    confidence_threshold: f32,
}

impl JetsonVisionSense {
    pub fn new(engine_path: impl Into<String>) -> Self {
        Self {
            engine_path: engine_path.into(),
            confidence_threshold: 0.7,
        }
    }
}

#[async_trait::async_trait]
impl SenseModule for JetsonVisionSense {
    fn module_id(&self) -> &str {
        "sense.vision.jetson"
    }

    fn description(&self) -> &str {
        "Jetson CUDA-accelerated object detection sense module"
    }

    fn poll_interval(&self) -> Duration {
        Duration::from_millis(200) // ~5 FPS
    }

    async fn observe(&mut self, _ctx: &SenseContext) -> SenseResult<Vec<ShadowRef>> {
        // In production, this calls into TensorRT via FFI.
        // For this guide, we show the interface:
        //
        // let frame = capture_frame().await?;
        // let detections = cuda_infer(&self.engine_path, &frame)?;
        // let shadows = detections.into_iter()
        //     .filter(|d| d.confidence >= self.confidence_threshold)
        //     .map(|d| ShadowRef::text("sense.vision.jetson", /* ... */))
        //     .collect();

        Ok(vec![])
    }
}
```

### Desktop: Full Agent Shell

A desktop node runs the complete OpenConstruct agent shell — all modules, the plato-correlator, and the full fleet protocol:

```bash
# Build the agent shell
cargo build --release -p openconstruct-agent

# Run with default config
./target/release/openconstruct-agent

# Or with a specific config
./target/release/openconstruct-agent --config /path/to/config.toml
```

The agent shell automatically discovers registered modules, starts the sense loop, connects to the fleet MQTT broker, and begins processing.

### Cloud: Fleet Orchestration

The cloud node orchestrates the fleet — it aggregates shadows from all nodes, runs cross-node correlation, and provides the fleet API:

```bash
# Deploy the fleet orchestrator
cargo build --release -p openconstruct-fleet
./target/release/openconstruct-fleet --config fleet-config.toml
```

A minimal fleet config:

```toml
# fleet-config.toml
[fleet]
name = "home-fleet"
bind = "0.0.0.0:8443"

[broker]
# Use embedded MQTT broker or connect to external
type = "embedded"

[storage]
type = "sqlite"
path = "./fleet.db"

[orchestrator]
# How often to run cross-node correlation
correlation_interval_secs = 60
# Maximum nodes in the fleet
max_nodes = 100
```

---

## 5. Building a Binding

Want OpenConstruct in Python, JavaScript, or your language of choice? Build a binding using the C ABI.

### Use the C ABI (`openconstruct-abi`)

The `openconstruct-abi` crate exposes a stable C interface:

```rust
// crates/openconstruct-abi/src/lib.rs (simplified)

/// Opaque handle to a shadow client.
#[no_mangle]
pub extern "C" fn openconstruct_client_new(
    broker_addr: *const c_char,
) -> *mut OcClient {
    let addr = unsafe { CStr::from_ptr(broker_addr) }.to_str().unwrap();
    let client = OcClient::connect(addr);
    Box::into_raw(Box::new(client))
}

/// Free a client handle.
#[no_mangle]
pub extern "C" fn openconstruct_client_free(client: *mut OcClient) {
    if !client.is_null() {
        unsafe { drop(Box::from_raw(client)); }
    }
}

/// Emit a text shadow.
#[no_mangle]
pub extern "C" fn openconstruct_emit_text_shadow(
    client: *mut OcClient,
    module_id: *const c_char,
    text: *const c_char,
    confidence: f64,
) -> i32 {
    // ... implementation
    0  // success
}

/// Query shadows by module ID. Returns JSON string.
#[no_mangle]
pub extern "C" fn openconstruct_query_shadows(
    client: *mut OcClient,
    module_id: *const c_char,
    out_json: *mut *mut c_char,
) -> i32 {
    // ... implementation
    0
}
```

### Python Binding Example (using cffi or pyo3)

Using `pyo3` for a native Python extension:

```rust
// bindings/python/src/lib.rs

use pyo3::prelude::*;
use openconstruct_core::shadow::{ShadowRef, TextShadow};

#[pyclass]
struct ShadowClient {
    inner: openconstruct_client::Client,
}

#[pymethods]
impl ShadowClient {
    #[new]
    fn new(broker_addr: &str) -> PyResult<Self> {
        let inner = openconstruct_client::Client::connect(broker_addr)
            .map_err(|e| pyo3::exceptions::PyConnectionError::new_err(e.to_string()))?;
        Ok(Self { inner })
    }

    fn emit_text(&self, module_id: &str, text: &str, confidence: f64) -> PyResult<()> {
        self.inner.emit(ShadowRef::text(
            module_id,
            TextShadow::new(text.to_string()).with_confidence(confidence),
        )).map_err(|e| pyo3::exceptions::PyRuntimeError::new_err(e.to_string()))
    }

    fn query(&self, module_id: &str) -> PyResult<String> {
        self.inner.query(module_id)
            .map_err(|e| pyo3::exceptions::PyRuntimeError::new_err(e.to_string()))
    }
}

#[pymodule]
fn openconstruct(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_class::<ShadowClient>()?;
    Ok(())
}
```

### Tests

```python
# bindings/python/tests/test_client.py

import pytest
from openconstruct import ShadowClient

def test_client_creation():
    """Client can be created with a broker address."""
    client = ShadowClient("tcp://localhost:1883")
    assert client is not None

def test_emit_text_shadow():
    """Can emit a text shadow."""
    client = ShadowClient("tcp://localhost:1883")
    client.emit_text("test.module", "hello world", 0.95)

def test_query_returns_json():
    """Query returns valid JSON."""
    client = ShadowClient("tcp://localhost:1883")
    result = client.query("test.module")
    import json
    data = json.loads(result)
    assert isinstance(data, list)
```

### Publish

```bash
# Build the wheel
maturin build --release

# Publish to PyPI
maturin publish
```

---

## 6. Building a Plato Room

Plato rooms are knowledge graph nodes that compute consistency ratios (CR) over sets of tiles. Think of them as verified knowledge compartments.

### Define Tiles with Dependencies

```rust
// crates/plato-room-environment/src/lib.rs

use openconstruct_plato::{Tile, Room, RoomGraph, TileValue, Dependency};

/// A tile representing the current temperature.
pub struct TemperatureTile;

impl Tile for TemperatureTile {
    fn tile_id(&self) -> &str {
        "environment.temperature"
    }

    fn dependencies(&self) -> Vec<Dependency> {
        vec![
            Dependency::Shadow("sense.temperature.thermal_zone"),
            Dependency::Shadow("sense.temperature.dht22"),
            Dependency::Tile("environment.humidity"),  // For heat index
        ]
    }

    fn compute(&self, deps: &TileInputs) -> TileValue {
        // Get the best temperature reading
        let temp = deps.best_numeric("sense.temperature")
            .unwrap_or_default();

        let humidity = deps.get_tile_value("environment.humidity")
            .and_then(|v| v.as_float())
            .unwrap_or(50.0);

        // Compute heat index (simplified Steadman formula)
        let hi = if temp >= 27.0 {
            -8.784 + 1.611 * temp + 2.338 * humidity
                - 0.1461 * temp * humidity
                - 0.01230 * temp * temp
                - 0.01642 * humidity * humidity
                + 0.002211 * temp * temp * humidity
                + 0.000725 * temp * humidity * humidity
                - 0.000003582 * temp * temp * humidity * humidity
        } else {
            temp  // No meaningful heat index below 27°C
        };

        TileValue::new(hi)
            .with_unit("°C")
            .with_metadata("source_temp", temp.to_string())
            .with_metadata("source_humidity", humidity.to_string())
    }
}

/// The environment room — aggregates environmental tiles.
pub struct EnvironmentRoom;

impl Room for EnvironmentRoom {
    fn room_id(&self) -> &str {
        "environment"
    }

    fn tiles(&self) -> Vec<Box<dyn Tile>> {
        vec![
            Box::new(TemperatureTile),
            // Add more tiles: humidity, pressure, air_quality, etc.
        ]
    }
}
```

### Compute CR (Consistency Ratio)

Every tile has a CR — how well its output agrees with its dependencies:

```rust
use openconstruct_plato::{ConsistencyRatio, CRThreshold};

impl Tile for TemperatureTile {
    // ... (as above)

    fn consistency_check(&self, value: &TileValue, deps: &TileInputs) -> ConsistencyRatio {
        let temp_shadow = deps.best_numeric("sense.temperature");
        let tile_temp = value.as_float();

        match (temp_shadow, tile_temp) {
            (Some(shadow_val), Some(tile_val)) => {
                let delta = (shadow_val - tile_val).abs();
                let cr = if delta < 0.5 {
                    1.0  // Within 0.5°C — excellent agreement
                } else if delta < 2.0 {
                    0.8  // Within 2°C — acceptable
                } else if delta < 5.0 {
                    0.5  // Diverging — investigate
                } else {
                    0.1  // Major disagreement
                };
                ConsistencyRatio::new(cr, CRThreshold::default())
            }
            _ => ConsistencyRatio::insufficient_data(),
        }
    }
}
```

### Add to the Room Graph and Connect Rooms

```rust
// In your module registration

use openconstruct_plato::register_room;

register_room!(EnvironmentRoom);

// Connect rooms in the graph
// This is done in the fleet config or programmatically:

fn build_room_graph() -> RoomGraph {
    let mut graph = RoomGraph::new();

    // Add rooms
    graph.add_room(EnvironmentRoom);
    graph.add_room(WeatherRoom);
    graph.add_room(OccupancyRoom);

    // Connect: weather → environment (external conditions inform indoor)
    graph.add_edge("weather", "environment", EdgeWeight::Influence(0.6));

    // Connect: occupancy → environment (people affect temperature)
    graph.add_edge("occupancy", "environment", EdgeWeight::Influence(0.3));

    graph
}
```

---

## 7. Testing Philosophy

Testing isn't optional. It's how we communicate what the code *means*.

### Every Module Has 10+ Tests

This is a guideline, not a hard rule. But aim for it. Test:

- **Construction** — does it build with defaults? With custom config?
- **Core behavior** — does the happy path work?
- **Edge cases** — empty inputs, max values, invalid data
- **Error handling** — does it fail gracefully?
- **Temporal behavior** — if your module tracks time or history
- **Traits** — does it satisfy the trait contract?

### Zero-Dependency Pure Rust Preferred

Tests should run without hardware, without network, without external services. Use the `openconstruct-test` crate for mocks:

```rust
use openconstruct_test::mock_context;

#[tokio::test]
async fn test_something_without_hardware() {
    let ctx = mock_context();  // No real hardware needed
    let mut module = MySense::new();
    let result = module.observe(&ctx).await.unwrap();
    assert!(!result.is_empty());
}
```

If you *must* talk to external services in tests, gate them behind feature flags:

```toml
[features]
integration = []

[dev-dependencies]
# only needed for integration tests
```

```rust
#[cfg(feature = "integration")]
#[tokio::test]
async fn test_real_sensor() {
    // This talks to actual hardware
}
```

### Tests Are Documentation

A good test suite is worth more than API docs. Write tests that *explain* the behavior:

```rust
#[test]
fn test_trend_detection_ignores_noise() {
    // Temperature readings with minor random noise should still be "stable"
    let mut s = TemperatureSense::new(test_config());
    let readings = [22.0, 22.01, 21.99, 22.02, 21.98];
    for (i, &t) in readings.iter().enumerate() {
        s.record_reading(i as i64 * 1000, t);
    }
    assert_eq!(s.detect_trend(), TemperatureTrend::Stable);
}
```

That test *tells* you: "noise within ±0.02°C is considered stable." It's a spec.

### Continuous Integration

Every PR runs:

```yaml
# .github/workflows/test.yml (simplified)
- name: Run tests
  run: cargo nextest run --workspace

- name: Check formatting
  run: cargo fmt --all -- --check

- name: Run clippy
  run: cargo clippy --workspace -- -D warnings
```

No warnings, no formatting issues, all tests green. That's the bar.

---

## 8. Publishing

Once your module is tested and reviewed, it's time to share it with the world.

### Crates.io (Rust)

```bash
# Make sure Cargo.toml has all required metadata
# - name, version, description, license, repository, keywords, categories

# Dry run first
cargo publish --dry-run

# For real
cargo publish
```

Required `Cargo.toml` fields for publishing:

```toml
[package]
name = "openconstruct-sense-temperature"
version = "0.1.0"
edition = "2021"
description = "Temperature sensor sense module for OpenConstruct"
license = "MIT OR Apache-2.0"
repository = "https://github.com/yourname/openconstruct-sense-temperature"
keywords = ["openconstruct", "sensor", "temperature", "iot"]
categories = ["hardware-support", "embedded"]
```

### PyPI (Python Bindings)

```bash
# Using maturin (for Rust-based Python packages)
maturin build --release
maturin publish

# Or, if pure Python:
python -m build
twine upload dist/*
```

### npm (JavaScript/TypeScript Bindings)

```bash
# Build the native addon
napi build --platform --release

# Package
npm pack

# Publish
npm publish
```

### Version Strategy

We follow [Semantic Versioning](https://semver.org/):

- **0.x.x** — Breaking changes are expected. API is not stable.
- **1.x.x** — Public API is stable. Breaking changes require a major version bump.

Modules within the OpenConstruct monorepo are versioned together. External modules (your own crates) version independently but should depend on `openconstruct-core` with a caret requirement:

```toml
[dependencies]
openconstruct-core = "0.1"  # Compatible with any 0.1.x
```

### Changelog

Every module should maintain a `CHANGELOG.md`:

```markdown
# Changelog

## [0.1.0] - 2026-05-29

### Added
- Initial release
- Temperature reading from thermal zones
- Trend detection (rising/falling/stable)
- Circular history buffer (configurable size)
- Plato correlator integration
```

---

## That's It. Go Build Something.

You now have everything you need to:

- ✅ Create a sense module from scratch
- ✅ Add temporal tracking and trend detection
- ✅ Wire it into the fleet (ESP32, Jetson, desktop, or cloud)
- ✅ Build bindings for any language
- ✅ Create Plato rooms with CR computation
- ✅ Write tests that matter
- ✅ Publish to the world

The monorepo is at [github.com/openconstruct/openconstruct](https://github.com/openconstruct/openconstruct). PRs welcome. Issues welcome. Questions welcome.

Go make the machines see.
