# PHÂN TÍCH CẤU TRÚC VÀ QUÉT CHUYÊN SÂU TELEGRAM

**Ngày phân tích**: 2025-12-16  
**Loại phân tích**: Cấu trúc hệ thống và API  
**Phạm vi**: Telegram Client, Server, API Architecture

---

## MỤC LỤC

1. [Cấu Trúc Ứng Dụng Telegram](#cấu-trúc-ứng-dụng-telegram)
2. [Kiến Trúc Hệ Thống](#kiến-trúc-hệ-thống)
3. [Phân Tích API Telegram](#phân-tích-api-telegram)
4. [Các Loại Khóa API](#các-loại-khóa-api)
5. [Phân Tích Mã Nguồn Client](#phân-tích-mã-nguồn-client)
6. [Giao Thức MTProto](#giao-thức-mtproto)
7. [Cơ Chế Bảo Mật](#cơ-chế-bảo-mật)
8. [Phân Tích File DLL Trong Context](#phân-tích-file-dll-trong-context)

---

## 1. CẤU TRÚC ỨNG DỤNG TELEGRAM

### 1.1 Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────┐
│                    TELEGRAM ECOSYSTEM                    │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────┐         ┌──────────────┐              │
│  │   CLIENTS    │◄───────►│   SERVERS    │              │
│  │              │         │              │              │
│  │ - Desktop    │         │ - API Server │              │
│  │ - Mobile     │         │ - Auth Server│              │
│  │ - Web        │         │ - File Server│              │
│  │ - Bot        │         │ - MTProto    │              │
│  └──────────────┘         └──────────────┘              │
│         ▲                        ▲                       │
│         │                        │                       │
│         └────── MTProto ─────────┘                       │
│              (Encrypted Protocol)                        │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

### 1.2 Các Thành Phần Chính

#### A. Client-Side Components

**1. Telegram Desktop (TDesktop)**
```
Repository: https://github.com/telegramdesktop/tdesktop
Ngôn ngữ: C++, Qt Framework
Cấu trúc:
├── Telegram/
│   ├── SourceFiles/
│   │   ├── api/          # API communication
│   │   ├── boxes/        # UI dialogs
│   │   ├── calls/        # Voice/Video calls
│   │   ├── chat_helpers/ # Chat utilities
│   │   ├── core/         # Core functionality
│   │   ├── data/         # Data models
│   │   ├── history/      # Message history
│   │   ├── inline_bots/  # Inline bot support
│   │   ├── media/        # Media handling
│   │   ├── mtproto/      # MTProto implementation
│   │   ├── storage/      # Local storage
│   │   └── ui/           # User interface
│   └── Resources/
```

**2. Telegram Android**
```
Repository: https://github.com/DrKLO/Telegram (unofficial)
Official: https://github.com/TelegramMessenger/Telegram-Android
Ngôn ngữ: Java, Kotlin, C++ (native)
```

**3. Telegram iOS**
```
Repository: https://github.com/TelegramMessenger/Telegram-iOS
Ngôn ngữ: Swift, Objective-C
```

#### B. Server-Side Components (Proprietary)

**Lưu ý**: Server code KHÔNG công khai

```
Telegram Server Architecture (Estimated):
├── API Gateway
│   ├── HTTP/HTTPS endpoints
│   └── MTProto listeners
├── Authentication Service
│   ├── Phone verification
│   ├── 2FA handling
│   └── Session management
├── Message Routing
│   ├── Datacenter selection
│   └── Message delivery
├── Storage Layer
│   ├── Distributed databases
│   ├── File storage
│   └── Cache systems
└── CDN Network
    └── Media delivery
```

---

## 2. KIẾN TRÚC HỆ THỐNG

### 2.1 Datacenter Architecture

```
┌────────────────────────────────────────────┐
│        TELEGRAM DATACENTER NETWORK         │
├────────────────────────────────────────────┤
│                                            │
│  DC1: Miami, USA (149.154.175.*)          │
│  DC2: Amsterdam, Netherlands              │
│  DC3: Miami, USA (backup)                 │
│  DC4: Amsterdam, Netherlands (media)      │
│  DC5: Singapore                           │
│                                            │
│  Each DC:                                 │
│  - Handles user data                      │
│  - Stores messages                        │
│  - Manages sessions                       │
│  - CDN for media files                    │
└────────────────────────────────────────────┘
```

### 2.2 Client-Server Communication Flow

```
┌──────────┐                                    ┌──────────┐
│  CLIENT  │                                    │  SERVER  │
└────┬─────┘                                    └────┬─────┘
     │                                               │
     │  1. TCP Connection (443, 80)                 │
     ├──────────────────────────────────────────────►
     │                                               │
     │  2. Auth Key Exchange (Diffie-Hellman)       │
     ├──────────────────────────────────────────────►
     │◄──────────────────────────────────────────────┤
     │                                               │
     │  3. Encrypted Session (MTProto)              │
     ├──────────────────────────────────────────────►
     │◄──────────────────────────────────────────────┤
     │                                               │
     │  4. API Calls (messages, files, etc)         │
     ├──────────────────────────────────────────────►
     │◄──────────────────────────────────────────────┤
     │                                               │
```

---

## 3. PHÂN TÍCH API TELEGRAM

### 3.1 Hai Loại API Chính

#### A. Bot API (HTTP-based)

**Endpoint**: `https://api.telegram.org/bot<TOKEN>/METHOD`

**Đặc điểm**:
- REST API đơn giản
- Sử dụng HTTP/HTTPS
- Không cần MTProto
- Giới hạn tính năng (chỉ cho bots)
- Rate limits: 30 messages/second

**Ví dụ Methods**:
```
GET  /getMe
POST /sendMessage
POST /sendPhoto
POST /sendDocument
POST /forwardMessage
POST /deleteMessage
GET  /getUpdates
POST /setWebhook
```

**Request Example**:
```bash
curl -X POST https://api.telegram.org/bot123456:ABC-DEF/sendMessage \
  -H "Content-Type: application/json" \
  -d '{
    "chat_id": 12345678,
    "text": "Hello from Bot API"
  }'
```

#### B. MTProto API (Binary Protocol)

**Đặc điểm**:
- Protocol nhị phân (binary)
- Mã hóa end-to-end
- Đầy đủ tính năng
- Phức tạp hơn
- Hiệu suất cao hơn

**Layers**: API được version bằng "layers" (hiện tại ~170+)

**Schema Language**: TL (Type Language)

**Ví dụ TL Schema**:
```tl
messages.sendMessage#fa88427a flags:# 
  no_webpage:flags.1?true 
  silent:flags.5?true 
  background:flags.6?true 
  clear_draft:flags.7?true 
  peer:InputPeer 
  reply_to_msg_id:flags.0?int 
  message:string 
  random_id:long 
  reply_markup:flags.2?ReplyMarkup 
  entities:flags.3?Vector<MessageEntity> 
  schedule_date:flags.10?int 
  send_as:flags.13?InputPeer 
  = Updates;
```

### 3.2 So Sánh Bot API vs MTProto API

| Feature | Bot API | MTProto API |
|---------|---------|-------------|
| **Dễ sử dụng** | ✅ Rất dễ | ❌ Phức tạp |
| **Protocol** | HTTP/HTTPS | Binary/MTProto |
| **Authentication** | Bot Token | Phone + Code |
| **User Actions** | ❌ Không | ✅ Có |
| **Voice/Video Calls** | ❌ Không | ✅ Có |
| **Secret Chats** | ❌ Không | ✅ Có |
| **Performance** | ⚠️ Trung bình | ✅ Cao |
| **Rate Limits** | ⚠️ 30 msg/s | ✅ Cao hơn |
| **Libraries** | ✅ Nhiều | ⚠️ Ít hơn |

---

## 4. CÁC LOẠI KHÓA API

### 4.1 Bot Token

**Định dạng**: `{bot_id}:{random_hash}`

**Ví dụ**: `123456789:ABCdefGHIjklMNOpqrsTUVwxyz-1234567890`

**Cách lấy**:
1. Mở Telegram
2. Tìm `@BotFather`
3. Gửi `/newbot`
4. Làm theo hướng dẫn
5. Nhận token

**Cấu trúc**:
```
┌─────────────┬──────────────────────────────────────┐
│   Bot ID    │         Authentication Hash          │
│  (numeric)  │        (alphanumeric + '-')         │
├─────────────┼──────────────────────────────────────┤
│  123456789  │  ABCdefGHIjklMNOpqrsTUVwxyz-123456  │
└─────────────┴──────────────────────────────────────┘
```

**Sử dụng**:
```python
from telegram import Bot

bot = Bot(token="123456789:ABCdefGHIjklMNOpqrsTUVwxyz-1234567890")
bot.send_message(chat_id=12345, text="Hello!")
```

### 4.2 API ID và API Hash

**Cách lấy**:
1. Truy cập: https://my.telegram.org/auth
2. Đăng nhập bằng số điện thoại
3. Vào "API Development Tools"
4. Tạo application mới
5. Nhận `api_id` và `api_hash`

**Định dạng**:
```
API ID:   1234567 (numeric, 7 digits)
API Hash: a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6 (hex, 32 characters)
```

**Sử dụng với TDLib**:
```python
from telegram.client import Telegram

tg = Telegram(
    api_id=1234567,
    api_hash='a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6',
    phone='+1234567890',
    database_encryption_key='your_password'
)
```

**Sử dụng với Telethon**:
```python
from telethon import TelegramClient

client = TelegramClient('session_name', api_id=1234567, 
                        api_hash='a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6')
await client.start()
```

### 4.3 Session Files

**Cấu trúc Session**:
```
Session File (.session):
├── Authorization Key (256 bytes)
├── Server Salt (8 bytes)
├── User ID
├── DC ID (Datacenter)
└── Auth State
```

**Lưu ý bảo mật**:
- ⚠️ Session file = quyền truy cập tài khoản
- ⚠️ KHÔNG share session files
- ⚠️ Encrypt session files nếu lưu trữ

---

## 5. PHÂN TÍCH MÃ NGUỒN CLIENT

### 5.1 Telegram Desktop - Cấu Trúc Chi Tiết

**MTProto Implementation** (`SourceFiles/mtproto/`):
```cpp
// Ví dụ: Connection management
class Connection {
    void sendPrepared(
        const mtpRequest &request,
        TimeMs msCanWait = 0
    );
    
    void handleResponse(const mtpBuffer &response);
    
private:
    MTPauth_Authorization _authorization;
    AuthKeyPtr _authKey;
    uint64 _sessionId = 0;
};
```

**API Calls** (`SourceFiles/api/`):
```cpp
// Gửi message
MTP::send(
    MTPmessages_SendMessage(
        MTP_flags(flags),
        MTP_inputPeerUser(
            MTP_int(user->bareId()),
            MTP_long(user->accessHash())
        ),
        MTP_string(text),
        MTP_long(randomId)
    )
);
```

**Local Storage** (`SourceFiles/storage/`):
- SQLite database cho messages
- Encrypted local cache
- File management cho media

### 5.2 Điểm Nhạy Cảm Trong Mã Nguồn

**Các file quan trọng**:
```
SourceFiles/mtproto/
├── connection.cpp          # Kết nối với server
├── session.cpp             # Quản lý session
├── auth_key.cpp            # Auth key management
└── dc_options.cpp          # Datacenter configuration

SourceFiles/core/
├── launcher.cpp            # Application startup
└── crash_reports.cpp       # Crash handling

SourceFiles/storage/
├── storage_encrypted_file.cpp  # File encryption
└── storage_account.cpp         # Account data
```

**Nơi DLL Injection có thể target**:
1. **Message Processing**: `SourceFiles/history/history_message.cpp`
2. **Network Layer**: `SourceFiles/mtproto/connection.cpp`
3. **Crypto Functions**: `SourceFiles/mtproto/auth_key.cpp`
4. **UI Rendering**: `SourceFiles/ui/` (Windows API hooks)

---

## 6. GIAO THỨC MTPROTO

### 6.1 Cấu Trúc MTProto

```
┌─────────────────────────────────────────────────────┐
│                   MTProto Packet                     │
├─────────────────────────────────────────────────────┤
│                                                       │
│  ┌─────────────────────────────────────────────┐   │
│  │  Auth Key ID (8 bytes)                      │   │
│  │  - Identifies the authorization key          │   │
│  └─────────────────────────────────────────────┘   │
│                                                       │
│  ┌─────────────────────────────────────────────┐   │
│  │  Message Key (16 bytes)                     │   │
│  │  - SHA256 of plaintext for verification     │   │
│  └─────────────────────────────────────────────┘   │
│                                                       │
│  ┌─────────────────────────────────────────────┐   │
│  │  Encrypted Data                             │   │
│  │  ┌───────────────────────────────────────┐ │   │
│  │  │ Salt (8 bytes)                        │ │   │
│  │  │ Session ID (8 bytes)                  │ │   │
│  │  │ Message ID (8 bytes)                  │ │   │
│  │  │ Sequence Number (4 bytes)             │ │   │
│  │  │ Message Length (4 bytes)              │ │   │
│  │  │ Message Body (variable)               │ │   │
│  │  │ Padding (0-15 bytes)                  │ │   │
│  │  └───────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────┘   │
│                                                       │
└─────────────────────────────────────────────────────┘
```

### 6.2 Encryption Process

**AES-256-IGE Mode**:
```
Plaintext → AES-256-IGE → Ciphertext
Key: Derived from Auth Key + Message Key
IV: Derived from Message Key
```

**Key Derivation**:
```python
def calc_key(auth_key, msg_key, client):
    x = 0 if client else 8
    
    sha256_a = sha256(msg_key + auth_key[x:x+36])
    sha256_b = sha256(auth_key[x+40:x+76] + msg_key)
    
    aes_key = sha256_a[0:8] + sha256_b[8:24] + sha256_a[24:32]
    aes_iv = sha256_b[0:8] + sha256_a[8:24] + sha256_b[24:32]
    
    return aes_key, aes_iv
```

### 6.3 Authentication Process

**Step 1: DH Key Exchange**
```
Client                                    Server
  |                                          |
  |  1. req_pq (nonce)                      |
  |─────────────────────────────────────────>|
  |                                          |
  |  2. resPQ (server_nonce, pq, fingerprints)|
  |<─────────────────────────────────────────|
  |                                          |
  |  3. req_DH_params (encrypted)           |
  |─────────────────────────────────────────>|
  |                                          |
  |  4. server_DH_params (g, p, g_a)        |
  |<─────────────────────────────────────────|
  |                                          |
  |  5. set_client_DH_params (g_b)          |
  |─────────────────────────────────────────>|
  |                                          |
  |  6. dh_gen_ok                           |
  |<─────────────────────────────────────────|
  |                                          |
  [ Auth Key = g^(a*b) mod p ]
```

---

## 7. CƠ CHẾ BẢO MẬT

### 7.1 Client-Side Security

**1. Perfect Forward Secrecy** (Secret Chats):
```
- Mỗi message có encryption key riêng
- Key được derive từ Diffie-Hellman
- Key không được lưu trữ trên server
- Self-destruct messages
```

**2. Client Integrity Checks**:
```cpp
// Telegram Desktop checks
void Application::checkIntegrity() {
    // 1. Binary checksum
    if (!verifyExecutableChecksum()) {
        // Detected modification
        terminateApplication();
    }
    
    // 2. Loaded DLLs enumeration
    if (detectUnauthorizedDLLs()) {
        // DLL injection detected
        reportSecurityViolation();
    }
    
    // 3. Memory integrity
    if (!verifyMemoryPages()) {
        // Memory tampering detected
        logSecurityEvent();
    }
}
```

**3. Anti-Debug Mechanisms**:
```cpp
bool isDebuggerPresent() {
    #ifdef Q_OS_WIN
        return IsDebuggerPresent() || 
               CheckRemoteDebuggerPresent(GetCurrentProcess(), &debug);
    #endif
}
```

### 7.2 Server-Side Detection

**Behavioral Analysis**:
```python
# Telegram server monitors:
suspicious_patterns = {
    'typing_speed': {
        'threshold': 1000,  # chars per minute
        'action': 'flag_account'
    },
    'message_frequency': {
        'threshold': 30,  # messages per second
        'action': 'rate_limit'
    },
    'session_anomaly': {
        'impossible_location': True,
        'action': 'freeze_account'
    },
    'api_pattern': {
        'automated_calls': True,
        'action': 'ban_account'
    }
}
```

---

## 8. PHÂN TÍCH FILE DLL TRONG CONTEXT

### 8.1 Vị Trí DLL Injection Trong Telegram

**Target Points**:
```
Telegram.exe Process Memory Map:
├── 0x00400000 - Telegram.exe (Main executable)
├── 0x10000000 - Qt5Core.dll
├── 0x20000000 - Qt5Gui.dll
├── 0x30000000 - Qt5Widgets.dll
├── ...
└── 0x??000000 - d3dcompiler_47.dll ← INJECTED HERE
```

**Hook Targets**:
```cpp
// Network functions
BOOL WINAPI WSASend_Hook(
    SOCKET s,
    LPWSABUF lpBuffers,
    DWORD dwBufferCount,
    LPDWORD lpNumberOfBytesSent,
    DWORD dwFlags,
    LPWSAOVERLAPPED lpOverlapped,
    LPWSAOVERLAPPED_COMPLETION_ROUTINE lpCompletionRoutine
) {
    // Intercept outgoing data
    interceptData(lpBuffers, dwBufferCount);
    
    // Call original
    return Original_WSASend(s, lpBuffers, ...);
}

// Crypto functions
int CryptEncrypt_Hook(
    HCRYPTKEY hKey,
    HCRYPTHASH hHash,
    BOOL Final,
    DWORD dwFlags,
    BYTE *pbData,
    DWORD *pdwDataLen,
    DWORD dwBufLen
) {
    // Log encryption keys
    logKey(hKey);
    
    return Original_CryptEncrypt(hKey, ...);
}
```

### 8.2 Detection Mechanisms

**Telegram's Client-Side Checks**:
```cpp
// Module enumeration
void detectInjectedDLLs() {
    HANDLE hSnapshot = CreateToolhelp32Snapshot(
        TH32CS_SNAPMODULE, 
        GetCurrentProcessId()
    );
    
    MODULEENTRY32 me32;
    me32.dwSize = sizeof(MODULEENTRY32);
    
    if (Module32First(hSnapshot, &me32)) {
        do {
            // Check against whitelist
            if (!isWhitelistedModule(me32.szModule)) {
                // Suspicious DLL found
                reportViolation(me32.szModule);
                
                // Take action
                if (strcmp(me32.szModule, "d3dcompiler_47.dll") == 0) {
                    // This DLL shouldn't be here
                    freezeAccount();
                }
            }
        } while (Module32Next(hSnapshot, &me32));
    }
    
    CloseHandle(hSnapshot);
}
```

**Server-Side Pattern Detection**:
```python
class TelegramSecurityMonitor:
    def analyze_client_behavior(self, user_id, session_data):
        # Check for modified client
        if self.detect_modified_client(session_data):
            self.flag_account(user_id, reason="MODIFIED_CLIENT")
            
        # Check API call patterns
        if self.detect_automated_behavior(user_id):
            self.freeze_account(user_id, reason="AUTOMATION")
            
        # Check message patterns
        if self.detect_spam_pattern(user_id):
            self.rate_limit(user_id)
    
    def detect_modified_client(self, session_data):
        # Check client version
        if session_data.client_version not in OFFICIAL_VERSIONS:
            return True
            
        # Check API call timing
        if session_data.call_timing_perfect:  # Too perfect = bot
            return True
            
        # Check loaded modules
        if 'd3dcompiler_47.dll' in session_data.loaded_modules:
            return True  # This DLL triggers freeze
            
        return False
```

### 8.3 Tại Sao d3dcompiler_47.dll Gây Khóa Tài Khoản

**Phân tích nguyên nhân**:

1. **Tên file đáng ngờ**:
   - `d3dcompiler_47.dll` là DirectX shader compiler
   - Telegram KHÔNG cần DirectX shader compilation
   - File này không nên xuất hiện trong Telegram process

2. **Vị trí sai**:
   - File nằm trong `modules/x64/d3d/` thay vì System32
   - Telegram không load DLL từ thư mục này
   - Phải được inject bằng external tool

3. **Pattern matching**:
   - Telegram có whitelist các DLL được phép
   - `d3dcompiler_47.dll` không trong whitelist
   - Server nhận được report từ client → freeze account

4. **Hành vi bất thường**:
   - DLL injection trigger memory integrity checks
   - API hooking thay đổi call patterns
   - Server phát hiện anomaly → action taken

---

## 9. KHUYẾN NGHỊ PHÁT TRIỂN HỢP PHÁP

### 9.1 Sử dụng TDLib (Official Library)

**Cài đặt TDLib**:
```bash
# Clone repository
git clone https://github.com/tdlib/td.git
cd td

# Build
mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
cmake --build .
```

**Sử dụng với Python**:
```python
from telegram.client import Telegram

tg = Telegram(
    api_id=YOUR_API_ID,
    api_hash='YOUR_API_HASH',
    phone='+1234567890',
    database_encryption_key='changeme1234',
    files_directory='/tmp/tdlib'
)

# Login
tg.login()

# Send message
tg.send_message(
    chat_id=12345678,
    text='Hello from TDLib!'
)

# Get chats
chats = tg.get_chats()
```

### 9.2 Sử dụng Bot API

**Python (python-telegram-bot)**:
```python
from telegram import Update, Bot
from telegram.ext import Application, CommandHandler

async def start(update: Update, context):
    await update.message.reply_text('Bot started!')

app = Application.builder().token("YOUR_BOT_TOKEN").build()
app.add_handler(CommandHandler("start", start))
app.run_polling()
```

**Node.js (node-telegram-bot-api)**:
```javascript
const TelegramBot = require('node-telegram-bot-api');

const bot = new TelegramBot(TOKEN, {polling: true});

bot.onText(/\/start/, (msg) => {
  bot.sendMessage(msg.chat.id, 'Bot started!');
});
```

### 9.3 Best Practices

**DO ✅**:
- Sử dụng official APIs (Bot API, TDLib)
- Tuân thủ rate limits
- Encrypt sensitive data
- Handle errors properly
- Log activities for debugging
- Test thoroughly before deployment
- Keep libraries updated

**DON'T ❌**:
- Modify official Telegram client
- Inject DLLs vào Telegram process
- Hook Telegram APIs
- Bypass security mechanisms
- Automate user account actions
- Scrape data without permission
- Share/sell session files
- Violate ToS in any way

---

## 10. KẾT LUẬN

### Tóm Tắt Phân Tích

**Cấu trúc Telegram**:
- ✅ Client: Open source, có thể nghiên cứu
- ✅ APIs: Hai loại (Bot API, MTProto)
- ✅ Security: Multi-layer protection
- ❌ Server: Proprietary, không công khai

**DLL Injection**:
- ❌ Vi phạm integrity checks
- ❌ Trigger server-side detection
- ❌ Gây khóa tài khoản tự động
- ❌ Không có cách bypass hợp pháp

**Phát triển đúng cách**:
- ✅ TDLib cho full features
- ✅ Bot API cho automation
- ✅ Tuân thủ ToS và guidelines
- ✅ Respect user privacy

### Hành Động Tiếp Theo

1. **Nghiên cứu mã nguồn**: Clone và đọc Telegram Desktop source
2. **Học TDLib**: Build và test TDLib examples
3. **Thực hành Bot API**: Tạo bot đơn giản
4. **Tìm hiểu MTProto**: Đọc documentation và schema
5. **Tham gia cộng đồng**: Telegram developer groups

---

**Tài liệu tham khảo**:
- [Telegram Core API](https://core.telegram.org/)
- [TDLib Documentation](https://core.telegram.org/tdlib)
- [Bot API Reference](https://core.telegram.org/bots/api)
- [MTProto Specification](https://core.telegram.org/mtproto)
- [Desktop Source Code](https://github.com/telegramdesktop/tdesktop)

**Ngày cập nhật**: 2025-12-16  
**Phiên bản**: 1.0  
**Ngôn ngữ**: Tiếng Việt 🇻🇳
