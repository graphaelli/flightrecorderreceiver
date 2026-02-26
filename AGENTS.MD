# AGENTS.md

This file provides guidance to AI Agents when working with code in this repository.

## Project

An OpenTelemetry Collector receiver that transforms Go Flight Recorder (`trace.FlightRecorder`) data into OTel Profiles and Metrics signals. Requires Go 1.25+.

## Commands

- `make build` — compile
- `make test` — run all tests (`go test ./...`)
- `go test -run TestConvert ./...` — run a single test
- `make lint` — staticcheck with all checks enabled
- `make fmt` — format with gofumpt

Linting and formatting tools are managed via `tools.mod` (not the main `go.mod`).

## Architecture

Single-package receiver (`package flightrecorderreceiver`) with no sub-packages.

**Factory & Config:** `factory.go` registers a receiver factory that creates both a Profiles receiver and a Metrics receiver. Both share the same underlying `flightRecorderReceiver` instance, coordinated via `sync.Once` in the `Config` struct. Config embeds `scraperhelper.ControllerConfig` for scheduling.

**Receiver lifecycle:** `receiver.go` implements Start/Shutdown. The main loop applies an initial delay, performs an immediate scrape, then scrapes periodically at `collection_interval`. Each scrape globs for flight recorder files (using `doublestar`), reads each with `trace.NewReader()`, and fans results to both consumers.

**Conversion pipeline:** `convert.go` processes trace events in a single pass:
- `EventSync` → captures clock snapshots for trace-time-to-wall-clock conversion
- `EventMetric` → emits OTel gauge metrics (preserving `runtime/metrics` naming)
- `EventRangeBegin/End` + `EventStateTransition` → builds OTel profile samples with stack traces

**Deduplication:** `lookup.go` maintains hash-based lookup tables (using xxh3) for locations, functions, strings, attributes, and stacks. These build the profile dictionary to avoid duplicates.

**Data flow:** Flight Recorder file → `trace.Reader` → event loop → lookup tables (dedup) → OTel Profiles + Metrics → consumers

## Testing

Tests in `convert_test.go` generate real flight recorder traces at runtime using `runtime/trace`, then validate conversion output. Helper functions (`primeFactors`, `fibonacci`) create CPU-bound workloads for trace generation.
