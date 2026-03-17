# Customer Engagement Platform — .NET 8 + Angular 17+ Project Specification

## Context

This is a full project specification for building a **customer engagement / omnichannel support platform** using **.NET 8 (ASP.NET Core)** and **Angular 17+**. The feature set is derived from a comprehensive audit of the Chatwoot open-source platform, scoped down to the features listed below.

The platform enables businesses to manage customer conversations across multiple channels (web chat, email, API), automate workflows, and provide AI-assisted support — all from a unified dashboard.

### Offline / Air-Gapped Deployment Constraint

This platform is designed to run **fully offline on an isolated network** with **no internet connectivity**. All external/cloud service dependencies have been replaced with self-hosted alternatives:

| Cloud Service | Self-Hosted Replacement |
|--------------|------------------------|
| AWS S3 / Azure Blob Storage | **MinIO** (S3-compatible object storage, self-hosted) |
| Google Dialogflow | **Rasa Open Source** (self-hosted NLU/dialogue engine) |
| Firebase Cloud Messaging (FCM) | **Self-hosted Web Push** (VAPID keys + custom push relay) |
| MaxMind GeoIP (online) | **MaxMind GeoLite2 local database** (offline `.mmdb` file) |
| OpenAI / Cloud LLM APIs | **Ollama** or **vLLM** (self-hosted LLM inference) |
| Google/Microsoft OAuth | Removed — use local SMTP credentials or **self-hosted SAML IdP** (e.g., Keycloak) |
| Cloud CI/CD (GitHub Actions) | **Local container registry** + **Gitea/GitLab** + **Drone CI** or manual deploy |
| npm / NuGet registries | **Verdaccio** (npm) + **BaGet** (NuGet) — local package mirrors |
| Seq (cloud logging) | **Seq self-hosted** or **Grafana Loki** |

All container images, npm packages, NuGet packages, and GeoIP databases must be pre-loaded onto the isolated network before deployment.

---

## 1. FEATURE INVENTORY

### 1.1 Communication Channels (3)

| Channel | Description | Key Capabilities |
|---------|-------------|------------------|
| **Web Widget** | Embeddable JavaScript chat widget for websites | Pre-chat forms, HMAC security, domain whitelisting, greeting messages, reply-time indicators, file attachments, emoji picker, continuity via email |
| **Email** | IMAP inbound / SMTP outbound email channel | IMAP polling, SMTP delivery, plain credential auth (no OAuth — offline network), forward-to address, email threading (In-Reply-To / References headers), HTML + plain text |
| **API** | Custom REST channel for third-party systems | HMAC token auth, webhook URL for outbound delivery, custom metadata, message create/update via REST |

### 1.2 Conversation Management

| Feature | Description |
|---------|-------------|
| **Conversations CRUD** | Create, view, update, resolve, reopen, snooze conversations |
| **Message Types** | Incoming, outgoing, activity (system), template messages |
| **Attachments** | File/image/video/audio uploads on messages (MinIO self-hosted object storage) |
| **Assignments** | Manual assignment to agents or teams, auto-assign via round-robin |
| **Labels** | Tag conversations and contacts with colored labels |
| **Participants/Watchers** | Add agents as watchers to follow conversation updates |
| **Mentions** | @mention agents in conversation messages |
| **CSAT Surveys** | Customer satisfaction survey triggered after resolution (1-5 rating + feedback) |
| **Custom Filters/Views** | Save multi-criteria filter sets as named views |
| **Bulk Actions** | Bulk assign, label, resolve, change status across multiple conversations |
| **Conversation Actions** | Mute, snooze (with timer), set priority (low/medium/high/urgent), typing indicators |
| **Draft Messages** | Auto-save message drafts per conversation |
| **Conversation Search** | Full-text search across message content, contacts, and metadata |
| **Conversation Status** | States: open, resolved, pending, snoozed |
| **Conversation Priority** | Levels: none, low, medium, high, urgent |

### 1.3 Contact Management

| Feature | Description |
|---------|-------------|
| **Contacts CRUD** | Create, view, update, delete contacts |
| **Contact Merge** | Merge duplicate contacts (combine conversations, attributes) |
| **Contact Inboxes** | Track which channels a contact communicates through |
| **Contact Notes** | Private agent notes attached to contacts |
| **Contact Labels** | Label/tag contacts for segmentation |
| **Contact Conversations** | View all conversations for a given contact |
| **Custom Attributes** | Define custom fields (text, number, date, list, checkbox, link, currency) for contacts and conversations |
| **Contact Search** | Full-text search by name, email, phone, identifier |
| **Contact Import** | CSV bulk import with field mapping |
| **Contact IP Lookup** | Geo-locate contacts by IP (MaxMind GeoLite2 offline `.mmdb` database) |
| **Contact Types** | visitor, lead, customer |

### 1.4 Inbox Management

| Feature | Description |
|---------|-------------|
| **Inbox CRUD** | Create and configure inboxes per channel type |
| **Inbox Members** | Assign specific agents to each inbox |
| **Working Hours** | Define business hours per inbox (timezone-aware, per-day schedules) |
| **Assignment Policies** | Round-robin, manual, or auto-assignment strategies |
| **Out-of-Office Messages** | Auto-reply when outside business hours |
| **CSAT Templates** | Custom CSAT survey templates per inbox |

### 1.5 Team Management

| Feature | Description |
|---------|-------------|
| **Teams CRUD** | Create and manage agent teams |
| **Team Members** | Add/remove agents from teams |
| **Team Assignment** | Assign conversations to entire teams |
| **Auto-Assign Toggle** | Enable/disable auto-assignment per team |

### 1.6 Agent & User Management

