# Phan 2 - Thuc thi cai tien quan ly Refresh Token

## 1. Mo rong RefreshToken Entity

Bo sung truong `deviceId` kieu `String` vao entity `RefreshToken`.

Vi du:

```java
@Entity
@Table(name = "refresh_tokens")
public class RefreshToken {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String token;

    @Column(nullable = false)
    private Instant expiryDate;

    @Column(nullable = false)
    private String deviceId;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    // getters and setters
}
```

Y nghia:

- `token`: chuoi Refresh Token duy nhat.
- `expiryDate`: thoi diem token het han.
- `deviceId`: dinh danh thiet bi hoac phien tao ra token.
- `user`: tai khoan so huu token.

Neu dung migration database, can bo sung cot:

```sql
ALTER TABLE refresh_tokens ADD COLUMN device_id VARCHAR(255) NOT NULL;
```

Trong truong hop bang da co du lieu cu, co the can cho phep null tam thoi, cap nhat du lieu, sau do moi chuyen sang `NOT NULL`.

## 2. Cap nhat RefreshTokenRepository

Repository can co cac phuong thuc ho tro tim, xoa theo token, xoa theo user, xoa theo user va device, dong thoi xoa token het han.

```java
public interface RefreshTokenRepository extends JpaRepository<RefreshToken, Long> {

    Optional<RefreshToken> findByToken(String token);

    @Transactional
    void deleteByToken(String token);

    @Transactional
    void deleteByUserId(Long userId);

    @Transactional
    void deleteByUserIdAndDeviceId(Long userId, String deviceId);

    @Transactional
    void deleteByExpiryDateBefore(Instant now);
}
```

Neu entity `RefreshToken` lien ket voi `User` bang field ten la `user`, Spring Data JPA co the can dat ten method theo duong dan thuoc tinh:

```java
void deleteByUser_Id(Long userId);

void deleteByUser_IdAndDeviceId(Long userId, String deviceId);
```

Chon ten method theo dung cau truc entity trong du an goc.

## 3. Cap nhat createRefreshToken

`createRefreshToken` can nhan them `deviceId`. Neu client khong gui len, server se sinh UUID moi.

```java
@Service
public class RefreshTokenService {

    private final RefreshTokenRepository refreshTokenRepository;
    private final UserRepository userRepository;
    private final long refreshTokenDurationMs;

    public RefreshTokenService(
            RefreshTokenRepository refreshTokenRepository,
            UserRepository userRepository,
            @Value("${app.jwtRefreshExpirationMs}") long refreshTokenDurationMs
    ) {
        this.refreshTokenRepository = refreshTokenRepository;
        this.userRepository = userRepository;
        this.refreshTokenDurationMs = refreshTokenDurationMs;
    }

    public RefreshToken createRefreshToken(Long userId, String deviceId) {
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new RuntimeException("User not found"));

        String resolvedDeviceId = resolveDeviceId(deviceId);

        RefreshToken refreshToken = new RefreshToken();
        refreshToken.setUser(user);
        refreshToken.setToken(UUID.randomUUID().toString());
        refreshToken.setExpiryDate(Instant.now().plusMillis(refreshTokenDurationMs));
        refreshToken.setDeviceId(resolvedDeviceId);

        return refreshTokenRepository.save(refreshToken);
    }

    private String resolveDeviceId(String deviceId) {
        if (deviceId == null || deviceId.isBlank()) {
            return UUID.randomUUID().toString();
        }
        return deviceId;
    }
}
```

Neu muon moi thiet bi chi co mot Refresh Token dang hoat dong, co the xoa token cu cua cung `userId + deviceId` truoc khi tao token moi:

```java
@Transactional
public RefreshToken createRefreshToken(Long userId, String deviceId) {
    User user = userRepository.findById(userId)
            .orElseThrow(() -> new RuntimeException("User not found"));

    String resolvedDeviceId = resolveDeviceId(deviceId);
    refreshTokenRepository.deleteByUser_IdAndDeviceId(userId, resolvedDeviceId);

    RefreshToken refreshToken = new RefreshToken();
    refreshToken.setUser(user);
    refreshToken.setToken(UUID.randomUUID().toString());
    refreshToken.setExpiryDate(Instant.now().plusMillis(refreshTokenDurationMs));
    refreshToken.setDeviceId(resolvedDeviceId);

    return refreshTokenRepository.save(refreshToken);
}
```

## 4. Request/Response DTO de truyen deviceId

Client co the gui `deviceId` trong body hoac header. Cach de doc ro trong bai nop la dung request body.

### LoginRequest

```java
public class LoginRequest {
    private String username;
    private String password;
    private String deviceId;

    // getters and setters
}
```

### LogoutRequest

```java
public class LogoutRequest {
    private String deviceId;

    // getters and setters
}
```

Neu muon logout dua truc tiep tren Refresh Token, co the bo sung:

```java
public class LogoutRequest {
    private String refreshToken;
    private String deviceId;

    // getters and setters
}
```

## 5. Cap nhat AuthController

### Endpoint dang nhap

Khi dang nhap thanh cong, server truyen `deviceId` vao `createRefreshToken`.

