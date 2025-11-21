# ZMA (Zalo Mini App) API Documentation

## Tổng quan

ZMA API cung cấp các endpoint cho ứng dụng Zalo Mini App để tương tác với hệ thống. API này tuân theo chuẩn RESTful và trả về JSON response với format chung.

## Response Format

Hầu hết các API responses đều tuân theo format chung:

```json
{
    "success": true|false,
    "data": {}, // hoặc []
    "message": "Success message",
    "errors": {}, // hoặc []
    "pagination": {} // nếu có phân trang
}
```

## Authentication

Các endpoint yêu cầu authentication, cần đăng nhập qua `/api/customer-auth/login` để lấy token, sau đó sử dụng Bearer Token trong header:

```
Authorization: Bearer <token>
```

## Endpoints Summary

| Method | Endpoint | Mô tả | Pagination | Filters/Params | Authentication |
|--------|----------|-------|------------|----------------|----------------|
| POST | `/api/customer-auth/login` | Đăng nhập với Zalo access_token và phone_token | - | body: `access_token`, `phone_token`, `campaign_id` (optional) | Public |
| GET | `/api/customer-auth/me` | Lấy thông tin customer hiện tại | - | - | Bearer Token |
| POST | `/api/customer-auth/logout` | Đăng xuất (revoke token hiện tại) | - | - | Bearer Token |
| POST | `/api/customer-auth/logout-all` | Đăng xuất tất cả devices | - | - | Bearer Token |
| POST | `/api/qr/submit` | Submit QR code (RSA encrypted) | - | body: `payload` | Bearer Token |
| GET | `/api/qr/available` | Kiểm tra xem có thể submit QR code không | - | - | Bearer Token |
| GET | `/api/wards` | Danh sách phường/xã | `ward_page`, `ward_per_page` | `ward_province_code`, `ward_keyword`, `ward_code`, `ward_name` | Bearer Token |
| GET | `/api/provinces` | Danh sách tỉnh/thành phố | `province_page`, `province_per_page` | `province_keyword`, `province_code`, `province_name` | Bearer Token |
| POST | `/api/customers` | Tạo khách hàng | - | body: `name`, `phone`, `identity_id`, `channel`, ... | Bearer Token |
| GET | `/api/customers/show` | Lấy KH theo `identity_id` | - | query: `identity_id` | Bearer Token |
| PUT | `/api/customers/{id}` | Cập nhật khách hàng | - | body: các trường thông tin khách hàng | Bearer Token |
| GET | `/api/campaigns/{id}/prizes` | Danh sách giải thưởng của campaign | `prize_page`, `prize_per_page` | - | Bearer Token |
| GET | `/api/campaigns/{id}` | Chi tiết campaign | - | - | Bearer Token |
| GET | `/api/campaigns/{campaignId}/customer/{customerId}/winners` | Lịch sử trúng giải của KH trong campaign | `page`, `per_page` | - | Bearer Token |
| GET | `/api/campaigns/{campaignId}/customer/{customerId}/winner-histories` | Lịch sử trúng giải (tất cả loại phần thưởng) | `page`, `per_page` | - | Bearer Token |

## Endpoints

### 1. Customer Authentication

Customer Authentication API cung cấp các endpoint để xác thực khách hàng thông qua Zalo Mini App. API này sử dụng API token để xác thực và phân biệt với user thông thường.

#### 1.1. Đăng nhập với Zalo Access Token

**POST** `/api/customer-auth/login`

Đăng nhập khách hàng bằng cách xác thực `access_token` từ Zalo Mini App. Backend sẽ gọi Zalo Graph API để verify token và lấy thông tin user. Nếu customer đã tồn tại (theo `identity_id`), hệ thống sẽ lấy customer đó. Nếu chưa tồn tại, hệ thống sẽ tạo mới customer rồi trả về kèm token.

##### Authentication
- **Required**: None (Public)

##### Request Body

```json
{
    "access_token": "zalo_access_token_from_mini_app",
    "phone_token": "phone_token_from_getPhoneNumber",
    "campaign_id": 1
}
```

**Lưu ý**: `campaign_id` là tùy chọn. Chỉ cần truyền khi mỗi chiến dịch sử dụng Zalo App riêng biệt.

##### Request Body Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `access_token` | string | Yes | Access token nhận được từ Zalo Mini App sau khi user authorize |
| `phone_token` | string | Yes | Token nhận được từ `getPhoneNumber()` API của Zalo SDK. Hệ thống sẽ tự động lấy số điện thoại và cập nhật vào customer |
| `campaign_id` | integer | No | ID của chiến dịch. **Chỉ cần truyền khi mỗi chiến dịch sử dụng Zalo App riêng biệt**. Nếu tất cả chiến dịch dùng chung một Zalo App, không cần truyền tham số này. Khi có `campaign_id`, hệ thống sẽ sử dụng cấu hình Zalo (secret_key) của chiến dịch đó để verify access token |

##### Response Example (200 - Success)

```json
{
    "success": true,
    "message": "Đăng nhập thành công",
    "data": {
        "customer": {
            "id": 1,
            "name": "Nguyễn Văn A",
            "email": null,
            "phone": "123456789012",
            "address": null,
            "buy_address": null,
            "identity_id": "123456789012",
            "channel": "zalo",
            "province_code": "",
            "ward_code": "",
            "created_at": "2024-06-15T10:30:00.000000Z",
            "updated_at": "2024-06-15T10:30:00.000000Z"
        },
        "token": "1|abcdef1234567890..."
    }
}
```

##### Error Responses

**400 - Access token không hợp lệ:**
```json
{
    "success": false,
    "message": "Access token không hợp lệ hoặc đã hết hạn.",
    "data": []
}
```

**400 - Không thể lấy identity từ Zalo:**
```json
{
    "success": false,
    "message": "Không thể lấy thông tin identity từ Zalo.",
    "data": []
}
```