| Feature | Description |
|---------|-------------|
| **User/Agent CRUD** | Create, update, delete agent accounts |
| **Multi-Account Membership** | Users can belong to multiple accounts (tenants) |
| **Roles** | Built-in: Administrator, Agent |
| **Custom Roles (Enterprise)** | Define custom permission sets (conversation_manage, contact_manage, report_manage, knowledge_base_manage) |
| **Agent Bots** | Webhook-based bot agents that auto-respond to messages |
| **Agent Bot Inboxes** | Assign bots to specific inboxes (active/inactive) |
| **Notification Settings** | Per-agent preferences for email and push notifications |
| **Notification Subscriptions** | Web Push subscription management (VAPID-based, self-hosted push relay) |
| **Profile Management** | Avatar, availability (online/offline/busy), UI settings |
| **MFA/2FA** | TOTP-based multi-factor authentication with backup codes |

### 1.7 Automation & Workflows

| Feature | Description |
|---------|-------------|
| **Automation Rules** | Event-driven rules: trigger (e.g. conversation_created) + conditions (e.g. inbox_id = X AND status = open) + actions (e.g. assign_team, add_label, send_message). Supports AND/OR logic |
| **Macros** | Saved sequences of actions agents can execute with one click (personal or global visibility) |
| **Campaigns** | Ongoing (triggered by rules) or one-off (scheduled) proactive messages |
| **Canned Responses** | Pre-written response templates searchable by short code |

**Automation Rule Actions (16):**
send_message, add_label, remove_label, send_email_to_team, assign_team, assign_agent, send_webhook_event, mute_conversation, send_attachment, change_status, resolve_conversation, snooze_conversation, change_priority, send_email_transcript, add_private_note, open_conversation

**Automation Rule Conditions (20+):**
content, email, country_code, status, message_type, browser_language, assignee_id, team_id, referer, city, inbox_id, mail_subject, phone_number, priority, conversation_language, labels, plus custom attributes

### 1.8 AI / Captain (Enterprise)

| Feature | Description |
|---------|-------------|
| **Captain Assistants** | Configurable AI assistants with name, description, temperature, response guidelines, guardrails |
| **Captain Documents** | Upload knowledge documents (PDF, text) for AI to reference |
| **Captain Scenarios** | Define guided conversation flows for the AI |
| **Captain Custom Tools** | Register external tools the AI can invoke (function calling) |
| **Copilot** | Agent-facing AI assistant: suggests replies, rewrites, summarizes, generates labels and follow-ups |
| **Article Embeddings** | Vector embeddings (pgvector) for semantic search across knowledge base |
| **Captain Inbox** | Connect AI assistants to specific inboxes for auto-response |
| **LLM Integration** | Pluggable LLM backend via **Ollama** or **vLLM** (self-hosted, OpenAI-compatible API). No cloud LLM dependency |

**Copilot Tasks:** rewrite, summarize, reply_suggestion, label_suggestion, follow_up

### 1.9 Help Center / Knowledge Base

| Feature | Description |
|---------|-------------|
| **Portals** | Create branded knowledge base portals (name, slug, custom domain, color, logo) |
| **Articles** | Rich-text articles with title, content, description, slug, status (draft/published/archived), locale, position |
| **Categories** | Organize articles into categories with ordering |
| **Related Categories** | Cross-link categories |
| **Folders** | Hierarchical article organization |
| **Portal Frontend** | Public-facing help center with search, locale switcher, table of contents |
| **Article Search** | Full-text search across published articles |

### 1.10 Reporting & Analytics

| Feature | Description |
|---------|-------------|
| **Conversation Reports** | Metrics: count, resolution time, first response time, by date range |
| **Agent Reports** | Per-agent performance metrics |
| **Inbox Reports** | Per-inbox volume and performance |
| **Team Reports** | Per-team metrics |
| **Label Reports** | Metrics grouped by label |
| **Summary Reports** | Overview dashboards with key metrics |
| **CSAT Reports** | Satisfaction scores, response rates, filterable by date/agent/inbox/team |
| **Reporting Events** | Event pipeline for analytics data capture |
| **Audit Logs** | Activity trail for admin actions (Enterprise) |
| **Conversation Traffic** | Heatmap of conversation volume by hour/day |
| **Bot Metrics** | Bot interaction performance |

### 1.11 Integrations (2)

| Integration | Description |
|-------------|-------------|
| **Webhooks** | Register webhook URLs to receive events on the local network: conversation_created, conversation_updated, conversation_status_changed, message_created, message_updated, contact_created, contact_updated, webwidget_triggered, inbox_created, inbox_updated. Supports account-level and inbox-level webhooks with HMAC-signed secrets |
| **Rasa NLU Bot** | Connect self-hosted **Rasa Open Source** agent to inboxes. Auto-detects intent from customer messages via on-premise NLU, sends fulfillment responses. Supports handoff to human agent on specific intents. Fully offline — no cloud dependency |

### 1.12 Platform API

| Feature | Description |
|---------|-------------|
| **Platform Apps** | Register third-party apps with API keys |
| **Platform Permissions** | Grant apps access to specific accounts or users |
| **Platform API Endpoints** | CRUD users, accounts, account_users, agent_bots across tenants |

### 1.13 Authentication & Security

| Feature | Description |
|---------|-------------|
| **Email/Password Auth** | Registration, login, password reset, email confirmation |
| **JWT Tokens** | Access + refresh token pattern for API auth |
| **API Access Tokens** | Long-lived tokens for programmatic access |
| **SAML SSO (Enterprise)** | SAML 2.0 identity provider integration with role mapping |
| **MFA/2FA** | TOTP with backup codes |
| **Authorization Policies** | 23+ resource-level policies (conversation, contact, inbox, team, label, report, webhook, etc.) |
| **Super Admin** | Instance-level admin with full access |
| **Rate Limiting** | Per-IP and per-account throttling (login, signup, API, widget, search, reports) |

### 1.14 Real-Time Features

| Feature | Description |
|---------|-------------|
| **WebSocket Hub** | SignalR hub for real-time message delivery, scoped by account |
| **Typing Indicators** | Real-time "agent/contact is typing" status |
| **Presence** | Online/offline/busy status for agents |
| **Event Broadcasting** | message_created, message_updated, conversation_updated, notification events pushed to connected clients |
| **Pubsub Tokens** | Token-based channel subscription for widget clients |

