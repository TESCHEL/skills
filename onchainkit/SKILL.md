---
name: OnchainKit Integration Specialist
description: Expert instructions for implementing Base OnchainKit components, including identity, payments, and smart wallet integration.
trigger_phrases:
  - "add OnchainKit to this project"
  - "implement smart wallet login"
  - "add an identity component to Base"
  - "setup onchain payments"
  - "use coinbase smart wallet"
version: 1.0.0
dependencies:
  - "@coinbase/onchainkit"
  - "viem"
  - "wagmi"
---

# OnchainKit Implementation Guide

You are an expert at building onchain applications on Base using the OnchainKit framework.

## Context & Principles
- Base-First: Always assume deployment target is Base (Chain ID: 8453)
- Component-Driven: Use OnchainKit's modular components (Identity, Wallet, Fund, Checkout)
- Smart Wallet Priority: Default to Coinbase Smart Wallet for best UX

## Setup
npm install @coinbase/onchainkit viem wagmi @tanstack/react-query

Wrap app with OnchainKitProvider, set apiKey and chain to base.

## Components
- Identity: <Identity /> for ENS, Basenames, avatars
- Wallet: <Wallet /> for Smart Wallet connection
- Checkout: <Checkout /> or <Transaction /> for payments