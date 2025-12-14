# XXII Chat

- Unified Messaging
- File Exchange
- Article Publishing

Tech stack:
- Languages: JavaScript, TypeScript, some critical code may be in C, C++, Rust, etc.
- Local-first architecture with [globalstorage](https://github.com/metarhia/Docs/blob/main/content/en/GLOBALSTORAGE.md) database with sync engine
- Client: PWA + (React & Vue.js & no framework)
- Storage: CRDT, CAS, OPFS, Blockchain
- Network: WebSocket, WebRTC, HTTPS
- Server: metarhia & fastify

## Product scope

* Messaging: public channels, private chats (1:1)
* Publishing: long-form posts (articles) with threaded replies
* File exchange: attachments everywhere, file publishing, folders
* Feeds: public feeds (channels) and channel timelines
* Discovery: full-text search across chats, feeds, posts, files metadata
* Engagement: reactions, voting/polls, forwarding, pinned posts
* See [Future features](#future-features) section for planned enhancements: voice/video calls, AI integration, bots, and more
UI solution: terminal-like text interface with modern controls.

## Core entities and relationships

Domain Model in Metaschema

### Author

Functional requirements:
* Registration and authentication
* Profile management: nick, name, bio, icon, photo
* Status management: change status (see schema)
* Settings management: update free-form object
* Author lookup: by uuid, nick, name

Schema:
```javascript
({
  nick: { type: 'string', unique: true },
  name: 'string',
  bio: 'string',
  icon: 'string',
  photo: 'File',
  status: { enum: ['online', 'away', 'dnd', 'offline', 'unknown'], default: 'offline' },
  created: 'datetime',
  updated: 'datetime',
  settings: 'object',
});
```

### Node

Functional requirements:
* Node: server registration and management
* Unique identification by name, domain, and IP
* Port configuration for services
* Node discovery and routing

Schema:
```javascript
({
  name: { type: 'string', unique: true },
  domain: { type: 'string', unique: true },
  ip: { type: 'ip', unique: true },
  ports: { array: 'number' },
});
```

### Peer

Functional requirements:
* Track active peer connections per author
* Monitor last seen timestamp
* Support multiple peers per author (web, mobile, desktop, terminal)
* Connection state management

Schema:
```javascript
({
  owner: 'Author',
  name: 'string',
  lastSeen: 'datetime',
});
```

### Folder

Functional requirements:
* Author can create folders and organize:
  * Chats
  * Feeds
  * Other folders (nested structure)
* Folder visibility: public or unlisted
* Hierarchical folder structure with parent folders

Schema:
```javascript
({
  name: 'string',
  owner: 'Author',
  parent: '?Folder',
  icon: '?string',
  description: '?string',
  visibility: { enum: ['public', 'unlisted'], default: 'unlisted' },
});
```

### File

Functional requirements:
* Upload/download files: streaming over http(s) and ws
* Attach files to messages (including replies) and posts
* File metadata: name, description, icon, mime type, size, checksum (SHA256)
* Preview support: images/audio/video basic metadata (optional implementation)
* File permissions:
  * `unlisted`: only accessible via direct link
  * `public`: discoverable and link accessible
* File versioning via modified timestamp

Schema:
```javascript
({
  name: 'string',
  owner: 'Author',
  icon: '?string',
  description: '?string',
  mime: 'string',
  size: 'number',
  checksum: { type: 'string', length: 64, comment: 'sha256' },
  created: 'datetime',
  modified: 'datetime',
  visibility: { enum: ['public', 'unlisted'], default: 'unlisted' },
});
```

### Feed

Functional requirements:
* Create and manage feeds (channels)
* Feed visibility: public, private, or unlisted
* Join policy: open, invite, or request
* Feed moderation: control who can publish
* Pin messages to feed
* Subscription (follow) feeds
* Notifications policy per feed
* Cross-post or forward posts to other feeds/chats (as link)

Schema:
```javascript
({
  name: { type: 'string', unique: true },
  owner: 'Author',
  icon: '?string',
  description: '?string',
  created: 'datetime',
  visibility: { enum: ['public', 'private', 'unlisted'], default: 'unlisted' },
  joinPolicy: { enum: ['open', 'invite', 'request'], default: 'open' },
  pinned: { many: 'Message' },
});

```

### Chat

Functional requirements:
* Direct (1:1) and group chats are the same
* Add/remove members
* Chat management: name, icon, description
* Track last activity
* Pin messages to chat
* Roles: owner/admin/moderator/member
* Moderation actions: delete content, mute author, ban
* Audit log for moderation actions

Schema:
```javascript
({
  name: { type: 'string', unique: true },
  owner: 'Author',
  icon: '?string',
  description: '?string',
  created: 'datetime',
  lastActivity: 'datetime',
  members: { many: 'Author' },
  admins: { many: 'Author' },
  moderators: { many: 'Author' },
  mute: { many: 'Author' },
  ban: { many: 'Author' },
  pinned: { many: 'Message' },
});
```

### Message

Functional requirements:
* Send/edit/delete messages (soft delete)
* Threaded replies
* Mentions: `@author`
* Forward messages into chats/feeds
* Pin/unpin messages
* Reactions on messages (emoji with author tracking)
* Attach files to messages
* Polls in messages (optional implementation)
* Message search: full-text search

Schema:
```javascript
({
  chat: 'Chat',
  author: 'Author',
  content: 'string',
  created: 'datetime',
  edited: '?datetime',
  deleted: '?datetime',
  replyTo: '?Message',
  forwarded: '?Message',
  reactions: { object: { string: { arrey: 'Author' } }, comment: 'emoji' },
  pinned: { type: 'boolean', default: false },
  attachments: { many: 'File' },
});
```

### Post

Functional requirements:
* Create draft post, publish, edit with revision history (recommended)
* Post status: draft, published, archived
* Reactions
* Polls
* Pin/unpin posts
* Attach files to posts
* Repost to other feeds or forward to chats (as link)
* Post search: full-text search

Schema:
```javascript
({
  feed: 'Feed',
  author: 'Author',
  title: 'string',
  subtitle: '?string',
  content: 'string',
  created: 'datetime',
  edited: '?datetime',
  published: '?datetime',
  deleted: '?datetime',
  status: { enum: ['draft', 'published', 'archived'], default: 'draft' },
  reactions: { object: { string: { arrey: 'Author' } }, comment: 'emoji' },
  pinned: { type: 'boolean', default: false },
  attachments: { many: 'File' },
});
```

## Search

Functional requirements:

Search domains:
* Messages, posts, replies, file names + metadata, optionally file text (indexing)

Search filters:
* by container/feed
* author
* date range
* has:files, has:poll, has:link
* type: message/post/reply/file

## Non-functional requirements

* Realtime exchange
* Consistency: event-ordered per container; client handles eventual updates (edits, deletes).
* Offline/poor network: local storage for last N messages and posts.
* Scalability: nodes and peers
* Security: end-to-end encryption optional (phase 2); baseline TLS.
* Access control: need datailed tasks
* Performance targets (MVP):
  * message send ACK < 200ms typical
  * search query < 500ms for typical scopes
  * load last 50 messages < 300ms (cached)

## User Interface

- Style: Terminal-like UI
- Global layout: 3 columns, collapsible
- Left sidebar: Navigation
- Header: app name + peer status indicator
- Sections:
  * Contacts (other authors, pinned on the top)
  * Feeds (pinned on the top)
  * Chats (pinned on the top)
  * Folders (expandable, pinned on the top)
  * Saved searches (optional, pinned on the top)
- Each item shows: icon (emoji), name/title, unread badge, last activity
- Center column: Main timeline + editor
- Top bar:
  * current context: `!chat` / `#feed`
  * description hover/expand
  * quick actions: Search, Pin list, Members, Files
- Content area:
  * message/post list
  * inline previews for forwarded items / file attachments
- Bottom composer:
  * multiline input
  * controls: attach, emoji, poll, send
- Right sidebar
  * search results
  * details/properties of selected block
- Terminal feel:
  * monospace font, tight spacing, clear separators
  * keyboard-first navigation

### Screen: Chat + Messages

Center list item format example:
* `12:41  timur: message text…`
* Reactions aligned: `:+1:3 :fire:1`
* Reply indicator: `↳ 5 replies` (clock to see list)
* Forwarded content: `↪ from #dev (timur, 12:10): text…`
* File attachment: `[file] report.pdf (2.3MB)` + quick open/download
* pin icon in message line
* pinned list opens in right panel, jump-to-item

### Screen: Feed + Posts

Center shows:
* post cards in text style: title (bold), author + date, stats (replies/reactions)
* center becomes "reader mode" (long answer)
* right panel: outline + pinned posts + thread

### Screen: Search

Search input in top bar (`Ctrl+F`).

Results in right panel or replace center:
* grouped by type (messages/posts/files)
* each result shows context line + jump action

Filters line (terminal-like):
* `in:#feed, in:!chat from:@author has:file before:2025-12-01`

## Permissions model

- Minimum viable rights
- Container membership required for:
  * reading container-scoped messages/files
  * posting/replying (subject to role)
- Feed:
  * read depends on visibility
  * publish depends on publisherPolicy
- File:
  * access determined by `visibility` + container membership + ownership

## Future features

- Advanced messaging
  * Message templates
  * Message threads
  * Rich text formatting (Markdown)
  * Code blocks with syntax highlighting
  * End-to-end encryption for sensitive conversations
  * Auto-delete messages after set time
  * Translate messages to preferred language
  * Scheduled messages: schedule messages to be sent at specific times
  * Auto-responses: set up automated responses based on keywords or patterns
- Voice and video communication
  * Voice messages: record and send audio messages in chats and feeds
  * Voice calls: one-on-one and group calls with WebRTC
  * Screen sharing support
  * Call recording (optional, with consent)
- Chat bots and automation
  * Bot framework: create and deploy chat bots in chats
  * Bot commands: custom commands and slash commands
  * Webhooks: receive and send messages via HTTP webhooks
  * Automation scripts: JavaScript custom workflows
- AI integration
  * Conversational AI assistant via chat bots
  * AI-powered search
  * Smart suggestions: AI-suggested replies, mentions, and content
  * AI help with writing messages, posts, and summaries
  * Real-time translation of messages and posts
  * Summarization
  * AI-assisted content moderation
  * AI-powered smart notifications based on chat activity
- Enhanced collaboration
  * Collaborative real-time editing of posts and documents
  * Shared documents: embed and collaborate on documents within chats/feeds
  * Shared whiteboards
  * Co-browsing websites together in real-time
  * Create and assign tasks within chats and feeds
  * Polls and voting: enhanced polling system with real-time results
  * Calendar integration: schedule events and meetings from chats
- Social features
  * Status updates: images, location
  * Mentions and notifications
  * Follow authors for their public content
  * Personalized activity feed across followed authors
- Search and discovery
  * Save frequently used search queries
  * Track and revisit previous searches
  * AI-powered search query suggestions
  * Discover related messages, posts, and authors
  * Discover trending topics and conversations
  * Personalized content recommendations
- Developer features
  * API for external integration
  * Outbound webhooks for events
  * SDK: client libraries
  * Plugin system
  * Vvisual workflow builder for automation
  * Usage analytics and insights API
  * Configurable rate limits and quotas
  * Sandbox environment
- Integration ecosystem
  * Email integration
  * Calendar sync
  * Gaming integration
- Analytics and insights
  * Detailed analytics dashboard
  * Track engagement across messages, posts, and authors
  * Visualize social network and connections
  * Monitor system performance and health
  * Analyze content performance and reach
  * Generate and export analytics reports