### 1.15 Notifications

| Feature | Description |
|---------|-------------|
| **In-App Notifications** | Types: conversation_creation, assignment, new_message, mention, participating_conversation_new_message. Read/unread, snooze, bulk read/delete |
| **Push Notifications** | Self-hosted Web Push (VAPID keys, custom push relay server — no FCM dependency) |
| **Email Notifications** | Agent, admin, and team email alerts for conversation events |
| **Notification Preferences** | Per-user toggle for each notification type x delivery channel (email, push) |

### 1.16 Widget (Customer-Facing)

| Feature | Description |
|---------|-------------|
| **Chat Widget** | Embeddable web component for customer websites |
| **Pre-chat Forms** | Configurable fields (email, name, phone, custom) with validation |
| **Widget Config API** | Configure appearance, greeting, features via API |
| **CSAT Survey UI** | Post-conversation rating interface |
| **Widget Campaigns** | Display proactive campaign messages in widget |
| **Unread Messages** | Show unread message count badge |
| **Agent Availability** | Display team/agent availability status |

### 1.17 Administration

| Feature | Description |
|---------|-------------|
| **Multi-Tenancy** | Account-scoped data isolation |
| **Super Admin Panel** | Manage accounts, users, platform apps, installation config, agent bots |
| **Account Settings** | Name, locale, domain, feature toggles, auto-resolve config |
| **Custom Domains (Enterprise)** | Custom domain per portal with SSL verification |
| **Email Templates** | Customizable email notification templates |
| **Installation Config** | Instance-wide settings (global feature toggles, branding) |

### 1.18 Background Processing & Events

| Feature | Description |
|---------|-------------|
| **Event System** | Domain event bus with 12+ listeners: real-time broadcast, automation triggers, bot dispatching, campaign triggers, CSAT surveys, webhook firing, notification dispatch, reporting events |
| **Background Jobs** | 30+ async job types: email delivery, webhook calls, IMAP polling, contact import, contact IP lookup, CSAT surveys, campaign execution, assignment, avatar fetching, data cleanup |
| **Scheduled Tasks** | Cron-based: auto-resolve conversations, reopen snoozed, trigger campaigns, IMAP email fetch, template sync |

---

## 2. TECHNOLOGY STACK

### 2.1 Backend — .NET 8 (ASP.NET Core)

| Concern | Technology | NuGet Package |
|---------|-----------|---------------|
| Web Framework | ASP.NET Core 8 Web API | `Microsoft.AspNetCore.App` |
| ORM | Entity Framework Core (Code-First) | `Microsoft.EntityFrameworkCore`, `Npgsql.EntityFrameworkCore.PostgreSQL` |
| Database | PostgreSQL | `Npgsql` |
| Real-Time | SignalR | `Microsoft.AspNetCore.SignalR` |
| Authentication | ASP.NET Core Identity + JWT | `Microsoft.AspNetCore.Identity.EntityFrameworkCore`, `Microsoft.AspNetCore.Authentication.JwtBearer` |
| Authorization | Policy-based auth | Built-in `IAuthorizationHandler` |
| Background Jobs | Hangfire | `Hangfire.Core`, `Hangfire.PostgreSql` |
| Message Bus / CQRS | MediatR | `MediatR` |
| Email (SMTP) | MailKit | `MailKit` |
| Email (IMAP) | MailKit | `MailKit` (IMAP client) |
| File Storage | MinIO (S3-compatible, self-hosted) | `AWSSDK.S3` (MinIO is S3-compatible, same SDK) |
| Redis Cache | StackExchange.Redis | `StackExchange.Redis`, `Microsoft.Extensions.Caching.StackExchangeRedis` |
| Search | PostgreSQL Full-Text | Built-in with Npgsql |
| Vector Search | pgvector | `Pgvector.EntityFrameworkCore` |
| Rate Limiting | ASP.NET Core Rate Limiting | Built-in (`Microsoft.AspNetCore.RateLimiting`) |
| SAML SSO | Sustainsys.Saml2 | `Sustainsys.Saml2.AspNetCore2` |
| TOTP (MFA) | OtpNet | `OtpNet` |
| Geocoding | MaxMind GeoLite2 (offline `.mmdb` file) | `MaxMind.GeoIP2` (reads local database, no internet needed) |
| Liquid Templates | Fluid | `Fluid.Core` |
| Push Notifications | Self-hosted Web Push (VAPID) | `WebPush` (`web-push-csharp`) — standards-based, no FCM dependency |
| NLU Bot Integration | Rasa Open Source (self-hosted) | Custom HTTP client to local Rasa REST API (`/webhooks/rest/webhook`) |
| Logging | Serilog | `Serilog.AspNetCore`, `Serilog.Sinks.Console`, `Serilog.Sinks.File` (or Seq self-hosted / Grafana Loki) |
| Health Checks | ASP.NET Core Health Checks | `AspNetCore.HealthChecks.NpgSql`, `AspNetCore.HealthChecks.Redis` |
| API Documentation | Swagger / OpenAPI | `Swashbuckle.AspNetCore` |
| Testing | xUnit + Moq | `xUnit`, `Moq`, `Microsoft.AspNetCore.Mvc.Testing` |

### 2.2 Frontend — Angular 17+

| Concern | Technology | npm Package |
|---------|-----------|-------------|
| Framework | Angular 17+ (standalone components) | `@angular/core` |
| State Management | NgRx (Redux pattern) | `@ngrx/store`, `@ngrx/effects`, `@ngrx/entity` |
| Routing | Angular Router (lazy-loaded) | `@angular/router` |
| HTTP | Angular HttpClient + interceptors | `@angular/common/http` |
| i18n | ngx-translate | `@ngx-translate/core`, `@ngx-translate/http-loader` |
| CSS Framework | Tailwind CSS | `tailwindcss`, `@tailwindcss/forms`, `@tailwindcss/typography` |
| Charts | ng2-charts (Chart.js) | `ng2-charts`, `chart.js` |
| Rich Text Editor | ProseMirror or TipTap | `@tiptap/core`, `@tiptap/starter-kit` |
| Icons | Material Design Icons | `@mdi/font` or `@iconify/angular` |
| Real-Time | SignalR Client | `@microsoft/signalr` |
| Date Handling | date-fns | `date-fns` |
| Forms | Reactive Forms | Built-in `@angular/forms` |
| Notifications | Web Push API | Built-in browser API |
| Widget | Angular Elements | `@angular/elements` |
| SSR (Portal) | Angular SSR | `@angular/ssr` |
| Testing | Jasmine + Karma, Playwright | `jasmine-core`, `karma`, `@playwright/test` |