```java
@PostMapping("/login")
public ResponseEntity<?> login(@RequestBody LoginRequest request) {
    // 1. Xac thuc username/password
    // 2. Tao Access Token
    // 3. Tao Refresh Token gan voi deviceId

    RefreshToken refreshToken = refreshTokenService.createRefreshToken(
            user.getId(),
            request.getDeviceId()
    );

    return ResponseEntity.ok(new LoginResponse(
            accessToken,
            refreshToken.getToken(),
            refreshToken.getDeviceId()
    ));
}
```

Client nen luu lai `deviceId` tra ve de gui len trong cac lan logout hoac refresh tiep theo.

### Endpoint logout phien hien tai

Endpoint hien co `/api/auth/logout` duoc cap nhat de chi logout theo `userId + deviceId`.

```java
@PostMapping("/logout")
public ResponseEntity<?> logout(
        @AuthenticationPrincipal UserDetails userDetails,
        @RequestBody LogoutRequest request
) {
    Long userId = getUserIdFromUserDetails(userDetails);
    refreshTokenService.logoutCurrentDevice(userId, request.getDeviceId());

    return ResponseEntity.ok("Logged out current device successfully");
}
```

Service tuong ung:

```java
@Transactional
public void logoutCurrentDevice(Long userId, String deviceId) {
    if (deviceId == null || deviceId.isBlank()) {
        throw new IllegalArgumentException("deviceId is required");
    }

    refreshTokenRepository.deleteByUser_IdAndDeviceId(userId, deviceId);
}
```

Ket qua:

- Chi Refresh Token cua phien hien tai bi xoa.
- Cac thiet bi khac cua cung user van tiep tuc hoat dong.

### Endpoint logout tat ca thiet bi

Them endpoint moi `/api/auth/logoutAllDevices`.

```java
@PostMapping("/logoutAllDevices")
public ResponseEntity<?> logoutAllDevices(
        @AuthenticationPrincipal UserDetails userDetails
) {
    Long userId = getUserIdFromUserDetails(userDetails);
    refreshTokenService.logoutAllDevices(userId);

    return ResponseEntity.ok("Logged out all devices successfully");
}
```

Service tuong ung:

```java
@Transactional
public void logoutAllDevices(Long userId) {
    refreshTokenRepository.deleteByUser_Id(userId);
}
```

Ket qua:

- Tat ca Refresh Token cua user bi xoa.
- Tat ca thiet bi khong the dung Refresh Token cu de xin Access Token moi.
- Neu Access Token hien tai van con hieu luc, no chi song den khi het han ngan han.

## 6. Phac thao co che tu dong don dep token het han

Khong can trien khai chi tiet `@Scheduled`, nhung nen co san ham service de goi dinh ky.

```java
@Transactional
public void cleanupExpiredRefreshTokens() {
    Instant now = Instant.now();
    refreshTokenRepository.deleteByExpiryDateBefore(now);
}
```

Neu tach thanh service rieng:

```java
@Service
public class RefreshTokenCleanupService {

    private final RefreshTokenRepository refreshTokenRepository;

    public RefreshTokenCleanupService(RefreshTokenRepository refreshTokenRepository) {
        this.refreshTokenRepository = refreshTokenRepository;
    }

    @Transactional
    public void cleanupExpiredRefreshTokens() {
        refreshTokenRepository.deleteByExpiryDateBefore(Instant.now());
    }
}
```

Phac thao cach goi tu task dinh ky:

```java
// Chua can trien khai chi tiet trong bai nay
// @Scheduled(cron = "0 0 0 * * *")
public void runDailyRefreshTokenCleanup() {
    refreshTokenCleanupService.cleanupExpiredRefreshTokens();
}
```

Cac buoc xu ly:

1. Lay thoi diem hien tai bang `Instant.now()`.
2. Goi repository xoa cac token co `expiryDate` nho hon thoi diem hien tai.
3. Dat `@Transactional` de thao tac xoa nhieu ban ghi duoc thuc hien an toan.
4. Ghi log so luong token da xoa neu repository tra ve count.

Neu muon biet so luong token da xoa, repository co the khai bao:

```java
long deleteByExpiryDateBefore(Instant now);
```

## 7. Goi y cach client truyen deviceId

Client co hai cach pho bien:

### Gui trong request body

```json
{
  "username": "user01",
  "password": "123456",
  "deviceId": "2c8c7f56-9c3a-4c3a-b06f-cc6f0b47d123"
}
```

### Gui trong header

```http
X-Device-Id: 2c8c7f56-9c3a-4c3a-b06f-cc6f0b47d123
```

Neu client chua co `deviceId`, server sinh moi UUID va tra ve trong response:

```json
{
  "accessToken": "jwt-access-token",
  "refreshToken": "refresh-token",
  "deviceId": "2c8c7f56-9c3a-4c3a-b06f-cc6f0b47d123"
}
```

## 8. Tong ket logic nghiep vu moi

Sau khi cai tien, Refresh Token duoc quan ly theo tung thiet bi/phien:

- Dang nhap tao Refresh Token moi gan voi `userId` va `deviceId`.
- Logout thuong xoa token theo `userId + deviceId`.
- Logout tat ca thiet bi xoa moi token theo `userId`.
- Token het han duoc don dep bang `cleanupExpiredRefreshTokens`.

Cach lam nay giup he thong bao mat hon, quan ly phien ro rang hon va tranh tich luy du lieu token khong con gia tri trong database.
