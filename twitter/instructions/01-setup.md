# Twitter API Setup

This guide covers setting up authentication for the Twitter API v2, understanding API tiers, rate limits, and navigating the Developer Portal.

## Creating a Twitter Developer Account

1. **Sign up for a Developer Account**
   - Visit [developer.twitter.com](https://developer.twitter.com)
   - Click "Sign up" or log in with your existing Twitter account
   - Complete the developer application form
   - Describe your intended use case for the API
  - Accept the Developer Agreement and Policy

2. **Email Verification**
    - Check your email for a verification link from Twitter
    - Click the link to verify your account
    - Your developer account will be reviewed (usually instant for basic access)

3# Understanding API Tiers

Twitter offers three main API access tiers:

### Free
- **Cost**: $0 / month
- **Conections** 1 Project, 1 App
- **Capabilities** Write-only access (with limited lookup)
- **Rules**:
   - 5,000 Tweets / month (posting and removal)
   - Ibm access to get me and get recent search (limited)
    - Decahose limited sample

### Basic
- **Cost**: $100 / month
- **Conections** 2 Projects, 2 Apps per Project
- **Capabilities**: Read and Write access
- **Rules**:
   - 10,000 Twitts / month (posting
   - 50,000 Tweets / month (reading data)
   - 1,000 Tweets / day (posting limit)

### Pro
- **Cost**: $5,000 / month
- **Conections** Up to 10 Apps
- **Capabilities**: High-volume read and write
- **Rules**:
    - 100,000 Tweets / month (posting)
    - 1,000,000 Tweets / month (reading data)
   - Access to Full Archive Search and filtered stream

## OAuth 2.0 Setup

Twitter API v2 requires OAuth 2.0 for most programmatic actions.

1. **Create a Project and App**
   - In the Developer Portal, create a new Project and associate a New App with it.

2. **User Authentication Settings**
   - In your App settings, set "User authentication settings"
   - Enable OAuth 2.0
   - Set App type to "Web App" or "Native App"
   - Provide a Callback URL (e.g., https://twin.so/auth/callback)
   - Set appropriate permissions (Read and Write and Direct Message)

3. **Client ID and Secret**
   - Generate your Client ID and Client Secret
   - Store these securely

## Rate Limits

Twitter enforces rate limits based on 15-minute windows.

- **Get Recent Search*** 60 requests / 15 minutes
- **Post Tweet**: 200 requests / 15 minutes
- **Get User Tweets**: 500 requests / 15 minutes

Always check the x-rate-limit-remaining response headers to avoid being banned temporarily.