---

## 3. PROJECT STRUCTURE

```
/
├── src/
│   ├── CustomerEngagement.Api/                    # ASP.NET Core Web API
│   │   ├── Controllers/
│   │   │   ├── V1/
│   │   │   │   ├── AccountsController.cs
│   │   │   │   ├── ConversationsController.cs
│   │   │   │   ├── MessagesController.cs
│   │   │   │   ├── ContactsController.cs
│   │   │   │   ├── InboxesController.cs
│   │   │   │   ├── TeamsController.cs
│   │   │   │   ├── AgentsController.cs
│   │   │   │   ├── LabelsController.cs
│   │   │   │   ├── AutomationRulesController.cs
│   │   │   │   ├── MacrosController.cs
│   │   │   │   ├── CampaignsController.cs
│   │   │   │   ├── CannedResponsesController.cs
│   │   │   │   ├── WebhooksController.cs
│   │   │   │   ├── ReportsController.cs
│   │   │   │   ├── NotificationsController.cs
│   │   │   │   ├── SearchController.cs
│   │   │   │   ├── PortalsController.cs
│   │   │   │   ├── ArticlesController.cs
│   │   │   │   ├── CategoriesController.cs
│   │   │   │   ├── CustomAttributesController.cs
│   │   │   │   ├── CustomFiltersController.cs
│   │   │   │   ├── ProfilesController.cs
│   │   │   │   ├── BulkActionsController.cs
│   │   │   │   └── CsatSurveyController.cs
│   │   │   ├── Widget/
│   │   │   │   ├── WidgetConversationsController.cs
│   │   │   │   ├── WidgetMessagesController.cs
│   │   │   │   ├── WidgetConfigController.cs
│   │   │   │   └── WidgetContactsController.cs
│   │   │   ├── Platform/
│   │   │   │   ├── PlatformUsersController.cs
│   │   │   │   ├── PlatformAccountsController.cs
│   │   │   │   └── PlatformAgentBotsController.cs
│   │   │   ├── Public/
│   │   │   │   ├── PublicInboxesController.cs
│   │   │   │   ├── PublicPortalsController.cs
│   │   │   │   └── PublicCsatController.cs
│   │   │   └── SuperAdmin/
│   │   │       ├── AdminAccountsController.cs
│   │   │       ├── AdminUsersController.cs
│   │   │       └── AdminConfigController.cs
│   │   ├── Hubs/
│   │   │   └── ConversationHub.cs               # SignalR hub
│   │   ├── Middleware/
│   │   │   ├── TenantMiddleware.cs
│   │   │   ├── RequestLoggingMiddleware.cs
│   │   │   └── ExceptionHandlingMiddleware.cs
│   │   ├── Filters/
│   │   │   └── AuthorizationFilters.cs
│   │   └── Program.cs
│   │
│   ├── CustomerEngagement.Core/                   # Domain layer
│   │   ├── Entities/
│   │   │   ├── Account.cs
│   │   │   ├── User.cs
│   │   │   ├── AccountUser.cs
│   │   │   ├── Conversation.cs
│   │   │   ├── Message.cs
│   │   │   ├── Attachment.cs
│   │   │   ├── Contact.cs
│   │   │   ├── ContactInbox.cs
│   │   │   ├── Inbox.cs
│   │   │   ├── InboxMember.cs
│   │   │   ├── Team.cs
│   │   │   ├── TeamMember.cs
│   │   │   ├── Label.cs
│   │   │   ├── Note.cs
│   │   │   ├── Mention.cs
│   │   │   ├── ConversationParticipant.cs
│   │   │   ├── AutomationRule.cs
│   │   │   ├── Macro.cs
│   │   │   ├── Campaign.cs
│   │   │   ├── CannedResponse.cs
│   │   │   ├── Webhook.cs
│   │   │   ├── CustomAttributeDefinition.cs
│   │   │   ├── CustomFilter.cs
│   │   │   ├── Notification.cs
│   │   │   ├── NotificationSetting.cs
│   │   │   ├── NotificationSubscription.cs
│   │   │   ├── CsatSurveyResponse.cs
│   │   │   ├── ReportingEvent.cs
│   │   │   ├── WorkingHour.cs
│   │   │   ├── AssignmentPolicy.cs
│   │   │   ├── Portal.cs
│   │   │   ├── Article.cs
│   │   │   ├── Category.cs
│   │   │   ├── Folder.cs
│   │   │   ├── AgentBot.cs
│   │   │   ├── AgentBotInbox.cs
│   │   │   ├── AccessToken.cs
│   │   │   ├── DataImport.cs
│   │   │   ├── EmailTemplate.cs
│   │   │   ├── InstallationConfig.cs
│   │   │   ├── PlatformApp.cs
│   │   │   ├── PlatformAppPermissible.cs
│   │   │   ├── IntegrationHook.cs
│   │   │   ├── Channels/
│   │   │   │   ├── ChannelWebWidget.cs
│   │   │   │   ├── ChannelEmail.cs
│   │   │   │   └── ChannelApi.cs
│   │   │   └── SuperAdmin.cs
│   │   ├── Enums/
│   │   │   ├── ConversationStatus.cs            # Open, Resolved, Pending, Snoozed
│   │   │   ├── ConversationPriority.cs          # None, Low, Medium, High, Urgent
│   │   │   ├── MessageType.cs                   # Incoming, Outgoing, Activity, Template
│   │   │   ├── ContactType.cs                   # Visitor, Lead, Customer
│   │   │   ├── UserAvailability.cs              # Online, Offline, Busy
│   │   │   ├── UserRole.cs                      # Administrator, Agent
│   │   │   ├── AttachmentType.cs                # Image, Audio, Video, File, Location
│   │   │   ├── CampaignType.cs                  # Ongoing, OneOff
│   │   │   ├── ArticleStatus.cs                 # Draft, Published, Archived
│   │   │   └── WebhookEventType.cs              # All supported event types
│   │   ├── Interfaces/
│   │   │   ├── IRepository.cs                   # Generic repository
│   │   │   ├── IUnitOfWork.cs
│   │   │   └── Services/                        # Service contracts
│   │   └── Events/
│   │       ├── ConversationCreatedEvent.cs
│   │       ├── MessageCreatedEvent.cs
│   │       ├── ContactCreatedEvent.cs
│   │       └── ...                              # Domain events for MediatR
│   │
│   ├── CustomerEngagement.Application/            # Application/service layer
│   │   ├── Services/
│   │   │   ├── Conversations/
│   │   │   │   ├── ConversationService.cs
│   │   │   │   ├── MessageService.cs
│   │   │   │   ├── AssignmentService.cs
│   │   │   │   ├── FilterService.cs
│   │   │   │   └── TypingStatusManager.cs
│   │   │   ├── Contacts/
│   │   │   │   ├── ContactService.cs
│   │   │   │   ├── ContactMergeService.cs
│   │   │   │   ├── ContactImportService.cs
│   │   │   │   └── ContactSearchService.cs
│   │   │   ├── Channels/
│   │   │   │   ├── WebWidgetService.cs
│   │   │   │   ├── EmailChannelService.cs
│   │   │   │   └── ApiChannelService.cs
│   │   │   ├── Automations/
│   │   │   │   ├── AutomationRuleEngine.cs
│   │   │   │   ├── MacroExecutionService.cs
│   │   │   │   └── CampaignService.cs
│   │   │   ├── Integrations/
│   │   │   │   ├── WebhookService.cs
│   │   │   │   └── RasaNluService.cs
│   │   │   ├── Reporting/
│   │   │   │   ├── ReportBuilder.cs
│   │   │   │   └── CsatReportService.cs
│   │   │   ├── Notifications/
│   │   │   │   ├── NotificationService.cs
│   │   │   │   ├── PushNotificationService.cs
│   │   │   │   └── EmailNotificationService.cs
│   │   │   ├── HelpCenter/
│   │   │   │   ├── PortalService.cs
│   │   │   │   └── ArticleService.cs
│   │   │   └── Search/
│   │   │       └── GlobalSearchService.cs
│   │   ├── Commands/                              # CQRS commands (MediatR)
│   │   ├── Queries/                               # CQRS queries (MediatR)
│   │   ├── EventHandlers/                         # Domain event handlers
│   │   │   ├── BroadcastEventHandler.cs           # SignalR broadcasting
│   │   │   ├── AutomationEventHandler.cs          # Trigger automation rules
│   │   │   ├── BotEventHandler.cs                 # Dispatch to agent bots
│   │   │   ├── WebhookEventHandler.cs             # Fire outgoing webhooks
│   │   │   ├── NotificationEventHandler.cs        # Create notifications
│   │   │   ├── CsatEventHandler.cs                # Trigger CSAT surveys
│   │   │   ├── CampaignEventHandler.cs            # Campaign triggers
│   │   │   └── ReportingEventHandler.cs           # Capture analytics events
│   │   ├── BackgroundJobs/                        # Hangfire jobs
│   │   │   ├── ImapEmailFetchJob.cs
│   │   │   ├── WebhookDeliveryJob.cs
│   │   │   ├── ContactIpLookupJob.cs
│   │   │   ├── ConversationAutoResolveJob.cs
│   │   │   ├── ReopenSnoozedConversationsJob.cs
│   │   │   ├── CampaignTriggerJob.cs
│   │   │   ├── DataImportJob.cs
│   │   │   └── CleanupJob.cs
│   │   └── DTOs/                                  # Data Transfer Objects
│   │
│   ├── CustomerEngagement.Infrastructure/         # Data access & external services
│   │   ├── Persistence/
│   │   │   ├── AppDbContext.cs
│   │   │   ├── Configurations/                    # EF Core entity configurations
│   │   │   └── Migrations/
│   │   ├── Repositories/
│   │   ├── ExternalServices/
│   │   │   ├── Email/
│   │   │   │   ├── SmtpEmailSender.cs
│   │   │   │   └── ImapEmailReceiver.cs
│   │   │   ├── Storage/
│   │   │   │   └── MinioStorageService.cs       # S3-compatible, self-hosted
│   │   │   ├── NluBot/
│   │   │   │   └── RasaClient.cs                # Self-hosted Rasa REST API
│   │   │   ├── GeoIp/
│   │   │   │   └── MaxMindOfflineGeoIpService.cs # Reads local .mmdb file
│   │   │   ├── Llm/
│   │   │   │   └── OllamaClient.cs              # Self-hosted LLM (OpenAI-compat API)
│   │   │   └── Push/
│   │   │       └── VapidWebPushService.cs       # VAPID Web Push, no FCM
│   │   └── Identity/
│   │       ├── IdentityService.cs
│   │       ├── JwtTokenService.cs
│   │       └── SamlAuthService.cs
│   │
│   └── CustomerEngagement.Enterprise/             # Enterprise features (separate assembly)
│       ├── Captain/
│       │   ├── Entities/
│       │   │   ├── CaptainAssistant.cs
│       │   │   ├── CaptainDocument.cs
│       │   │   ├── CaptainScenario.cs
│       │   │   ├── CaptainCustomTool.cs
│       │   │   ├── CaptainInbox.cs
│       │   │   ├── CopilotThread.cs
│       │   │   ├── CopilotMessage.cs
│       │   │   └── ArticleEmbedding.cs
│       │   ├── Services/
│       │   │   ├── AssistantChatService.cs
│       │   │   ├── CopilotService.cs
│       │   │   ├── EmbeddingService.cs
│       │   │   └── ToolRegistryService.cs
│       │   └── Controllers/
│       │       ├── CaptainAssistantsController.cs
│       │       ├── CaptainDocumentsController.cs
│       │       └── CopilotController.cs
│       ├── CustomRoles/
│       │   ├── Entities/
│       │   │   └── CustomRole.cs
│       │   └── Services/
│       │       └── CustomRoleService.cs
│       └── Saml/
│           ├── Entities/
│           │   └── AccountSamlSettings.cs
│           └── Services/
│               └── SamlAuthenticationService.cs
│
├── frontend/
│   ├── dashboard/                                 # Angular 17+ Dashboard App
│   │   ├── src/app/
│   │   │   ├── core/
│   │   │   │   ├── guards/                        # Auth, role guards
│   │   │   │   ├── interceptors/                  # JWT, error interceptors
│   │   │   │   ├── services/                      # Auth, SignalR, API base
│   │   │   │   └── models/                        # TypeScript interfaces
│   │   │   ├── shared/
│   │   │   │   ├── components/                    # Reusable UI components
│   │   │   │   ├── pipes/                         # Custom pipes
│   │   │   │   └── directives/                    # Custom directives
│   │   │   ├── features/
│   │   │   │   ├── auth/                          # Login, register, forgot password
│   │   │   │   ├── conversations/                 # Conversation list, chat, filters
│   │   │   │   ├── contacts/                      # Contact list, detail, merge
│   │   │   │   ├── inboxes/                       # Inbox setup, config
│   │   │   │   ├── settings/                      # All settings pages
│   │   │   │   ├── reports/                       # Analytics dashboards
│   │   │   │   ├── help-center/                   # Portal, article management
│   │   │   │   ├── notifications/                 # Notification center
│   │   │   │   ├── captain/                       # AI assistant management
│   │   │   │   └── super-admin/                   # Instance admin
│   │   │   └── store/                             # NgRx store
│   │   │       ├── conversations/
│   │   │       ├── contacts/
│   │   │       ├── inboxes/
│   │   │       ├── agents/
│   │   │       ├── teams/
│   │   │       ├── labels/
│   │   │       ├── notifications/
│   │   │       ├── reports/
│   │   │       └── auth/
│   │   ├── tailwind.config.js
│   │   └── angular.json
│   │
│   ├── widget/                                    # Angular Elements (Web Component)
│   │   └── src/app/
│   │       ├── components/
│   │       │   ├── chat-window/
│   │       │   ├── message-bubble/
│   │       │   ├── pre-chat-form/
│   │       │   ├── csat-survey/
│   │       │   └── typing-indicator/
│   │       └── services/
│   │           ├── widget-api.service.ts
│   │           └── signalr.service.ts
│   │
│   └── portal/                                    # Angular SSR (Help Center)
│       └── src/app/
│           ├── pages/
│           │   ├── home/
│           │   ├── article/
│           │   ├── category/
│           │   └── search/
│           └── services/
│               └── portal-api.service.ts
│
├── tests/
│   ├── CustomerEngagement.Api.Tests/              # API integration tests
│   ├── CustomerEngagement.Application.Tests/      # Service unit tests
│   ├── CustomerEngagement.Infrastructure.Tests/   # Repository tests
│   └── e2e/                                       # Playwright E2E tests
│
├── docker-compose.yml                             # PostgreSQL, Redis, app
├── .github/workflows/                             # CI/CD pipelines
└── README.md
```