**400 - Không thể lấy số điện thoại:**
```json
{
    "success": false,
    "message": "Không thể lấy số điện thoại từ phone_token. Vui lòng thử lại.",
    "data": []
}
```

**422 - Validation Error:**
```json
{
    "success": false,
    "message": "Dữ liệu không hợp lệ",
    "data": {
        "access_token": ["The access token field is required."],
        "phone_token": ["The phone token field is required."]
    }
}
```

**500 - Server Error:**
```json
{
    "success": false,
    "message": "Có lỗi xảy ra khi đăng nhập: <error_message>",
    "data": []
}
```

##### cURL Example

**Không có campaign_id (dùng chung Zalo App):**
```bash
curl -X POST http://localhost:8000/api/customer-auth/login \
  -H "Content-Type: application/json" \
  -d '{"access_token":"zalo_access_token_here","phone_token":"phone_token_from_getPhoneNumber"}'
```

**Có campaign_id (mỗi chiến dịch dùng Zalo App riêng):**
```bash
curl -X POST http://localhost:8000/api/customer-auth/login \
  -H "Content-Type: application/json" \
  -d '{"access_token":"zalo_access_token_here","phone_token":"phone_token_from_getPhoneNumber","campaign_id":1}'
```

##### JavaScript Example

**Không có campaign_id (dùng chung Zalo App):**
```javascript
// Lấy access_token từ Zalo
const accessToken = await getAccessToken();

// Lấy phone_token từ getPhoneNumber() (bắt buộc)
const phoneResult = await getPhoneNumber();
const phoneToken = phoneResult.token;

// Gửi login request
const response = await fetch('/api/customer-auth/login', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
    },
    body: JSON.stringify({
        access_token: accessToken,
        phone_token: phoneToken
    })
});

const data = await response.json();
if (data.success) {
    const { customer, token } = data.data;
    // Lưu token để sử dụng cho các request tiếp theo
    localStorage.setItem('customer_token', token);
} else {
    // Xử lý lỗi
    console.error('Login failed:', data.message);
}
```

**Có campaign_id (mỗi chiến dịch dùng Zalo App riêng):**
```javascript
// Lấy access_token từ Zalo
const accessToken = await getAccessToken();

// Lấy phone_token từ getPhoneNumber() (bắt buộc)
const phoneResult = await getPhoneNumber();
const phoneToken = phoneResult.token;

// Lấy campaign_id từ context hoặc config
const campaignId = 1; // ID của chiến dịch hiện tại

// Gửi login request
const response = await fetch('/api/customer-auth/login', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
    },
    body: JSON.stringify({
        access_token: accessToken,
        phone_token: phoneToken,
        campaign_id: campaignId
    })
});

const data = await response.json();
if (data.success) {
    const { customer, token } = data.data;
    // Lưu token để sử dụng cho các request tiếp theo
    localStorage.setItem('customer_token', token);
} else {
    // Xử lý lỗi
    console.error('Login failed:', data.message);
}
```

##### Notes

- Access token từ Zalo có thời gian hết hạn. Client cần refresh token khi cần thiết.
- `phone_token` là bắt buộc. Hệ thống sẽ tự động lấy số điện thoại từ Zalo và:
  - Lưu vào customer mới nếu đang tạo mới
  - Cập nhật vào customer hiện có nếu đã tồn tại
- Nếu customer đã tồn tại với `identity_id` tương ứng, hệ thống sẽ không tạo mới mà lấy customer hiện có và cập nhật số điện thoại.
- Token trả về là API token, có thể sử dụng cho các API protected.
- **`campaign_id` (tùy chọn)**: 
  - **Khi nào cần truyền**: Nếu mỗi chiến dịch sử dụng một Zalo App riêng biệt (có secret_key riêng), cần truyền `campaign_id` để hệ thống xác định đúng cấu hình Zalo của chiến dịch đó để verify access token.
  - **Khi nào không cần**: Nếu tất cả các chiến dịch dùng chung một Zalo App (cùng secret_key), không cần truyền `campaign_id`. Hệ thống sẽ sử dụng cấu hình Zalo chung từ config.
  - Khi có `campaign_id`, hệ thống sẽ lấy `secret_key` từ cấu hình của chiến dịch đó thay vì dùng `secret_key` chung từ config.

---

#### 1.2. Lấy thông tin khách hàng hiện tại

**GET** `/api/customer-auth/me`

Lấy thông tin của khách hàng đang đăng nhập.

##### Authentication
- **Required**: Bearer Token

##### Headers

```
Authorization: Bearer <token>
```

##### Response Example (200 - Success)

```json
{
    "success": true,
    "message": "Thông tin khách hàng hiện tại",
    "data": {
        "id": 1,
        "name": "Nguyễn Văn A",
        "email": null,
        "phone": "123456789012",
        "address": null,
        "buy_address": null,
        "identity_id": "123456789012",
        "channel": "zalo",
        "province_code": "",
        "ward_code": "",
        "created_at": "2024-06-15T10:30:00.000000Z",
        "updated_at": "2024-06-15T10:30:00.000000Z"
    }
}
```

##### Error Responses

**401 - Unauthenticated:**
```json
{
    "success": false,
    "message": "Unauthenticated",
    "data": []
}
```

##### cURL Example

```bash
curl http://localhost:8000/api/customer-auth/me \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

##### JavaScript Example

```javascript
const token = localStorage.getItem('customer_token');
const response = await fetch('/api/customer-auth/me', {
    method: 'GET',
    headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
    }
});

const data = await response.json();
if (data.success) {
    const customer = data.data;
    console.log('Customer info:', customer);
}
```

---

#### 1.3. Đăng xuất (Revoke token hiện tại)

**POST** `/api/customer-auth/logout`

Đăng xuất khách hàng bằng cách revoke token hiện tại. Token này sẽ không thể sử dụng được nữa.

##### Authentication
- **Required**: Bearer Token

##### Headers

```
Authorization: Bearer <token>
```

##### Response Example (200 - Success)

```json
{
    "success": true,
    "message": "Đăng xuất thành công",
    "data": null
}
```

##### Error Responses

**401 - Unauthenticated:**
```json
{
    "success": false,
    "message": "Unauthenticated",
    "data": []
}
```

##### cURL Example

```bash
curl -X POST http://localhost:8000/api/customer-auth/logout \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

