# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview
This repository contains the specification and implementation for the **营小助配置中心 (Yingxiaozhu Configuration Center)**, a bank internal operation assistance system.

### Core Capabilities
The system is divided into two main modules:
1. **智能提示 (Sentinel)**: Regulatory requirement communication, in-process risk reminders, and new employee operation guidance
2. **智能作业 (Smart Operation)**: Page recognition, data acquisition, automatic form filling, and execution control

### Architecture
- **Backend Configuration System**: Manages trigger conditions, prompt strategies, operation scenarios, and field configurations
- **Browser Plugin**: Runs on operation pages to monitor events, render prompts, and execute automatic processing
- **Third-party Integration**: Works with external difference highlighting applications for audit scenarios

### Current Status
This repository is currently in the specification phase. Only the requirement specification document exists at `spec/smart-operation/spec.md`. No implementation code has been written yet.

## Key Specifications & Constraints
- First phase priority: Implement **智能提示 (Sentinel)** capability first, smart operation capability will follow in later milestones
- First phase components: Only support录入 (input), 赋值 (assignment), 接口请求 (API request), 获取字段值 (get field value) components
- Scene orchestration: Supports branch logic, does not support loop logic in first phase
- Page recognition: Priority uses page code (obtained via API), URL matching as fallback
- Configuration lifecycle: Draft → To be published → Published → Offline, with version tracking and rollback support
- Security: Custom scripts run in a restricted sandbox, no network access or DOM modification allowed
- Performance: Additional page load time from plugin should not exceed 2000ms, automatic process failure rate ≤10%

## Repository Structure
```
/
├── spec/
│   └── smart-operation/
│       └── spec.md   # Full requirement specification document
└── CLAUDE.md         # This file
```

## Development Commands
Since no implementation code exists yet, no build/lint/test commands are available. This section will be updated as code is added to the repository.