---

## 4. DATA ENTITIES (45+ Models)

**Core:** Account, User, AccountUser, AccessToken, SuperAdmin, InstallationConfig
**Messaging:** Conversation, Message, Attachment, ConversationParticipant, Mention
**Contacts:** Contact, ContactInbox, Note, CustomAttributeDefinition, DataImport
**Channels (3):** ChannelWebWidget, ChannelEmail, ChannelApi
**Inbox:** Inbox, InboxMember, WorkingHour, AssignmentPolicy
**Teams:** Team, TeamMember
**Automation:** AutomationRule, Macro, Campaign, CannedResponse
**Labels:** Label
**Notifications:** Notification, NotificationSetting, NotificationSubscription
**Reporting:** ReportingEvent, CsatSurveyResponse
**Integrations:** IntegrationHook, Webhook
**Help Center:** Portal, Article, Category, RelatedCategory, Folder
**Platform:** PlatformApp, PlatformAppPermissible
**Bots:** AgentBot, AgentBotInbox
**Filters:** CustomFilter
**Email:** EmailTemplate
**Enterprise:** CustomRole, AccountSamlSettings, CaptainAssistant, CaptainDocument, CaptainScenario, CaptainCustomTool, CaptainInbox, CopilotThread, CopilotMessage, ArticleEmbedding