##### JavaScript Example

```javascript
const token = localStorage.getItem('customer_token');
const response = await fetch('/api/customer-auth/logout', {
    method: 'POST',
    headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
    }
});

const data = await response.json();
if (data.success) {
    // Xóa token khỏi localStorage
    localStorage.removeItem('customer_token');
}
```

---

#### 1.4. Đăng xuất tất cả devices (Revoke tất cả tokens)

**POST** `/api/customer-auth/logout-all`

Đăng xuất khách hàng khỏi tất cả devices bằng cách revoke tất cả tokens của customer đó.

##### Authentication
- **Required**: Bearer Token

##### Headers

```
Authorization: Bearer <token>
```

##### Response Example (200 - Success)

```json
{
    "success": true,
    "message": "Đã thu hồi tất cả token",
    "data": null
}
```

##### Error Responses

**401 - Unauthenticated:**
```json
{
    "success": false,
    "message": "Unauthenticated",
    "data": []
}
```

##### cURL Example

```bash
curl -X POST http://localhost:8000/api/customer-auth/logout-all \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

##### JavaScript Example

```javascript
const token = localStorage.getItem('customer_token');
const response = await fetch('/api/customer-auth/logout-all', {
    method: 'POST',
    headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
    }
});

const data = await response.json();
if (data.success) {
    // Xóa token khỏi localStorage
    localStorage.removeItem('customer_token');
}
```

---

#### 1.5. Flow Xác thực

##### Client Flow

1. User mở Zalo Mini App và authorize
2. Zalo Mini App trả về `access_token` từ `getAccessToken()`
3. Client gọi `getPhoneNumber()` để lấy `phone_token` (bắt buộc)
4. Client gửi `access_token`, `phone_token` và `campaign_id` (nếu cần) lên backend: `POST /api/customer-auth/login`
   - **Nếu mỗi chiến dịch dùng Zalo App riêng**: Cần truyền `campaign_id` để backend xác định đúng cấu hình Zalo
   - **Nếu tất cả chiến dịch dùng chung Zalo App**: Không cần truyền `campaign_id`
5. Backend verify `access_token` với Zalo Graph API (sử dụng `secret_key` từ campaign config nếu có `campaign_id`, ngược lại dùng `secret_key` chung) và lấy thông tin user
6. Backend lấy số điện thoại từ `phone_token` qua Zalo Open API
7. Backend tìm hoặc tạo customer dựa trên `identity_id`, lưu/cập nhật số điện thoại
8. Backend tạo API token và trả về cho client
9. Client lưu token và sử dụng cho các request tiếp theo

##### Sử dụng Token

Sau khi có token, client gửi token trong header `Authorization` cho các API protected:

```javascript
fetch('/api/some-protected-endpoint', {
    headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
    }
})
```

##### Integration với Zalo Mini App

**Không có campaign_id (dùng chung Zalo App):**
```javascript
import { getAccessToken, getPhoneNumber } from "zmp-sdk/apis";

// Lấy access token từ Zalo
const accessTokenResult = await getAccessToken();
const accessToken = accessTokenResult.access_token;

// Lấy phone token (bắt buộc)
const phoneResult = await getPhoneNumber();
const phoneToken = phoneResult.token;

// Gửi access token và phone token lên backend
const response = await fetch('/api/customer-auth/login', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
    },
    body: JSON.stringify({
        access_token: accessToken,
        phone_token: phoneToken
    })
});

const data = await response.json();
if (data.success) {
    const { customer, token } = data.data;
    // Lưu token để sử dụng
    localStorage.setItem('customer_token', token);
} else {
    // Xử lý lỗi
    console.error('Login failed:', data.message);
}
```

**Có campaign_id (mỗi chiến dịch dùng Zalo App riêng):**
```javascript
import { getAccessToken, getPhoneNumber } from "zmp-sdk/apis";

// Lấy access token từ Zalo
const accessTokenResult = await getAccessToken();
const accessToken = accessTokenResult.access_token;

// Lấy phone token (bắt buộc)
const phoneResult = await getPhoneNumber();
const phoneToken = phoneResult.token;

// Lấy campaign_id từ context hoặc config của app
const campaignId = getCurrentCampaignId(); // Hàm lấy campaign_id hiện tại

// Gửi access token, phone token và campaign_id lên backend
const response = await fetch('/api/customer-auth/login', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
    },
    body: JSON.stringify({
        access_token: accessToken,
        phone_token: phoneToken,
        campaign_id: campaignId
    })
});

