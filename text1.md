# Phan 1 - Phan tich logic quan ly Refresh Token

## 1. Boi canh hien tai

Trong co che dang nhap bang JWT, Access Token thuong co thoi gian song ngan va duoc dung de goi cac API duoc bao ve. Refresh Token co thoi gian song dai hon, duoc dung de xin cap Access Token moi khi Access Token het han.

Theo mo ta bai toan, he thong hien tai moi tap trung vao viec vo hieu hoa mot Refresh Token cu the khi nguoi dung dang xuat. Cach lam nay phu hop voi truong hop nguoi dung chi co mot phien dang nhap, nhung khong du manh khi nguoi dung dang nhap tren nhieu thiet bi cung luc.

Vi du:

- Nguoi dung dang nhap tren dien thoai.
- Nguoi dung tiep tuc dang nhap tren may tinh bang.
- Khi dang xuat khoi dien thoai, he thong chi xoa hoac vo hieu hoa Refresh Token cua dien thoai.
- Refresh Token tren may tinh bang van con hieu luc.

Dieu nay co nghia la viec logout hien tai chi tac dong den mot token rieng le, khong tac dong den toan bo cac phien dang nhap cua tai khoan.

## 2. Phan tich RefreshTokenService hien co

Trong mot du an Spring Boot thong dung, `RefreshTokenService` thuong co cac phuong thuc sau:

### `createRefreshToken(userId)`

Phuong thuc nay tao Refresh Token moi cho nguoi dung.

Logic thuong gap:

```java
RefreshToken refreshToken = new RefreshToken();
refreshToken.setUser(user);
refreshToken.setToken(UUID.randomUUID().toString());
refreshToken.setExpiryDate(Instant.now().plusMillis(refreshTokenDurationMs));
return refreshTokenRepository.save(refreshToken);
```

Danh gia:

- Tao duoc token moi va gan voi nguoi dung.
- Co thoi gian het han ro rang.
- Chua phan biet token duoc tao tu thiet bi nao.
- Neu cung mot user dang nhap tren nhieu thiet bi, server khong co du thong tin de quan ly tung phien mot cach chinh xac.

### `findByToken(token)`

Phuong thuc nay tim Refresh Token trong database dua tren chuoi token.

Logic thuong gap:

```java
return refreshTokenRepository.findByToken(token);
```

Danh gia:

- Huu ich khi client gui Refresh Token len de xin Access Token moi.
- Chi xac dinh duoc token rieng le.
- Khong tra loi duoc cau hoi token nay thuoc phien/thiet bi nao neu entity chua co `deviceId`.

### `verifyExpiration(refreshToken)`

Phuong thuc nay kiem tra Refresh Token da het han hay chua.

Logic thuong gap:

```java
if (refreshToken.getExpiryDate().isBefore(Instant.now())) {
    refreshTokenRepository.delete(refreshToken);
    throw new TokenRefreshException("Refresh token was expired");
}
return refreshToken;
```

Danh gia:

- Dam bao token het han khong duoc dung de cap Access Token moi.
- Co the xoa token het han tai thoi diem token duoc su dung.
- Chi don dep bi dong: token het han nhung khong bao gio duoc client gui lai thi van nam trong database.

### `deleteByToken(token)` hoac `logout(refreshToken)`

Phuong thuc nay xoa hoac vo hieu hoa mot Refresh Token cu the khi logout.

Logic thuong gap:

```java
refreshTokenRepository.deleteByToken(token);
```

Danh gia:

- Dang xuat duoc phien hien tai neu client gui dung Refresh Token.
- Khong thu hoi cac token khac cua cung user.
- Neu nguoi dung muon dang xuat tat ca thiet bi, he thong hien tai chua dap ung duoc.

## 3. Phan tich RefreshTokenRepository hien co

Repository thuong co cac phuong thuc sau:

```java
Optional<RefreshToken> findByToken(String token);

void deleteByToken(String token);

void deleteByUserId(Long userId);
```

Neu chi co `findByToken` va `deleteByToken`, he thong moi quan ly duoc tung Refresh Token don le.

Han che:

- Khong co cach xoa tat ca token cua mot user neu thieu `deleteByUserId`.
- Khong co cach xoa token theo tung thiet bi neu thieu `deleteByUserIdAndDeviceId`.
- Khong co cach don dep token het han neu thieu `deleteByExpiryDateBefore`.

Repository can duoc bo sung cac phuong thuc sau:

```java
Optional<RefreshToken> findByToken(String token);

void deleteByToken(String token);

void deleteByUserId(Long userId);

void deleteByUserIdAndDeviceId(Long userId, String deviceId);

void deleteByExpiryDateBefore(Instant now);
```