---

## 5. PHASED IMPLEMENTATION ROADMAP

### Phase 1: Foundation (Weeks 1–4)
- [ ] .NET solution scaffolding (Clean Architecture, 5 projects)
- [ ] EF Core DbContext with all entities & migrations (PostgreSQL)
- [ ] ASP.NET Core Identity setup (registration, login, JWT access + refresh tokens)
- [ ] Multi-tenancy middleware (Account-scoped data isolation via tenant header/claim)
- [ ] SignalR hub for real-time messaging
- [ ] Redis setup for caching and presence tracking
- [ ] Angular dashboard project with routing, NgRx, Tailwind, i18n
- [ ] Auth pages (login, register, forgot password, email confirmation)

### Phase 2: Core Messaging (Weeks 5–8)
- [ ] Conversation entity & CRUD APIs (create, list, show, update, status transitions)
- [ ] Message entity with attachments (MinIO self-hosted S3-compatible upload, presigned download URLs)
- [ ] Web Widget channel (channel config, widget token, contact creation)
- [ ] Real-time message delivery via SignalR (account-scoped groups)
- [ ] Agent assignment (manual + round-robin auto-assignment)
- [ ] Angular conversation list & chat UI (message bubbles, reply box, sidebar)
- [ ] Embeddable widget app (Angular Elements web component)
- [ ] Typing indicators & agent presence (Redis-backed)