const data = await response.json();
if (data.success) {
    const { customer, token } = data.data;
    // Lưu token để sử dụng
    localStorage.setItem('customer_token', token);
} else {
    // Xử lý lỗi
    console.error('Login failed:', data.message);
}
```

##### Notes

- **Token Expiry**: API tokens mặc định không có thời gian hết hạn. Bạn có thể cấu hình expiry trong hệ thống.
- **Multiple Devices**: Customer có thể đăng nhập trên nhiều devices. Mỗi device sẽ có một token riêng.
- **Token Revocation**: Khi logout, chỉ token hiện tại bị revoke. Để logout tất cả devices, sử dụng `/logout-all`.
- **Zalo Token**: Access token từ Zalo có thời gian hết hạn. Client cần refresh token từ Zalo khi cần thiết.
- **Customer Creation**: Nếu customer chưa tồn tại, hệ thống sẽ tự động tạo mới với thông tin từ Zalo. Các trường không có trong Zalo response sẽ được set giá trị mặc định.
- **Campaign ID và Zalo App Configuration**:
  - **Khi mỗi chiến dịch dùng Zalo App riêng**: Mỗi chiến dịch có cấu hình Zalo riêng (app_id, secret_key). Khi đăng nhập, cần truyền `campaign_id` để backend xác định đúng cấu hình để verify access token. Backend sẽ lấy `secret_key` từ cấu hình của chiến dịch đó.
  - **Khi tất cả chiến dịch dùng chung Zalo App**: Tất cả chiến dịch dùng chung một cấu hình Zalo (app_id, secret_key) từ config chung. Không cần truyền `campaign_id`. Backend sẽ sử dụng `secret_key` chung từ config.
  - Việc xác định có cần truyền `campaign_id` hay không phụ thuộc vào cách hệ thống được cấu hình. Nếu không chắc chắn, hãy liên hệ với team backend để xác nhận.
- **Security**: 
  - Luôn sử dụng HTTPS trong production để bảo vệ token.
  - Lưu token ở nơi an toàn (localStorage, secure cookie, hoặc secure storage trên mobile).
  - Backend luôn verify token với Zalo API trước khi tạo API token.
  - `identity_id` từ Zalo là unique và được dùng làm identifier cho customer.

---

### 2. Submit QR Code

**POST** `/api/qr/submit`

Submit QR code đã được mã hóa RSA để xử lý quay thưởng. QR code phải được mã hóa bằng RSA public key trước khi gửi lên.

#### Authentication
- **Required**: Bearer Token

#### Request Body

```json
{
    "payload": "base64_encoded_rsa_encrypted_string"
}
```

#### Example js
```js

 // sha256 with Salt + Qrcode
 async function sha256Hex(text: string): Promise<string> {
    const enc = new TextEncoder().encode(text);
    const digest = await crypto.subtle.digest('SHA-256', enc);
    const bytes = Array.from(new Uint8Array(digest));
    return bytes.map(b => b.toString(16).padStart(2, '0')).join('');
 }

 // rsa with public key pem
 async function rsaOaepEncryptBase64(publicKeyPem: string, data: string): Promise<string> {
    const key = await importRsaPublicKey(publicKeyPem);
    const enc = new TextEncoder().encode(data);
    const cipher = await crypto.subtle.encrypt({ name: 'RSA-OAEP' }, key, enc);
    const bytes = new Uint8Array(cipher);
    let bin = '';
    bytes.forEach(b => { bin += String.fromCharCode(b); });
    return btoa(bin);
 }

 // submit function
  const payloadObj = {
    nonce: `nonce_${Date.now()}_${Math.random().toString(36).slice(2, 10)}`, // make unique string
    ts: Date.now(), // timestamp
    qr: qrText, // original qr
    qr_hash: await sha256Hex(campaignSalt + qrText), // hash
  };
  const json = JSON.stringify(payloadObj);
  const b64 = await rsaOaepEncryptBase64(pubKey, json); // rsa b64
  // example request
  const res = await fetch('/api/qr/submit', {
    method: 'POST',
    headers: { 
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}` // Token từ customer-auth/login
    },
    body: JSON.stringify({ payload: b64 }),
  });

```

#### Request Body Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `payload` | string | Yes | Base64 encoded RSA encrypted string. Sau khi decrypt sẽ là JSON chứa: `nonce`, `ts` (timestamp), `qr`, `qr_hash` |

#### Decrypted Payload Format

Payload sau khi decrypt sẽ có format:

```json
{
    "nonce": "unique_nonce_string",
    "ts": 1234567890,
    "qr": "original_qr_code_string",
    "qr_hash": "hashed_qr_code"
}
```

#### Response Example (Success - 200 OK)

**Trường hợp trúng giải:**

```json
{
    "success": true,
    "message": "Congratulations! You won a prize!",
    "data": {
    "prize": {
        "id": 1,
        "name": "Giải nhất - iPhone 15 Pro"
        }
    }
}
```

**Trường hợp không trúng giải:**

```json
{
    "success": true,
    "message": "QR processed but no prize available!",
    "data": []
}
```

#### Response Example (Error - 422 Validation Failed)

```json
{
    "success": false,
    "message": "Invalid base64 payload",
    "errors": {
        "code": "INVALID_BASE64_PAYLOAD"
    }
}
```

#### Response Example (Error - 422 Missing Fields)

```json
{
    "success": false,
    "message": "Missing fields",
    "errors": {
        "code": "MISSING_FIELDS"
    }
}
```

#### Response Example (Error - 422 QR Not Valid)

```json
{
    "success": false,
    "message": "QR not valid",
    "errors": {
        "code": "QR_UNPROCESSED"
    }
}
```

#### Response Example (Error - 404 Campaign Not Found)

```json
{
    "success": false,
    "message": "Campaign not found",
    "errors": {
        "code": "CAMPAIGN_NOT_FOUND"
    }
}
```

#### Response Example (Error - 409 QR Already Used)

```json
{
    "success": false,
    "message": "QR already used",
    "errors": {
        "code": "QR_ALREADY_USED"
    }
}
```

#### Response Example (Error - 403 Permanent Banned)

```json
{
    "success": false,
    "message": "Account permanently banned due to too many failed attempts",
    "errors": {
        "code": "PERMANENT_BANNED"
    }
}
```

#### Response Example (Error - 429 Daily Limit Exceeded)

```json
{
    "success": false,
    "message": "Daily failed attempts limit reached",
    "errors": {
        "code": "DAILY_LIMIT_EXCEEDED"
    }
}
```

#### Notes
- QR code phải được mã hóa bằng RSA public key trước khi gửi
- Hệ thống sử dụng RSA với PKCS1 OAEP padding để decrypt
- Hệ thống có rate limiting:
  - Tổng số lần thất bại: mặc định 15 lần (config: `limit_total_failed`)
  - Tổng số lần thất bại theo IP: mặc định 100 lần (config: `limit_by_ip_total_failed`)
  - Số lần thất bại mỗi ngày: mặc định 5 lần (config: `limit_each_day_failed`)
- Khi trúng giải, hệ thống sẽ tự động gửi thông báo ZNS nếu có `zns_template_id`
- QR code chỉ có thể sử dụng một lần duy nhất

#### Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `INVALID_BASE64_PAYLOAD` | 422 | Payload không phải base64 hợp lệ |
| `DECRYPT_FAILED` | 422 | Không thể decrypt payload |
| `INVALID_DECRYPTED_JSON` | 422 | JSON sau khi decrypt không hợp lệ |
| `MISSING_FIELDS` | 422 | Thiếu các trường bắt buộc trong payload |
| `QR_UNPROCESSED` | 422 | QR code không hợp lệ hoặc không tồn tại trong hệ thống |
| `CAMPAIGN_NOT_FOUND` | 404 | Chiến dịch không tồn tại hoặc QR không gắn với chiến dịch hợp lệ |
| `QR_ALREADY_USED` | 409 | QR code đã được sử dụng trước đó |
| `PERMANENT_BANNED` | 403 | Tài khoản/IP bị ban vĩnh viễn do quá nhiều lần thất bại (mặc định: 15 lần theo customer_id hoặc 100 lần theo IP, có thể config) |
| `DAILY_LIMIT_EXCEEDED` | 429 | Đã vượt quá số lần thất bại cho phép trong ngày (mặc định: 5 lần, có thể config) |
| `SERVER_KEY_NOT_CONFIGURED` | 500 | Private key chưa được cấu hình |
| `INVALID_PRIVATE_KEY` | 500 | Private key không hợp lệ |

---

### 3. Kiểm tra có thể submit QR code

**GET** `/api/qr/available`

Kiểm tra xem khách hàng có thể submit QR code hay không. API này kiểm tra các giới hạn rate limit và trạng thái cấm của tài khoản trước khi cho phép submit QR code.

#### Authentication
- **Required**: Bearer Token

#### Headers

```
Authorization: Bearer <token>
```

#### Request Body
Không có request body (GET request)

#### Response Example (Success - 200 OK)

Khi khách hàng có thể submit QR code:

```json
{
    "success": true,
    "message": "Success",
    "data": []
}
```

#### Response Example (Error - 401 Unauthorized)

```json
{
    "success": false,
    "message": "Unauthorized",
    "errors": {
        "code": "UNAUTHORIZED"
    }
}
```

#### Response Example (Error - 403 Permanent Banned)

```json
{
    "success": false,
    "message": "Account permanently banned due to too many failed attempts",
    "errors": {
        "code": "PERMANENT_BANNED"
    }
}
```

#### Response Example (Error - 429 Daily Limit Exceeded)

```json
{
    "success": false,
    "message": "Daily failed attempts limit reached",
    "errors": {
        "code": "DAILY_LIMIT_EXCEEDED"
    }
}
```

#### Notes
- API này nên được gọi trước khi submit QR code để kiểm tra trạng thái
- Rate limits được kiểm tra dựa trên lịch sử các lần submit QR code
- Tất cả các giới hạn có thể được cấu hình trong hệ thống
- API kiểm tra các giới hạn sau:
  - **Theo Customer ID:**
    - Tổng số lần nhập sai (lũy kế): Nếu vượt quá giới hạn (mặc định: 15 lần, config: `limit_total_failed`), trả về 403 `PERMANENT_BANNED`
    - Số lần nhập sai trong ngày: Nếu vượt quá giới hạn (mặc định: 5 lần, config: `limit_each_day_failed`), trả về 429 `DAILY_LIMIT_EXCEEDED`
  - **Theo IP Address:**
    - Tổng số lần nhập sai từ IP (lũy kế): Nếu vượt quá giới hạn (mặc định: 100 lần, config: `limit_by_ip_total_failed`), trả về 403 `PERMANENT_BANNED`

#### Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Chưa đăng nhập hoặc token không hợp lệ |
| `PERMANENT_BANNED` | 403 | Tổng số lần nhập sai vượt quá giới hạn (mặc định: 15 lần theo customer_id hoặc 100 lần theo IP, có thể config), cấm vĩnh viễn |
| `DAILY_LIMIT_EXCEEDED` | 429 | Vượt quá số lần nhập sai cho phép trong ngày (mặc định: 5 lần, có thể config) |

#### cURL Example

```bash
curl -X GET http://localhost:8000/api/qr/available \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

#### JavaScript Example

```javascript
const token = localStorage.getItem('customer_token');
const response = await fetch('/api/qr/available', {
    method: 'GET',
    headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
    }
});

const data = await response.json();
if (data.success) {
    // Customer có thể submit QR code
    console.log('Can submit QR code');
} else {
    // Xử lý lỗi
    if (data.errors?.code === 'PERMANENT_BANNED') {
        console.error('Account permanently banned');
    } else if (data.errors?.code === 'DAILY_LIMIT_EXCEEDED') {
        console.error('Daily limit exceeded');
    }
}
```

---

### 4. Lấy danh sách Phường/Xã

**GET** `/api/wards`

Lấy danh sách phường/xã với phân trang và bộ lọc.

#### Authentication
- **Required**: Bearer Token

#### Query Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `ward_page` | integer | No | 1 | Số trang hiện tại |
| `ward_per_page` | integer | No | 10 | Số lượng phường/xã trên mỗi trang |
| `ward_province_code` | string | No | - | Lọc theo mã tỉnh/thành phố (exact match) |
| `ward_keyword` | string | No | - | Tìm kiếm trong: code, name, name_en, full_name, full_name_en (LIKE) |
| `ward_code` | string | No | - | Tìm kiếm theo mã phường/xã (LIKE) |
| `ward_name` | string | No | - | Tìm kiếm theo tên phường/xã (LIKE) |

#### Response Example (Success - 200 OK)

```json
{
    "success": true,
    "message": "Lấy danh sách phường/xã thành công",
    "data": [
        {
            "code": "00101",
            "name": "Phường Phúc Xá",
            "name_en": "Phuc Xa Ward",
            "full_name": "Phường Phúc Xá, Quận Ba Đình, Thành phố Hà Nội",
            "full_name_en": "Phuc Xa Ward, Ba Dinh District, Ha Noi City",
            "code_name": "00101_Phường Phúc Xá",
            "province_code": "01"
        }
    ],
    "pagination": {
        "current_page": 1,
        "per_page": 10,
        "total": 10500,
        "last_page": 1050
    }
}
```

#### Response Example (Error - 500 Server Error)

```json
{
    "success": false,
    "message": "Có lỗi xảy ra khi lấy danh sách phường/xã: <error_message>",
    "errors": []
}
```

#### Notes
- Kết quả được sắp xếp theo tên (`name`) tăng dần
- Hỗ trợ filter theo mã tỉnh/thành phố để lấy danh sách phường/xã của một tỉnh cụ thể
- Filter `ward_keyword` tìm kiếm trong nhiều trường cùng lúc

---

### 5. Lấy danh sách Tỉnh/Thành phố

**GET** `/api/provinces`

Lấy danh sách tỉnh/thành phố với phân trang và bộ lọc.

#### Authentication
- **Required**: Bearer Token

#### Query Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `province_page` | integer | No | 1 | Số trang hiện tại |
| `province_per_page` | integer | No | 10 | Số lượng tỉnh/thành phố trên mỗi trang |
| `province_keyword` | string | No | - | Tìm kiếm trong: name, name_en, full_name, full_name_en (LIKE) |
| `province_code` | string | No | - | Tìm kiếm theo mã tỉnh/thành phố (LIKE) |
| `province_name` | string | No | - | Tìm kiếm theo tên tỉnh/thành phố (LIKE) |

#### Response Example (Success - 200 OK)

```json
{
    "success": true,
    "message": "Lấy danh sách tỉnh/thành phố thành công",
    "data": [
        {
            "code": "01",
            "name": "Thành phố Hà Nội",
            "name_en": "Ha Noi City",
            "full_name": "Thành phố Hà Nội",
            "full_name_en": "Ha Noi City",
            "code_name": "01_Thành phố Hà Nội"
        }
    ],
    "pagination": {
        "current_page": 1,
        "per_page": 10,
        "total": 63,
        "last_page": 7
    }
}
```

#### Response Example (Error - 500 Server Error)

```json
{
    "success": false,
    "message": "Có lỗi xảy ra khi lấy danh sách tỉnh/thành phố: <error_message>",
    "errors": []
}
```

#### Notes
- Kết quả được sắp xếp theo tên (`name`) tăng dần
- Filter `province_keyword` tìm kiếm trong nhiều trường cùng lúc

---

### 6. Tạo khách hàng mới

**POST** `/api/customers`

Tạo một khách hàng mới trong hệ thống.

#### Authentication
- **Required**: Bearer Token

#### Request Body

```json
{
    "name": "Nguyễn Văn A",
    "email": "nguyenvana@example.com",
    "phone": "0123456789",
    "address": "123 Đường ABC, Phường XYZ",
    "buy_address": "456 Đường DEF, Phường UVW",
    "identity_id": "123456789012",
    "channel": "zalo",
    "province_code": "01",
    "ward_code": "00101",
    "dob": "1990-01-15",
    "gender": "male"
}
```

#### Request Body Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Tên khách hàng (tối đa 255 ký tự) |
| `email` | string | No | Email khách hàng (phải là email hợp lệ, tối đa 255 ký tự) |
| `phone` | string | Yes | Số điện thoại (tối đa 20 ký tự) |
| `address` | string | No | Địa chỉ thường trú (tối đa 500 ký tự) |
| `buy_address` | string | No | Địa chỉ mua hàng (tối đa 500 ký tự) |
| `identity_id` | string | Yes | Zalo ID (tối đa 50 ký tự, phải unique) |
| `channel` | string | Yes | Kênh đăng ký (tối đa 100 ký tự) |
| `province_code` | string | No | Mã tỉnh/thành phố |
| `ward_code` | string | No | Mã phường/xã |
| `dob` | string | No | Ngày sinh (định dạng: YYYY-MM-DD) |
| `gender` | string | No | Giới tính |

#### Response Example (Success - 201 Created)

```json
{
    "success": true,
    "message": "Khách hàng đã được thêm thành công!",
    "data": {
        "id": 1,
        "name": "Nguyễn Văn A",
        "email": "nguyenvana@example.com",
        "phone": "0123456789",
        "address": "123 Đường ABC, Phường XYZ",
        "buy_address": "456 Đường DEF, Phường UVW",
        "identity_id": "123456789012",
        "channel": "zalo",
        "province_code": "01",
        "ward_code": "00101",
        "dob": "1990-01-15",
        "gender": "male",
        "created_at": "2024-01-01T00:00:00.000000Z",
        "updated_at": "2024-01-01T00:00:00.000000Z"
    }
}
```

#### Response Example (Error - 422 Validation Failed)

```json
{
    "success": false,
    "message": "Dữ liệu không hợp lệ",
    "errors": {
        "name": ["Tên khách hàng là bắt buộc."],
        "identity_id": ["Identity ID đã tồn tại trong hệ thống."]
    }
}
```

---

### 7. Lấy thông tin khách hàng theo Identity ID

**GET** `/api/customers/show?identity_id={identity_id}`

Lấy thông tin khách hàng dựa trên Identity ID (Zalo ID).

#### Authentication
- **Required**: Bearer Token

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `identity_id` | string | Yes | Zalo ID của khách hàng |

#### Response Example (Success - 200 OK)

```json
{
    "success": true,
    "message": "Lấy thông tin khách hàng thành công",
    "data": {
        "id": 1,
        "name": "Nguyễn Văn A",
        "email": "nguyenvana@example.com",
        "phone": "0123456789",
        "address": "123 Đường ABC, Phường XYZ",
        "buy_address": "456 Đường DEF, Phường UVW",
        "identity_id": "123456789012",
        "channel": "zalo",
        "province_code": "01",
        "ward_code": "00101",
        "created_at": "2024-01-01T00:00:00.000000Z",
        "updated_at": "2024-01-01T00:00:00.000000Z"
    }
}
```

#### Response Example (Error - 404 Not Found)

```json
{
    "success": false,
    "message": "Không tìm thấy khách hàng với Identity ID này."
}
```

#### Response Example (Error - 422 Validation Failed)

```json
{
    "success": false,
    "message": "Identity ID là bắt buộc",
    "errors": {
        "identity_id": ["The identity id field is required."]
    }
}
```

---

### 8. Cập nhật thông tin khách hàng

**PUT** `/api/customers/{id}`

Cập nhật thông tin của một khách hàng.

#### Authentication
- **Required**: Bearer Token

#### Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | integer | Yes | ID của khách hàng |

#### Request Body

```json
{
    "name": "Nguyễn Văn A Updated",
    "email": "newemail@example.com",
    "phone": "0987654321",
    "address": "789 Đường GHI, Phường JKL",
    "buy_address": "321 Đường MNO, Phường PQR",
    "identity_id": "123456789012",
    "channel": "zalo_updated",
    "dob": "1990-01-15",
    "gender": "male"
}
```

#### Request Body Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Tên khách hàng (tối đa 255 ký tự) |
| `email` | string | No | Email khách hàng (phải là email hợp lệ, tối đa 255 ký tự) |
| `phone` | string | Yes | Số điện thoại (tối đa 20 ký tự) |
| `address` | string | No | Địa chỉ thường trú (tối đa 500 ký tự) |
| `buy_address` | string | No | Địa chỉ mua hàng (tối đa 500 ký tự) |
| `identity_id` | string | Yes | Zalo ID (tối đa 50 ký tự, phải unique) |
| `channel` | string | Yes | Kênh đăng ký (tối đa 100 ký tự) |
| `dob` | string | No | Ngày sinh (định dạng: YYYY-MM-DD) |
| `gender` | string | No | Giới tính |

#### Response Example (Success - 200 OK)

```json
{
    "success": true,
    "message": "Cập nhật khách hàng thành công",
    "data": {
        "id": 1,
        "name": "Nguyễn Văn A Updated",
        "email": "newemail@example.com",
        "phone": "0987654321",
        "address": "789 Đường GHI, Phường JKL",
        "buy_address": "321 Đường MNO, Phường PQR",
        "identity_id": "123456789012",
        "channel": "zalo_updated",
        "province_code": "01",
        "ward_code": "00101",
        "dob": "1990-01-15",
        "gender": "male",
        "created_at": "2024-01-01T00:00:00.000000Z",
        "updated_at": "2024-01-15T10:30:00.000000Z"
    }
}
```

#### Response Example (Error - 404 Not Found)

```json
{
    "success": false,
    "message": "Khách hàng không tồn tại"
}
```

---

### 9. Lấy danh sách giải thưởng của chiến dịch

**GET** `/api/campaigns/{id}/prizes`

Lấy danh sách giải thưởng của một chiến dịch với phân trang.

#### Authentication
- **Required**: Bearer Token

#### Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | integer | Yes | ID của chiến dịch |

#### Query Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `prize_page` | integer | No | 1 | Số trang hiện tại |
| `prize_per_page` | integer | No | 10 | Số lượng giải thưởng trên mỗi trang |

#### Response Example (Success - 200 OK)

```json
{
    "success": true,
    "message": "Lấy danh sách giải thưởng thành công",
    "data": [
        {
            "id": 1,
            "campaign_id": 1,
            "name": "Giải nhất",
            "description": "iPhone 15 Pro",
            "reward_type": "product",
            "reward_value": "iPhone 15 Pro",
            "quantity": 1,
            "win_rate": 0.01,
            "image": "/storage/prizes/iphone15.jpg",
            "zns_template_id": "123456",
            "default_award_status": "pending",
            "sort_order": 1,
            "winners_count": 5,
            "created_at": "2024-01-01T00:00:00.000000Z",
            "updated_at": "2024-01-01T00:00:00.000000Z"
        }
    ],
    "pagination": {
        "current_page": 1,
        "per_page": 10,
        "total": 25,
        "last_page": 3
    }
}
```

#### Response Example (Error - 404 Not Found)

```json
{
    "success": false,
    "message": "Chiến dịch không tồn tại",
    "errors": []
}
```

#### Notes
- Kết quả được sắp xếp theo `id` tăng dần

---

### 10. Lấy chi tiết chiến dịch

**GET** `/api/campaigns/{id}`

Lấy thông tin chi tiết của một chiến dịch. Chỉ trả về chiến dịch đang hoạt động.

#### Authentication
- **Required**: Bearer Token
- Chỉ lấy được chiến dịch đang hoạt động

#### Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | integer | Yes | ID của chiến dịch |

#### Response Example (Success - 200 OK)

```json
{
    "success": true,
    "message": "Lấy chi tiết chiến dịch thành công",
    "data": {
        "id": 1,
        "name": "Chiến dịch mùa hè",
        "code": "SUMMER2024",
        "description": "Chiến dịch khuyến mãi mùa hè",
        "start_date": "2024-06-01T00:00:00.000000Z",
        "end_date": "2024-08-31T23:59:59.000000Z",
        "policy": "Điều khoản và điều kiện...",
        "salt_key": "campaign_salt_key_123",
        "is_generated_qr_code": true,
        "generated_qr_code_file_name": "SUMMER2024_QR_CODE_2024-07-15.zip",
        "config": {
            "background_image": "/storage/campaigns/background.jpg",
            "banner_image": "/storage/campaigns/banner.jpg",
            "zalo_app_url": "https://oa.zalo.me/123456"
        },
        "created_at": "2024-01-01T00:00:00.000000Z",
        "updated_at": "2024-01-15T10:30:00.000000Z"
    }
}
```

#### Response Example (Error - 404 Not Found)

```json
{
    "success": false,
    "message": "Chiến dịch chưa bắt đầu",
    "errors": {
        "code": "CAMPAIGN_NOT_START_YET"
    }
}
```

```json
{
    "success": false,
    "message": "Chiến dịch đã kết thúc",
    "errors": {
        "code": "CAMPAIGN_HAS_FINISHED"
    }
}
```

```json
{
    "success": false,
    "message": "Không tìm thấy chiến dịch hoặc chiến dịch không hoạt động"
}
```

#### Notes
- URL ảnh trong `config` đã được chuẩn hóa (http | /storage | public/...)

---

### 11. Lấy lịch sử trúng giải theo chiến dịch và khách hàng (phần thưởng lớn)

**GET** `/api/campaigns/{campaignId}/customer/{customerId}/winners`

Lấy lịch sử trúng giải (lọc theo phần thưởng lớn) của một khách hàng trong một chiến dịch cụ thể với phân trang.

#### Authentication
- **Required**: Bearer Token

#### Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `campaignId` | integer | Yes | ID của chiến dịch |
| `customerId` | integer | Yes | ID của khách hàng |

#### Query Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `page` | integer | No | 1 | Số trang hiện tại |
| `per_page` | integer | No | 10 | Số lượng bản ghi trên mỗi trang |

#### Response Example

```json
{
    "success": true,
    "message": "Lấy danh sách giải thưởng thành công",
    "data": [
        {
            "id": 1,
            "campaign_id": 1,
            "customer_id": 1,
            "prize_id": 1,
            "qr_code": "abc123def456",
            "award_status": "pending",
            "customer": {
                "id": 1,
                "name": "Nguyễn Văn A",
                "identity_id": "123456789012",
                "phone": "0123456789",
                "email": "nguyenvana@example.com"
            },
            "prize": {
                "id": 1,
                "name": "Giải nhất - iPhone 15 Pro"
            },
            "created_at": "2024-06-15T10:30:00.000000Z",
            "updated_at": "2024-06-15T10:30:00.000000Z"
        }
    ],
    "pagination": {
        "current_page": 1,
        "per_page": 10,
        "total": 5,
        "last_page": 1
    }
}
```

#### Notes
- Kết quả được sắp xếp theo `id` giảm dần (mới nhất trước)
- Mỗi bản ghi có kèm thông tin `customer` và `prize` (chỉ một số trường cơ bản)

---

### 12. Lấy lịch sử trúng giải theo chiến dịch và khách hàng (tất cả phần thưởng)

**GET** `/api/campaigns/{campaignId}/customer/{customerId}/winner-histories`

Lấy lịch sử trúng giải (không lọc theo loại phần thưởng) của một khách hàng trong một chiến dịch cụ thể với phân trang.

#### Authentication
- **Required**: Bearer Token

#### Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `campaignId` | integer | Yes | ID của chiến dịch |
| `customerId` | integer | Yes | ID của khách hàng |

#### Query Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `page` | integer | No | 1 | Số trang hiện tại |
| `per_page` | integer | No | 10 | Số lượng bản ghi trên mỗi trang |

#### Response Example

```json
{
    "success": true,
    "message": "Lấy danh sách giải thưởng thành công",
    "data": [
        {
            "id": 1,
            "campaign_id": 1,
            "customer_id": 1,
            "prize_id": 2,
            "qr_code": "abc123def456",
            "award_status": "pending",
            "customer": {
                "id": 1,
                "name": "Nguyễn Văn A",
                "identity_id": "123456789012",
                "phone": "0123456789",
                "email": "nguyenvana@example.com"
            },
            "prize": {
                "id": 2,
                "name": "Voucher 50k"
            },
            "created_at": "2024-06-15T10:30:00.000000Z",
            "updated_at": "2024-06-15T10:30:00.000000Z"
        }
    ],
    "pagination": {
        "current_page": 1,
        "per_page": 10,
        "total": 5,
        "last_page": 1
    }
}
```

#### Notes
- Kết quả được sắp xếp theo `id` giảm dần (mới nhất trước)

## Error Responses

### 400 Bad Request

```json
{
    "success": false,
    "message": "Dữ liệu không hợp lệ",
    "errors": {
        "field_name": ["Error message"]
    }
}
```

### 404 Not Found

```json
{
    "success": false,
    "message": "Không tìm thấy tài nguyên",
    "errors": []
}
```

### 409 Conflict

```json
{
    "success": false,
    "message": "QR already used",
    "errors": {
        "code": "QR_ALREADY_USED"
    }
}
```

### 422 Validation Failed

```json
{
    "success": false,
    "message": "Dữ liệu không hợp lệ",
    "errors": {
        "field_name": ["Validation error message"]
    }
}
```

### 429 Too Many Requests

```json
{
    "success": false,
    "message": "Daily failed attempts limit reached",
    "errors": {
        "code": "DAILY_LIMIT_EXCEEDED"
    }
}
```

### 500 Internal Server Error

```json
{
    "success": false,
    "message": "Có lỗi xảy ra khi xử lý yêu cầu",
    "errors": []
}
```

## Rate Limiting

Một số endpoints có rate limiting:

- **POST `/api/qr/submit`**: 
  - Tổng số lần thất bại: mặc định 15 lần (config: `limit_total_failed`)
  - Tổng số lần thất bại theo IP: mặc định 100 lần (config: `limit_by_ip_total_failed`)
  - Số lần thất bại mỗi ngày: mặc định 5 lần (config: `limit_each_day_failed`)

## Security Notes

### QR Code Submission
- QR code phải được mã hóa bằng RSA public key trước khi gửi
- Payload được decrypt bằng RSA private key với PKCS1 OAEP padding