## 4. Danh gia co che thu hoi token hien tai khi logout nhieu thiet bi

Co che hien tai chi vo hieu hoa mot Refresh Token cu the. Neu nguoi dung co nhieu phien dang nhap, moi phien se co mot Refresh Token rieng. Viec xoa token cua phien A khong anh huong den token cua phien B.

Ve trai nghiem nguoi dung:

- Logout tren dien thoai nhung may tinh bang van dang nhap.
- Neu nguoi dung ky vong "dang xuat khoi tat ca noi", he thong khong dap ung.
- Khong co kha nang hien thi hoac quan ly danh sach thiet bi/phien dang nhap.

Ve bao mat:

- Neu mot thiet bi bi mat, token tren thiet bi do co the van song den khi het han neu khong duoc thu hoi dung cach.
- Neu token bi ro ri, viec chi xoa token hien tai khong dam bao tai khoan duoc bao ve tren toan bo thiet bi.
- Token het han con ton tai trong database lam tang rui ro neu database bi khai thac qua lo hong khac.

## 5. Cac han che hien co

### Khong co dinh danh thiet bi

Refresh Token chua co truong `deviceId`, nen server khong biet token nao thuoc thiet bi nao. Dieu nay lam cho viec logout theo tung thiet bi tro nen kho khan.

### Logout chi xu ly mot token

Endpoint logout hien tai chi xoa token client gui len. Neu user muon dang xuat tat ca thiet bi, can endpoint rieng de xoa toan bo token theo `userId`.

### Khong co co che don dep token het han

Token het han co the tich luy lau dai trong database. Ve lau dai, bang refresh token se phinh to, lam lang phi tai nguyen va anh huong den hieu nang truy van.

### Chua ho tro quan ly phien lam viec

Vi thieu `deviceId`, he thong khong the:

- Biet user dang dang nhap tren bao nhieu thiet bi.
- Xoa chinh xac phien cua mot thiet bi.
- Trien khai man hinh "quan ly thiet bi dang nhap".
- Phat hien bat thuong theo thiet bi.

## 6. Phuong an cai tien de xuat

### Bo sung `deviceId` vao RefreshToken

Moi Refresh Token can duoc gan voi mot `deviceId`. Gia tri nay co the:

- Do client gui len trong header, vi du `X-Device-Id`.
- Do server sinh bang `UUID.randomUUID().toString()` neu client chua co.

Khi dang nhap lan dau tren mot thiet bi, client co the nhan `deviceId` tu response va luu lai o local storage, secure storage hoac bo nho ung dung tuy nen tang.

### Cap nhat tao Refresh Token

Phuong thuc `createRefreshToken` can nhan them `deviceId`.

Neu client co gui `deviceId`, server dung gia tri do. Neu khong, server tao moi UUID.

### Logout phien hien tai

Endpoint `/api/auth/logout` can nhan `deviceId` hoac nhan ca `refreshToken` va `deviceId`.

Logic de xuat:

- Xac dinh user hien tai tu Access Token.
- Lay `deviceId` tu request body hoac header.
- Xoa Refresh Token theo `userId` va `deviceId`.
- Chi phien hien tai bi dang xuat.

### Logout tat ca thiet bi

Them endpoint `/api/auth/logoutAllDevices`.

Logic de xuat:

- Xac dinh user hien tai tu Access Token.
- Goi service xoa toan bo Refresh Token cua user do.
- Tat ca phien dang nhap cua user se mat kha nang refresh Access Token.

### Don dep token het han

Bo sung phuong thuc service:

```java
@Transactional
public void cleanupExpiredRefreshTokens() {
    refreshTokenRepository.deleteByExpiryDateBefore(Instant.now());
}
```

Phuong thuc nay co the duoc goi boi mot `@Scheduled` task trong tuong lai, vi du chay moi ngay luc 0 gio.

## 7. Tam quan trong cua `deviceId`

`deviceId` giup bien Refresh Token tu mot chuoi token don le thanh mot phien lam viec co danh tinh ro rang.

Loi ich:

- Dang xuat dung thiet bi hien tai.
- Dang xuat tat ca thiet bi khi can bao mat tai khoan.
- Ho tro quan ly danh sach thiet bi dang nhap.
- Giam rui ro khi token cua mot thiet bi bi lo.
- Tao nen trai nghiem nhat quan hon cho nguoi dung.

Ket luan: he thong nen quan ly Refresh Token theo cap `userId + deviceId`. Cach nay giup phan biet tung phien dang nhap, ho tro ca logout hien tai va logout tat ca thiet bi, dong thoi tao nen nen tang tot hon cho bao mat va quan ly phien.