### Phase 3: Contact & Inbox Management (Weeks 9–11)
- [ ] Contact CRUD, full-text search, merge, CSV import
- [ ] Custom attribute definitions & values for contacts and conversations
- [ ] Inbox CRUD & channel configuration
- [ ] Inbox members, working hours (per-day schedule with timezone)
- [ ] Labels for conversations & contacts
- [ ] Teams & team members, team assignment
- [ ] Angular settings pages (inboxes, teams, agents, labels, custom attributes)

### Phase 4: Additional Channels (Weeks 12–13)
- [ ] Email channel: IMAP polling (MailKit), inbound email → conversation creation, email threading
- [ ] Email channel: SMTP outbound (agent replies sent as email)
- [ ] API channel: REST endpoints for external systems to send/receive messages, HMAC auth

### Phase 5: Automation & Productivity (Weeks 14–16)
- [ ] Canned responses (CRUD + search by short code)
- [ ] Macros (CRUD + one-click execution of saved action sequences)
- [ ] Automation rules engine (event → conditions → actions, AND/OR logic)
- [ ] Campaigns (ongoing + one-off scheduled, with audience targeting)
- [ ] CSAT surveys (auto-trigger after resolution, rating + feedback collection)
- [ ] Bulk actions on conversations (assign, label, resolve, status change)
- [ ] Custom filters / saved views

### Phase 6: Integrations (Weeks 17–18)
- [ ] Webhook system: CRUD for webhook registrations, event subscriptions, signed payloads, async delivery via background job
- [ ] Rasa NLU integration: connect self-hosted Rasa bot to inbox, intent detection on incoming messages, fulfillment responses, handoff to human

### Phase 7: Reporting & Help Center (Weeks 19–21)
- [ ] Reporting event pipeline (capture events on conversation/message lifecycle)
- [ ] Report APIs: conversation, agent, inbox, team, label reports with date ranges
- [ ] CSAT reports with filtering
- [ ] Summary/overview dashboard reports
- [ ] Help Center: Portal CRUD (name, slug, logo, custom domain)
- [ ] Help Center: Article & Category CRUD with ordering
- [ ] Public help center Angular SSR app (search, locale, SEO)

### Phase 8: Notifications & Administration (Weeks 22–23)
- [ ] In-app notification system (create, list, read/unread, bulk operations)
- [ ] Push notifications (self-hosted VAPID Web Push — no FCM)
- [ ] Email notifications (agent, admin, team alerts)
- [ ] Notification preferences (per-user, per-type, per-channel toggle)
- [ ] Super Admin panel (account, user, bot, config management)
- [ ] Account settings & branding
- [ ] Audit logs (track admin actions)
- [ ] Agent bots (webhook-based, assign to inboxes)

### Phase 9: Enterprise Features (Weeks 24–27)
- [ ] Custom Roles: define permission sets, assign to agents
- [ ] SAML SSO: SAML 2.0 IdP integration, role mapping, SP metadata
- [ ] Captain AI assistants: CRUD, configure temperature/guidelines/guardrails
- [ ] Captain documents: upload PDFs/text to MinIO, generate embeddings via self-hosted LLM (pgvector)
- [ ] Captain scenarios & custom tools (function calling)
- [ ] Copilot: agent-facing AI (reply suggestions, rewrite, summarize, label suggestions)
- [ ] Captain inbox connection (auto-respond via AI on selected inboxes)

### Phase 10: Platform API & Polish (Weeks 28–30)
- [ ] Platform API: multi-tenant CRUD for users, accounts, bots
- [ ] Platform apps & permissions
- [ ] MFA/2FA (TOTP enrollment, verification, backup codes)
- [ ] Rate limiting (per-IP, per-account, endpoint-specific rules)
- [ ] Security hardening (CORS, CSP, input sanitization, SQL injection prevention)
- [ ] Performance optimization (query optimization, caching, connection pooling)
- [ ] E2E testing (Playwright: all key flows)
- [ ] API documentation (Swagger/OpenAPI)

---

## 6. VERIFICATION / TESTING STRATEGY

### Unit Tests (xUnit + Moq / Jasmine)
- Service layer: all business logic methods
- Event handlers: verify correct events trigger correct actions
- Authorization policies: verify access control for each role

### Integration Tests (WebApplicationFactory)
- API endpoints: request/response validation for all controllers
- Database: EF Core operations with test PostgreSQL
- SignalR: hub connection and message broadcasting

### E2E Tests (Playwright)
1. Agent login → view conversations → reply to message → see real-time update
2. Widget loads on external site → customer sends message → agent sees it in real-time
3. Create inbox (Web Widget / Email / API) → configure → receive message through channel
4. Email channel: inbound IMAP pickup → creates conversation → agent replies → SMTP outbound
5. API channel: external system sends message via REST → conversation created → reply sent back
6. Automation rule triggers on new conversation → auto-assigns to team
7. Contact search, merge, and custom attributes work correctly
8. Reports load with correct aggregated data
9. Help center article CRUD and public portal rendering
10. Webhook fires on conversation/message/contact events
11. Rasa NLU bot auto-responds to customer messages in connected inbox
12. CSAT survey sent after resolution and responses recorded

---

## 7. INFRASTRUCTURE & DEPLOYMENT (Offline / Air-Gapped)

### Architecture Overview — All Self-Hosted

```
┌─────────────────────────────────────────────────────────────┐
│                    ISOLATED NETWORK                          │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐  │
│  │ Nginx/   │  │ .NET API  │  │ Hangfire  │  │ Angular   │  │
│  │ Reverse  │──│ Server    │  │ Worker    │  │ Dashboard │  │
│  │ Proxy    │  └─────┬─────┘  └─────┬─────┘  │ (static)  │  │
│  └──────────┘        │              │         └───────────┘  │
│                      │              │                        │
│  ┌──────────┐  ┌─────┴─────┐  ┌────┴──────┐  ┌──────────┐  │
│  │ MinIO    │  │PostgreSQL │  │   Redis   │  │ Rasa     │  │
│  │(S3-compat│  │  + pgvec  │  │           │  │ NLU Bot  │  │
│  │ storage) │  └───────────┘  └───────────┘  └──────────┘  │
│                                                             │
│  ┌──────────┐  ┌───────────┐  ┌───────────┐                │
│  │ Ollama / │  │ Keycloak  │  │ Mail Srv  │                │
│  │ vLLM     │  │ (SAML IdP)│  │ (Postfix) │                │
│  │ (LLM)    │  └───────────┘  └───────────┘                │
│  └──────────┘                                               │
└─────────────────────────────────────────────────────────────┘
```

### Docker Compose (All Services)
```yaml
services:
  # --- Core Application ---
  api:
    image: customer-engagement-api:latest
    ports: ["5000:5000"]
    depends_on: [db, redis, minio]

  worker:
    image: customer-engagement-worker:latest
    depends_on: [db, redis, minio]

  dashboard:
    image: customer-engagement-dashboard:latest    # nginx serving Angular static build
    ports: ["4200:80"]

  widget:
    image: customer-engagement-widget:latest
    ports: ["4201:80"]

  portal:
    image: customer-engagement-portal:latest       # Angular SSR
    ports: ["4202:4000"]

  # --- Data Layer ---
  db:
    image: pgvector/pgvector:pg16                  # PostgreSQL 16 + pgvector
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  # --- Object Storage (replaces S3/Azure) ---
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    ports: ["9000:9000", "9001:9001"]
    volumes: ["minio-data:/data"]

  # --- NLU Bot (replaces Dialogflow) ---
  rasa:
    image: rasa/rasa:latest
    ports: ["5005:5005"]
    volumes: ["./rasa-models:/app/models"]

  # --- Self-hosted LLM (replaces OpenAI) ---
  ollama:
    image: ollama/ollama:latest
    ports: ["11434:11434"]
    volumes: ["ollama-models:/root/.ollama"]
    # Pre-load models: ollama pull llama3, mistral, etc.

  # --- Identity Provider (replaces cloud SSO) ---
  keycloak:
    image: quay.io/keycloak/keycloak:latest
    command: start-dev
    ports: ["8080:8080"]

  # --- Local Mail Server ---
  mailserver:
    image: mailserver/docker-mailserver:latest
    ports: ["25:25", "143:143", "587:587"]
    volumes: ["mail-data:/var/mail"]

  # --- Reverse Proxy ---
  nginx:
    image: nginx:alpine
    ports: ["80:80", "443:443"]
    volumes: ["./nginx.conf:/etc/nginx/nginx.conf", "./certs:/etc/nginx/certs"]

volumes:
  pgdata:
  minio-data:
  ollama-models:
  mail-data:
```

### Production Deployment (Offline)
- **Container runtime**: Docker / Podman / K3s (lightweight Kubernetes) on bare-metal or VMs
- **All images pre-built and loaded** via `docker save` / `docker load` (no registry pull at runtime)
- **Optional local registry**: Harbor or Docker Registry v2 on the isolated network
- **TLS**: Self-signed CA or internal PKI — distribute root CA cert to all nodes and browsers
- **DNS**: Local DNS server (CoreDNS/dnsmasq) for service name resolution
- **Backup**: PostgreSQL `pg_dump`, MinIO `mc mirror`, Redis RDB snapshots — all to local NAS/SAN

### Offline Preparation Checklist
Before deploying to the air-gapped network:
- [ ] Build all Docker images on an internet-connected machine
- [ ] `docker save` all images to `.tar` files for transfer
- [ ] Download all npm packages (`npm pack` or use Verdaccio cache)
- [ ] Download all NuGet packages (`dotnet restore` + copy cache, or use BaGet)
- [ ] Download MaxMind GeoLite2 `.mmdb` database file
- [ ] Download and convert Rasa NLU model files
- [ ] Download Ollama model weights (e.g., `ollama pull llama3`)
- [ ] Generate TLS certificates (self-signed CA + server certs)
- [ ] Generate VAPID key pair for Web Push
- [ ] Prepare Keycloak realm export (if using SAML SSO)
- [ ] Transfer all artifacts to isolated network via approved media

### Environment Variables (Key)
```
# Database
DATABASE_URL=postgresql://user:pass@db:5432/engagement
REDIS_URL=redis://redis:6379/0

# Auth
JWT_SECRET=<generate-strong-secret>
JWT_EXPIRY_MINUTES=60
REFRESH_TOKEN_EXPIRY_DAYS=30

# Storage (MinIO — S3-compatible)
STORAGE_PROVIDER=s3
S3_ENDPOINT=http://minio:9000
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_BUCKET=attachments
S3_FORCE_PATH_STYLE=true

# Email (local mail server)
SMTP_HOST=mailserver
SMTP_PORT=587
IMAP_HOST=mailserver
IMAP_PORT=143

# NLU Bot (Rasa — replaces Dialogflow)
RASA_URL=http://rasa:5005

# Self-hosted LLM (Ollama — replaces OpenAI)
LLM_PROVIDER=ollama
OLLAMA_URL=http://ollama:11434
OLLAMA_MODEL=llama3

# GeoIP (offline database)
GEOIP_DATABASE_PATH=/data/GeoLite2-City.mmdb

# Web Push (VAPID — replaces FCM)
VAPID_PUBLIC_KEY=<generate>
VAPID_PRIVATE_KEY=<generate>
VAPID_SUBJECT=mailto:admin@engagement.local

# SAML SSO (Keycloak — self-hosted IdP)
SAML_IDP_METADATA_URL=http://keycloak:8080/realms/engagement/protocol/saml/descriptor
```
